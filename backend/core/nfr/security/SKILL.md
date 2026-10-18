# Security

Security is non-negotiable and belongs at every layer, not bolted on at the edge. Assume every input is hostile and every dependency can be compromised.

## Rules

- **Never trust input.** Validate type, length, and shape at the API boundary. Reject on failure; don't try to "sanitize" into validity.
- **AuthN at the edge, AuthZ at every resource.** A valid token proves identity, not permission. Check ownership and role on every mutation and read.
- **Use parameterized queries only.** String-concatenated SQL/NoSQL/LDAP/shell is the oldest exploit still working. There is no safe way to interpolate user data into a query.
- **Secrets live in a secret manager.** Never in code, `.env` committed to git, container images, or logs. Rotate on exposure.
- **Least privilege everywhere.** Each service account, DB user, and IAM role gets exactly the permissions its code needs — nothing more.
- **TLS end-to-end.** Including east-west traffic inside the cluster. Plaintext internal calls are one misconfigured network away from leaking.
- **Rate-limit every public endpoint.** Especially auth, password reset, and anything that triggers an email/SMS. Protects both you and the user.
- **Hash passwords with a slow KDF.** `bcrypt`, `argon2`, or `scrypt` — never MD5, SHA-1, or raw SHA-256.
- **Never log secrets or PII.** Tokens, passwords, card numbers, full emails, and auth headers must be masked at the log appender, not left to the caller.
- **Pin and scan dependencies.** Lockfiles committed; SCA tool (Snyk, Dependabot, Trivy) blocking CVEs in CI.
- **Fail closed.** On any ambiguity in an auth or permissions check, deny access and alert.

## Red flags

- Any endpoint that returns another user's data when you flip the ID in the URL.
- `role == "admin"` check in frontend code only.
- Secrets committed to git — even in a "private" repo.
- Error responses that leak stack traces, SQL, or internal paths to the client.
- Dependencies more than a year behind on patches.
