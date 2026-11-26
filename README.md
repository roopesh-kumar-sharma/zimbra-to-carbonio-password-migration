# Zimbra to Carbonio Password Migration

A tested procedure for migrating an existing `{SSHA512}` password hash from Zimbra to Carbonio CE without forcing the user to reset their password.

## Tested Environment

- Source: Zimbra
- Destination: Carbonio CE 26.9.2
- Source password hash: `{SSHA512}`
- Carbonio default password hash: `{ARGON2}`
- Tested scope: **1 account**

## Migration Procedure

### 1. Retrieve the password hash from Zimbra

Run on the source server as the Zimbra user:

```bash
zmprov -l ga user@example.com userPassword
```

Copy the existing `{SSHA512}` hash from the output.

### 2. Import the hash into Carbonio

Run on the Carbonio server as the `zextras` user:

```bash
carbonio prov -l ma user@example.com userPassword "{SSHA512}..."
```

Replace the example address and hash with the actual account and retrieved hash.

**Important:** Do not use `carbonio prov sp` for this operation. It treats the supplied value as a plaintext password rather than importing the existing hash.

### 3. Verify the imported hash

Retrieve the password attribute:

```bash
carbonio prov -l ga user@example.com userPassword
```

Compare the returned hash with the original Zimbra hash. They should match exactly.

### 4. Test authentication

Log in to Carbonio Webmail using the account's original password.

## Test Results

| Test | Result |
|---|---|
| Retrieve source password hash | Successful |
| Import `{SSHA512}` into Carbonio | Successful |
| Verify hash preservation | Successful |
| Login using original password | Successful |
| Multi-account migration | Not tested |

## Important Notes

- This procedure has been verified with one account only.
- Multi-account migration has not yet been tested.
- Never publish or commit real email addresses, password hashes, or plaintext passwords.
- Test in a non-production environment before applying this procedure to production accounts.
