# Zimbra to Carbonio Password Migration

A Bash-based utility to migrate existing user password hashes from **Zimbra 8.8.12 GA** to **Carbonio CE 26.9.2** while preserving users' existing passwords.

The primary objective is to avoid forced password resets during email server migration.

## Overview

During a Zimbra-to-Carbonio migration, one important challenge is preserving existing user credentials.

Zimbra 8.8.12 and Carbonio CE 26.9.2 can use different password-hashing schemes.

In our tested environment:

- Source: Zimbra 8.8.12 GA
- Destination: Carbonio CE 26.9.2
- Source password scheme: `{SSHA512}`
- Destination default password scheme for new passwords: `{ARGON2}`
- Migration method: Direct LDAP attribute modification using Carbonio provisioning.

The migration preserves the original encoded password hash rather than attempting to recover or convert the plaintext password.

## Key Finding

The important distinction is between setting a new password and importing an existing password hash.

**Incorrect approach for importing an existing hash:**

```bash
carbonio prov sp user@domain.com "{SSHA512}..."
```

The `sp` command treats its argument as a plaintext password and applies the configured hashing mechanism.

**Working approach in the tested environment:**

```bash
carbonio prov -l ma user@domain.com userPassword "{SSHA512}..."
```

The `-l` option uses the LDAP provisioning path and allows the existing encoded hash to be written directly to the LDAP attribute.

The imported hash must be verified and actual user authentication must be tested.

## Architecture

```text
             ZIMBRA 8.8.12
                    |
                    |
          test_accounts.txt
                    |
                    v
          export_test_passwords.sh
                    |
                    v
        zimbra_password_export.csv
                    |
                    |
             Secure SCP Transfer
                    |
                    v
            CARBONIO CE 26.9.2
                    |
                    v
          import_test_passwords.sh
                    |
                    v
            LDAP userPassword
                    |
                    v
             Hash Verification
                    |
                    v
           Original Password Login
```

## Prerequisites

### Source server: Zimbra

- Zimbra 8.8.12 GA
- Access to `zmprov`
- Zimbra administrative privileges
- Bash
- SSH/SCP access to Carbonio

### Destination server: Carbonio

- Carbonio CE 26.9.2
- Access to `carbonio prov`
- Appropriate LDAP provisioning privileges
- Bash
- Existing destination user accounts

### Important

This project was tested with `{SSHA512}` password hashes. Other password schemes require separate compatibility testing.

## Step 1: Select test accounts

On the Zimbra server, switch to the Zimbra service account.

Create a list of selected accounts:

```bash
nano test_accounts.txt
```

Example:

```text
test1@websurfer.com.np
test2@websurfer.com.np
test3@websurfer.com.np
test4@websurfer.com.np
test5@websurfer.com.np
```

Add only the accounts you intend to test.

The initial pilot should contain 10–15 non-critical accounts.

## Step 2: Export password hashes from Zimbra

Create:

`scripts/export_test_passwords.sh`

```bash
#!/bin/bash

INPUT="test_accounts.txt"
OUTPUT="zimbra_password_export.csv"

if [[ ! -f "$INPUT" ]]; then
    echo "ERROR: Account list not found."
    exit 1
fi

umask 077

echo "email,password_hash" > "$OUTPUT"

SUCCESS=0
FAILED=0

while IFS= read -r EMAIL || [[ -n "$EMAIL" ]]; do

    EMAIL=$(echo "$EMAIL" | xargs)

    [[ -z "$EMAIL" || "$EMAIL" == \#* ]] && continue

    HASH=$(zmprov -l ga "$EMAIL" userPassword 2>/dev/null |
        awk -F': ' '$1 == "userPassword" {print $2}')

    if [[ "$HASH" == \{SSHA512\}* ]]; then

        printf '%s,%s\n' "$EMAIL" "$HASH" >> "$OUTPUT"

        echo "[SUCCESS] $EMAIL"
        ((SUCCESS++))

    else

        echo "[FAILED] $EMAIL - Missing or unsupported hash"
        ((FAILED++))

    fi

done < "$INPUT"

echo
echo "========== EXPORT SUMMARY =========="
echo "Successful: $SUCCESS"
echo "Failed: $FAILED"
echo "Output: $OUTPUT"
echo "===================================="
```

Make the script executable:

```bash
chmod +x scripts/export_test_passwords.sh
```

Run:

```bash
./scripts/export_test_passwords.sh
```

The output file contains:

```csv
email,password_hash
test1@websurfer.com.np,{SSHA512}...
test2@websurfer.com.np,{SSHA512}...
```

Only supported SSHA512 hashes are exported.

## Step 3: Transfer the export file

Transfer the generated CSV to the Carbonio server using SCP:

```bash
scp zimbra_password_export.csv \
root@CARBONIO_SERVER_IP:/root/
```

On Carbonio, move it to a location accessible to the `zextras` service account:

```bash
install -o zextras -g zextras -m 600 \
/root/zimbra_password_export.csv \
/tmp/zimbra_password_export.csv
```

Switch to the Carbonio service account:

```bash
su - zextras
```

## Step 4: Import password hashes into Carbonio

Create:

`scripts/import_test_passwords.sh`

