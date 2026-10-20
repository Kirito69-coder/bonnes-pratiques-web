# Project audit output example

This example shows the kind of output expected from a web-best-practices review. It is illustrative, not a real audit result.

## Project context

- Type: PHP/MySQL application with a public API
- Target: pre-deployment review

## Findings

### P0

- CORS: `Access-Control-Allow-Origin` uses a wildcard in the API configuration.
  - File: `config/cors.php`, line 12 (assumed)
  - Suggested correction: restrict allowed origins to known domains.
  - Risk: unauthorized cross-origin requests may consume the API.

- Secrets: a Stripe secret key appears in a client-side example file.
  - File: `public/js/stripe-example.js`, line 3 (assumed)
  - Suggested correction: remove the secret key and keep only the public key in client code.
  - Risk: secret key exposure could allow credential misuse.

### P1

- Backups: backup script exists but restore has not been tested recently.
  - File: `scripts/backup.sh` (assumed)
  - Suggested correction: schedule and document a restore test.
  - Risk: backup may not be usable in an incident.

- Error handling: some endpoints return stack traces in production-like configuration.
  - File: `app/config/debug.php`, line 4 (assumed)
  - Suggested correction: disable verbose error output in production and log errors instead.
  - Risk: sensitive information leakage.

### P2

- Accessibility: a decorative image has a descriptive alt text.
  - File: `templates/header.php`, line 8 (assumed)
  - Suggested correction: set `alt=""` for a decorative image.
  - Risk: unnecessary verbosity for assistive technologies.

- Performance: a frequently queried column lacks an index.
  - File: `schema/users.sql`, line 15 (assumed)
  - Suggested correction: add an index where relevant and verify with `EXPLAIN`.
  - Risk: slower queries under load.

### P3

- Favicon: missing in production build.
  - File: not found (assumed)
  - Suggested correction: add `favicon.ico` and the corresponding link tag.
  - Risk: minor professional polish issue.

## Human validation requested

- CORS, secrets and privacy-sensitive items should be validated by a human before changes are applied in production.

## Notes

- This is an example output only. Real findings depend on the actual project files and the reviewer.
