# Authorization & responsible use

BugHunter is an **offensive** security tool. Active scanning sends real attack
payloads (SQLi probes, XSS markers, redirect tests, etc.) to the target.

Only use it against systems where you have **explicit written authorization**:

- applications and infrastructure you own, or
- targets that are **in scope** for a bug-bounty program whose rules you have
  read and are following, or
- a client engagement with a signed scope/contract.

The `scope` you declare is enforced: BugHunter refuses to send requests to
hosts outside it. That is a safety net, not permission. Testing out-of-scope or
unauthorised systems may be illegal (e.g. CFAA and equivalents) regardless of
tooling.

Rate-limit yourself (`rate_limit`) on production targets, respect program
rules on automation, and never test third parties without consent.
