# Security

## Committed a secret or sensitive data?

If a password, key, token, connection string with credentials, or any customer or
personal data has been committed — even to a branch, even if you deleted it in a
later commit:

1. **Stop.** Don't push more commits trying to hide it. Git history keeps it.
2. **Don't open an issue or PR about it.** That spreads the details further.
3. **Tell the IT Manager directly, right away** — in person, by phone, or by direct
   message.
4. **Rotate the credential.** Treat it as compromised, even in an internal
   repository. Rotation comes before cleanup.
5. The IT Manager coordinates removing it from history (`git filter-repo`), because
   that rewrites history for everyone with a clone.

Nobody gets in trouble for reporting quickly. Delay makes it worse.

## Found a vulnerability?

Report it to the IT Manager directly, not in a public issue. Include the repository,
what you found, and how to reproduce it.
