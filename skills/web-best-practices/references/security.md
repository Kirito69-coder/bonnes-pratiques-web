# Security checklist

This reference lists security items to verify during a review. Adapt them to the project context.

## Secrets

- No secret, token or password in the repository.
- Secrets supplied by the environment or a secrets manager.
- `.env` and `.gitignore` configured so secrets are not committed.

## Passwords

- Passwords hashed with an appropriate algorithm (for example `password_hash()` with Argon2id in PHP when applicable).
- No plaintext storage or comparison of passwords.

## Validation and injection

- User input validated server-side.
- SQL queries use prepared statements or bound parameters.
- Output escaping applied when needed (for example `htmlspecialchars()` in PHP).

## Sessions and access control

- Session cookies configured securely (HTTPS, HttpOnly, SameSite where applicable).
- Access control enforced server-side, not only client-side.

## API and integrations

- Public keys exposed client-side only; secret keys kept server-side.
- CORS restricted to allowed origins.
- Rate limiting considered for public endpoints and sensitive actions.
- CSRF protection for sensitive forms when applicable.

## HTTPS

- HTTPS enabled across the project, with redirects from HTTP.
- Valid certificate and updated TLS configuration.

## Dependencies

- Dependencies kept reasonably up to date.
- Vulnerabilities checked with available tooling (for example `composer audit`, `npm audit`) when applicable.

## Backup and recovery

- Automatic backups configured.
- Restore procedure tested.

## Logging and monitoring

- Errors logged for production debugging.
- Security-relevant events recorded where relevant.

## Notes

- These items are verification aids, not a certification.
- Legal and security decisions should be validated with an official source or a competent professional when relevant.