```bash
#!/bin/bash

INPUT="/tmp/zimbra_password_export.csv"

SUCCESS=0
FAILED=0
SKIPPED=0

if [[ ! -r "$INPUT" ]]; then
    echo "ERROR: Cannot read $INPUT"
    exit 1
fi

echo "Starting Carbonio password migration..."
echo

while IFS=',' read -r EMAIL HASH; do

    [[ "$EMAIL" == "email" ]] && continue

    if [[ -z "$EMAIL" || "$HASH" != \{SSHA512\}* ]]; then
        echo "[SKIPPED] Invalid entry: $EMAIL"
        ((SKIPPED++))
        continue
    fi

    if ! carbonio prov -l ga "$EMAIL" >/dev/null 2>&1; then
        echo "[SKIPPED] Account does not exist: $EMAIL"
        ((SKIPPED++))
        continue
    fi

    if ! carbonio prov -l ma "$EMAIL" userPassword "$HASH" \
        >/dev/null 2>&1; then

        echo "[FAILED] Import: $EMAIL"
        ((FAILED++))
        continue
    fi

    ACTUAL=$(carbonio prov -l ga "$EMAIL" userPassword |
        awk -F': ' '$1 == "userPassword" {print $2}')

    if [[ "$ACTUAL" == "$HASH" ]]; then

        echo "[SUCCESS] $EMAIL"
        ((SUCCESS++))

    else

        echo "[FAILED] Hash verification: $EMAIL"
        ((FAILED++))

    fi

done < "$INPUT"

echo
echo "========== MIGRATION SUMMARY =========="
echo "Successful: $SUCCESS"
echo "Failed: $FAILED"
echo "Skipped: $SKIPPED"
echo "======================================="
```

Make executable:

```bash
chmod +x scripts/import_test_passwords.sh
```

Run:

```bash
./scripts/import_test_passwords.sh
```

The script:

1. Reads the exported CSV.
2. Validates the SSHA512 prefix.
3. Checks whether the destination account exists.
4. Imports the original hash using LDAP provisioning.
5. Reads back the destination hash.
6. Compares the imported hash with the original.
7. Reports successful, failed and skipped accounts.

## Step 5: Authentication testing

Hash preservation alone does not guarantee successful authentication.

Test each pilot account using:

| Test | Expected result |
|---|---|
| LDAP hash verification | Original SSHA512 hash preserved |
| Carbonio Webmail | Successful login |
| IMAP over TLS (993) | Successful authentication |
| SMTP submission (587) | Successful authentication |
| Password change | Successful |
| New password storage | Carbonio configured scheme |

Use the original plaintext password when testing login.

Never use the encoded password hash as the login password.

## Step 6: Security considerations

Password hashes are sensitive authentication material.

Follow these precautions:

- Restrict access to exported password files.
- Use SCP or another encrypted transfer mechanism.
- Never commit password hashes to GitHub.
- Never print actual password hashes in migration logs.
- Keep backups protected.
- Remove temporary export files according to your organization's retention policy.
- Test with non-critical accounts before production migration.

Add the following to `.gitignore`:

```gitignore
# Sensitive migration data
*.csv
*.hash
*.password
test_accounts.txt

# Temporary files
*.tmp
```

## Troubleshooting

### Problem 1: Password violation: invalid password: too long

Example:

```text
Password violation: invalid password: too long.
Password is set successfully.
```

Likely cause: An encoded password hash was passed to the normal password-setting command.

Incorrect:

```bash
carbonio prov sp user@domain.com "{SSHA512}..."
```

Correct for the tested environment:

```bash
carbonio prov -l ma user@domain.com userPassword "{SSHA512}..."
```

### Problem 2: Permission denied

Ensure the hash export is accessible to the Carbonio service account.

Example:

```bash
install -o zextras -g zextras -m 600 \
/root/zimbra_password_export.csv \
/tmp/zimbra_password_export.csv
```

### Problem 3: Authentication fails after importing the hash

Check:

- Whether the original hash was preserved.
- Whether the account identity matches.
- Whether the authentication service accepts the imported scheme.
- Relevant Carbonio mailbox and LDAP logs.

Do not assume that a successful LDAP modification means authentication will work.

## Current Testing Status

| Stage | Status |
|---|---|
| Zimbra hash extraction | Tested |
| SSHA512 identification | Tested |
| Carbonio LDAP import | Tested |
| Original hash preservation | Tested |
| Original password Webmail login | Tested successfully |
| Automated 10–15 account batch | Pending |
| IMAP and SMTP testing | Pending |
| Production-scale migration | Pending |

## Future Improvements

- Support additional password-hashing schemes after compatibility testing.
- Add dry-run mode.
- Add detailed migration reports.
- Add account mapping validation.
- Add secure backup and rollback procedures.
- Support larger batch migrations.
- Integrate password migration into a complete Zimbra-to-Carbonio migration workflow.

## Disclaimer

This project is intended for authorized system administrators performing legitimate email server migrations.

Tested with Zimbra 8.8.12 GA and Carbonio CE 26.9.2 in the documented environment. Behaviour may differ between releases and installations.

Always test in a non-production environment before applying changes to production accounts.

## Author

Rupesh Kumar Sharma

Linux System Administration | Infrastructure Engineering | Email Systems | Linux Security

GitHub: [Your GitHub Profile]

LinkedIn: [Your LinkedIn Profile]
