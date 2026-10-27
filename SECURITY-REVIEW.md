# Security review — 2026-10-28

## Scope and requirements

Scan the cloned demo, upgrade dependencies affected by known advisories, preserve application behavior, and record remaining issues before the separate build/dependency repair phase. This is a dependency audit and targeted source review, not a penetration test. No new application mechanism was added.

## Dependency audit

Counts are affected packages (including transitive dependencies), not unique CVEs or confirmed exploitable endpoints. Results depend on the npm advisory database at scan time.

| Scan | Critical | High | Moderate | Low | Total |
| --- | ---: | ---: | ---: | ---: | ---: |
| Before | 3 | 16 | 5 | 5 | 29 |
| After | 0 | 0 | 0 | 0 | 0 |

## Installed direct versions

- mysql2: 3.24.6
- express: 4.22.3
- express-session: 1.19.0
- bcrypt: 6.0.0
- argon2: 0.45.1
- nodemon: 3.1.14

Compatible transitive security updates are captured in package-lock.json. Use npm ci to reproduce this tree.

## Evidence and unresolved application findings

- mysql2: critical code-execution and other advisories required the 3.x upgrade. See https://github.com/advisories/GHSA-fpw7-j2hg-69v5.
- bcrypt/argon2: updated their native installation dependency chains to remove vulnerable node-pre-gyp/tar. Nodemon 3 removes the vulnerable notifier/semver chain.
- High: routers/reviews.js:24 exposes POST /post-review without checkLogin; controller/reviews.js uses caller-supplied member_id to validate ownership and insert/update reviews. Anonymous callers can impersonate a member if the corresponding book ownership exists. Add authentication and derive identity from the session.
- High: routers/order.js:13,27 uses caller-supplied member_id despite checking login. Derive user ID from the session to prevent cross-account order reads.
- High: routers/market.js:43,49,58,66 trusts caller-supplied prices and memberId. Derive membership from the session and calculate prices from the database. Order creation also needs a transaction and the INSERT result's ID instead of reading the last order ID.
- High: routers/bookshelf.js:20 exposes owned-books without authentication; models/bookshelf.js:19 returns member 1's books. Authenticate and scope the query to the session user.
- Medium: routers/auth.js:90 includes the password hash in the session member and login response. controller/member.js returns that session member as well. Return an allowlist of public fields and remove hashes from sessions. Login also does not regenerate the session ID.

These application issues remain unchanged in this dependency-first pass. Their severity is a source-review assessment; no live database exploitation was performed.

## Verification

- npm audit: zero known dependency vulnerabilities.
- npm ls --depth=0: passed.
- bcrypt hash/compare round trip: passed.
- mysql2 promise pool initialization and shutdown: passed without connecting to a database.
- Backend JavaScript parse checks: passed.
- Basic private-key/AWS/GitHub-token pattern scan: no matches; not a complete secret or history scan.
- Database queries, authentication flows, and HTTP integration are not verified. No database dump was present in either clone.

## Runtime verification

The local runtime is Node.js 24.22.1, the patched 24.x release in the October 21, 2026 security bulletin: https://nodejs.org/en/blog/vulnerability/october-2026-security-releases. The repository now records it in .nvmrc. This file does not automatically switch a running process or guarantee that CI, containers, or another machine use the same version. The application-level findings above remain unresolved.
