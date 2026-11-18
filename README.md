# BugHunter  (v1.17.1)

**Fixed in v1.17.1 (auth is strictly non-fatal):** authentication is an additive
overlay — any failure in sign-up, login, verification, or mid-scan session renewal
now degrades to the normal unauthenticated scan instead of aborting it. Discovery,
account provisioning, primary login, the per-subdomain pass, and session renewal
are each guarded so an unexpected error is logged and the scan continues and
completes (status `done`), exactly as it did before authentication existed.

**New in v1.17 (authenticate across subdomains):** after subdomain enumeration,
BugHunter now finds which live subdomains have a login and gets authenticated on
each one, so the authenticated scan covers the whole estate — not just the apex.
For a subdomain with its own sign-up it registers there (temp mailbox, same flow
as the primary); for a subdomain that shares the target's auth (the common
`Domain=.root` SSO-cookie case) it reuses the primary session. Sessions are now
tracked per-host, so each subdomain carries the right credentials. The scan
records an **auth-surface map** (`_brain.auth_surface`: which hosts have
login/signup and which were authenticated, and how). Runs when `--register` is set
or a primary session exists; the per-host fan-out is capped. Honest limits are
unchanged — disposable-mail blocks and CAPTCHA/SMS still require the own-domain
IMAP or paste-a-cookie fallback, now applied per subdomain.

**New in v1.16 (target-aware auto-registration):** registration now follows the
domain you scan — no separate URL to configure. Run `bughunter scan <target>
--register` and BugHunter probes the target for its sign-up page (`/signup`,
`/register`, `/join`, `/users/sign_up`, …), discovers the login page too, creates
test accounts there using the built-in temp mailbox, completes e-mail/OTP
verification, logs in, and scans the authenticated surface — all against the exact
domain you entered. `--register-count N` sets how many accounts (default 2, which
unlocks IDOR/authz). Config equivalent: `"register": { "auto": true }`. If no
sign-up page is found it says so and scans unauthenticated. (Same disposable-mail
caveat as v1.15 — for targets that block throwaway domains, point `register.email`
at an IMAP mailbox on a domain you own.)

**New in v1.15 (zero-config temp mailbox — fully hands-off):** you no longer need
to provide a mailbox at all. Just set `register.url`, and BugHunter provisions its
own disposable inbox (built-in mail.tm provider), signs up with it, reads the
confirmation link or OTP code from that temp inbox, verifies, logs in, and scans —
nothing to configure. Explicit form: `"register": { "url": "https://app/signup",
"count": 2, "email": { "backend": "mailtm" } }` (the `mailtm` backend is also the
automatic default whenever registration is requested with no mailbox set). Needs
outbound HTTPS to `api.mail.tm`. **Honest caveat:** disposable-mail domains are
widely blocked by real sign-up forms — where the target refuses throwaway
addresses, switch `register.email` to an IMAP mailbox on a domain you own (v1.14).

**New in v1.14 (fully-automated e-mail / OTP verification):** account creation now
completes the confirmation step by itself. Point `register.email` at a mailbox the
tool can read (IMAP for a domain you own — use plus-addressing so every test
account's mail lands in one inbox) and BugHunter will, after sign-up, poll for the
verification e-mail, extract either the **confirmation link** (and follow it) or the
**numeric OTP code** (and submit it to the verify form), then log in and scan — no
human in the loop. Backends: `imap` (reliable, recommended) and `http` (generic
disposable-mailbox JSON API, best-effort). It stays honest: if no mailbox is
configured, or the mail never arrives, or the flow is gated by CAPTCHA/SMS, it says
so instead of reporting a dead account as usable. Config:
```json
"register": {
  "url": "https://app/signup", "count": 2, "login_url": "https://app/login",
  "verify_url": "https://app/verify", "address_template": "you+bh{tag}@you.com",
  "email": { "backend": "imap", "imap_host": "imap.you.com",
             "imap_user": "you@you.com", "imap_pass": "APP_PASSWORD" }
}
```

**New in v1.13 (automated test-account creation):** point `register` at a target's
sign-up URL and BugHunter will create its own labelled test accounts for
authenticated + IDOR/authz scanning — it reads the sign-up form (handling quoted
*and* unquoted HTML), maps the email/username/password/confirm fields, carries the
CSRF token (fetched fresh per account so single-use tokens aren't reused), accepts
required "agree to terms" boxes, submits, and captures the session (logging in
afterwards if the app doesn't auto-login on sign-up). It creates N accounts and
auto-populates both the primary authenticated session and the multi-account authz
matrix. It is honest about what it can't do: CAPTCHA, e-mail verification and MFA
stop automated sign-up, and it says so (and tells you to paste a cookie instead)
rather than reporting a dead account as usable. Runs only against the authorised,
in-scope target, labels every account (`bhtest_…` / `bughunter+…@`) for easy
cleanup, and records the created credentials in the scan result. Config:
```json
"register": { "url": "https://app/signup", "count": 2, "login_url": "https://app/login", "success": "Log out" }
```

**New in v1.12 (human-grade confirmed proof):** every finding the tool *confirms*
now carries a structured, submit-ready proof record — the **exact reproduction
request** that proved it (captured live during verification, not reconstructed),
the **response marker** that is the proof, plain-English **why-exploitable**
reasoning, and step-by-step reproduction — alongside the full rating block
(**Severity, CVSS 3.1 vector + score, CWE, OWASP Top 10, CVE, CISA KEV, EPSS**).
The HTML report shows a green **"CONFIRMED — no manual re-verification needed"**
panel for each, and the JSON export gains a `confirmed_vulnerabilities` section
listing only the proven findings with their ratings and proof. Findings the tool
could *not* prove are never dressed up as confirmed — they stay in a clearly
separate queue. This is the "a human confirmed it" standard: for the bug classes
that can be proven safely and non-destructively (SQLi, XSS, LFI, SSTI, cmd
injection, open redirect, CORS, CRLF, and OOB-confirmed SSRF/XXE/blind-XSS), the
tool proves them and shows the receipt; for classes that genuinely require human
judgment or a destructive action to prove (business logic, some IDOR, RCE-by-
execution), it says so honestly rather than faking a confirmation.

**New in v1.11 (real-world precision):** three fixes driven by live scans where
the scanner was hitting a CDN/WAF edge rather than the application —
(1) **unproven injections are downgraded**: when an active verifier runs against
an injection finding and cannot reproduce it, the finding drops to tentative and
is capped to Info (never left High/"real"); (2) **CDN/WAF awareness**: findings on
edge infrastructure (`/cdn-cgi/`, `__cf_bm`, `cf-ray`, Akamai/Imperva/Fastly
markers) are tagged `cdn-infrastructure` / likely-out-of-scope and capped to Info —
genuine app bugs behind the CDN are *not* affected; (3) **Set-Cookie parser fix**:
no longer mis-parses an `Expires=…GMT` date or an attribute keyword as a cookie
name. Benchmark still 100% recall / 100% precision / 0 FP on the test server.

**New in v0.4:** auto-authentication (log in and scan the authenticated surface),
a **coverage & residual-risk** report that lists exactly what was *not* tested so
nothing is silently missed, an adaptive **LLM agent loop** (observe→hypothesize→
test→adjudicate; needs an API key), a headless-browser **SPA/DOM-XSS crawler**
(needs Playwright), webhook **notifications** for monitoring, and a tech→CVE
**vuln-intel** advisory layer.

> On "never miss / zero-day": no scanner can guarantee completeness or find
> zero-days on demand — that's a hard limit. BugHunter's answer is to never
> *silently* miss: it maximises coverage and then explicitly reports every gap it
> could not test, with a manual follow-up for each.


An internal **web-application vulnerability assessment platform** — your own
Burp-style workflow for authorised bug-bounty and pentest testing, with a
reasoning layer that adapts to each target, confirms blind bugs out-of-band,
triages its own findings, and writes submission-ready reports.

**New in v0.3:** triage engine (exploitability verdict + false-positive killing),
out-of-band server for **blind** SSRF/XSS/RCE, **IDOR/BOLA** authorization testing,
learning memory that improves each run, WAF detection + auto-evasion, CVSS 3.1
scoring, proxy + repeater + intruder, scope import with exclusions, continuous
monitoring with diff alerts, and one-click **HackerOne/Bugcrowd submission reports**.

It runs the full assessment pipeline:

```
recon / info gathering → hypotheses → scanning → anomaly sweep → chaining → report
  (subdomains, hosts,     (the brain   (15 checks   (differential   (attack    (SOP +
   ports, tech, crawl,     infers what   + 60-tool    fuzzing for     paths      walkthrough +
   content discovery)      to test)      arsenal)     unknowns)       by impact) remediation)
```

**The "thinking" layer (what makes it more than a scanner):**
- **Hypothesis generation** — reads the attack surface and infers which bug
  classes are plausible where, and which tools would confirm them.
- **Technique/payload mutation** — one logical probe becomes many concrete
  attempts across encodings and WAF-evasion variants.
- **Adaptive planning** — picks the tools/checks that fit what recon found,
  rather than blindly running everything.
- **Anomaly sweep** — differential-fuzzes parameters to surface *candidate*
  novel issues (leads for manual review — the honest version of "find unknowns").
- **Finding chaining** — combines issues into ranked attack paths.
- **Optional LLM reasoning** — off by default; add an API key to deepen it.

> Honest scope: this gets you a strong, adaptive assistant that finds and
> chains a wide range of issues and surfaces leads. It does **not** autonomously
> discover true zero-days — that remains human work; the tool accelerates it.

- **Runs locally / internally.** The dashboard binds to `127.0.0.1` by design.
- **Zero dependencies to start.** Pure-Python engine (stdlib only) — runs today.
- **Gets stronger with tools.** Auto-detects `nuclei`, `nmap`, `subfinder`,
  `httpx`, `ffuf` on your `PATH` and orchestrates them when present.
- **Manual + automated findings** live together and export to one report.

> ⚠️ **Authorised testing only.** Active scanning sends real payloads. Only
> point BugHunter at assets you own or are explicitly authorised to test
> (your apps, in-scope bug-bounty programs). The scope guard refuses anything
> outside the scope you declare, but you are responsible for staying in scope.

BugHunter operates by a stated **[operating doctrine](DOCTRINE.md)** — a
scientist's hypothesis-driven, minimum-impact method, with every finding
labelled by evidence status (HYPOTHESIS / OBSERVATION / INFERENCE / CONFIRMED)
so a lead is never mistaken for proof. The same doctrine is the system prompt
for the optional LLM reasoning layer. A complete 50-section reference —
**[METHODOLOGY.md](METHODOLOGY.md)** — ships alongside it as the full manual +
automated testing playbook.

---

## Quick start (no installs needed)

```bash
cd bughunter

# 1) Full scan of a target you are authorised to test
python3 -m bughunter scan example.com --name prod

# 2) Passive-only (no payloads sent) — safe for first recon
python3 -m bughunter scan https://app.example.com/ --no-active

# 3) Launch the local web dashboard
python3 -m bughunter serve
#   -> open http://127.0.0.1:8777

# which optional external tools are detected?
python3 -m bughunter tools

# blind-vuln testing with the out-of-band listener (point --oob-host at a
# place the target can reach you — a VPS/tunnel; defaults to a local listener)
python3 -m bughunter scan target.com --oob --oob-host your-vps.example:8971

# auto-authenticate, then scan the logged-in surface
python3 -m bughunter scan app.example.com \
  --login-url https://app.example.com/login --login-user you --login-pass '***'

# turn on the adaptive LLM agent (needs BUGHUNTER_LLM_* env). The SPA/browser
# crawl and JS secret/endpoint mining are ON by default (auto-degrade if missing)
python3 -m bughunter scan app.example.com --llm-agent

# polite, safe client scanning: throttle per host + hard request kill-switch
python3 -m bughunter scan app.example.com --rate-limit 0.5 --max-requests 5000

# verify an engagement's request log was not tampered with
python3 -m bughunter verify-audit reports/audit_<scan_id>.jsonl

# load scope (and out-of-scope) from a file or a bug-bounty platform JSON export
python3 -m bughunter scan --scope-file program_scope.json --name acme

# write HackerOne/Bugcrowd submission-ready reports (one .md per real finding)
python3 -m bughunter submit <scan_id> --out reports/submissions

# continuous monitoring: re-run and show only what's NEW since last time
python3 -m bughunter monitor target.com --name acme   # cron this for hands-off watching

# list past scans / regenerate a report
python3 -m bughunter list
python3 -m bughunter report <scan_id>
```

Reports are written to `./reports/report_<name>.html` and `.json`.

## Reaching more surface: browser crawl + JS intelligence

Modern apps hide most of their attack surface behind JavaScript. BugHunter now,
**by default**:

- renders pages in a real headless browser (Playwright/Chromium when installed)
  to capture client-side routes and XHR/fetch endpoints, and feeds them into the
  vulnerability scan — not just a log line. Without Playwright it auto-degrades to
  the HTTP crawler and records the gap (`pip install playwright && playwright install chromium`).
- downloads the site's JavaScript and mines it for **hidden endpoints/parameters**
  (merged into the scan surface) and **leaked secrets** — AWS/Google/Stripe/Slack/
  GitHub keys, private-key blocks, JWTs — with an entropy gate to keep noise down.

Disable with `--no-browser-crawl` / `--no-js-intel` if you need a lighter scan.

## Proof-carrying verification (v1.6) — true positives, not guesses

Detection finds candidates; **verification proves them.** After scanning, BugHunter
re-exploits each high-value finding to extract a concrete, expected result the
server could only produce if the bug is real, and only then marks it CONFIRMED with
the proof attached:

- **SQLi** — a stable `1=1` vs `1=2` boolean differential
- **LFI** — the response actually contains `/etc/passwd`'s root line
- **SSTI** — a unique expression is evaluated (`1337*1337` → `1787569`)
- **Command injection** — an arithmetic echo executes (`echo CI$((7*7))CI` → `CI49CI`)
- **Open redirect** — the `Location` header points at an attacker-controlled host
- **Reflected XSS** — context-classified (html/attribute/tag/script) and proven to
  land **unescaped** in an executable context
- **Time-based blind** — confirmed by consistent delay across trials **plus** a
  zero-delay control that stays fast (kills the "slow endpoint" false positive)

A verifier only ever *upgrades* a finding to CONFIRMED and attaches evidence — it
never invents findings and never downgrades a real one — so precision can't drop.
Findings that prove themselves are the ones you can submit with confidence.

## Production features (v1.5)

- **Deduplication / correlation** — the same bug across many subdomains and
  parameter values collapses into one issue (with an instance count and the full
  affected-location list), instead of 50 duplicate rows that get a bounty program
  to reject you. Reports and the JSON export both show unique issues.
- **Benchmark harness** — `tests/benchmark.py` scores a run: precision, recall,
  true/false positives and false negatives against a labelled target, measured
  after dedup. Run it against the bundled server, or point it at OWASP Juice Shop
  with your own ground-truth file for a real TP/FP rate:
  ```bash
  python3 tests/benchmark.py                                   # bundled server
  python3 tests/benchmark.py --target https://host --scope host --ground-truth app.json
  ```
- **OpenAPI / Swagger import** — `--openapi spec.json` (file or URL) generates a
  test target for every documented endpoint + parameter and merges them into the
  scan surface. Supports OpenAPI 3.x and Swagger 2.0.
- **Scan resumability** — an interrupted scan resumes from stored assets instead
  of re-crawling: `python3 -m bughunter scan --resume <scan_id>`.
- **Auth that survives long scans** — sessions are kept alive; if the session
  drops mid-scan (the protected URL bounces to a login wall) BugHunter
  re-authenticates automatically. CSRF/hidden tokens are harvested at login.
- **Program etiquette** — `--identifier-header 'X-Bug-Bounty: yourhandle'` tags
  every request so a program can attribute the (authorised) traffic to you, and
  `--scope-file` imports a HackerOne/Bugcrowd scope (in- and out-of-scope).

## Native nuclei-template engine (no binary required)

BugHunter runs real nuclei YAML templates natively — no nuclei binary, no Go. It
reads the template's requests, matchers (word/regex/status/binary/dsl), conditions
and extractors and executes them with its own HTTP client. A small starter set
ships in `bughunter/nuclei/templates/`; point it at the full corpus to run thousands:

```bash
git clone https://github.com/projectdiscovery/nuclei-templates
python3 -m bughunter scan app.example.com --nuclei-templates-dir ./nuclei-templates
# filter: only run high/critical CVE + exposure templates
#   "nuclei_severity": ["high","critical"], "nuclei_tags": ["cve","exposure"]
```

Each match becomes a finding with the template's severity, references and CWE.

## Software-composition analysis (vulnerable libraries + outdated server)

BugHunter fingerprints client-side library versions (from script URLs and version
banners) and server/framework versions (from banners), then flags known-vulnerable
ones with their CVE. With feeds enabled it queries **OSV.dev** live for the full,
current advisory list per `library@version`; offline it uses a bundled database.

## Official vulnerability data — stay current with NVD / OSV / CISA KEV / EPSS

Ratings are backed by official sources, not guesses:

- **NVD** — canonical CVSS v3.1 vector/score and CWE for any CVE (fetched on demand, cached).
- **OSV.dev** — authoritative, current library advisories for the SCA check.
- **CISA KEV** — if a finding's CVE is *actively exploited in the wild*, it's forced to the top.
- **EPSS** — exploit-probability score feeds triage prioritisation.
- **OWASP Top 10 (2021)** — every finding is tagged with its category (e.g. `owasp:A03:2021`) via its CWE, fully offline.

```bash
python3 -m bughunter update            # sync the CISA KEV catalog (NVD/OSV/EPSS are on-demand)
```

A bundled offline seed (real CVSS values + a KEV snapshot) means enrichment works
with no network; `update` and on-demand lookups refresh it when you're online. Run
`update` from a nightly scheduled task to stay current automatically.

## Multi-account authorization testing (IDOR / BOLA / privilege escalation)

The highest-paying bug class needs more than one identity. Give BugHunter two or
more sessions (plus the implicit anonymous one) and it diffs what each can reach:

```jsonc
// config.json
"authz_identities": [
  {"name": "alice", "role": "user",  "headers": {"Cookie": "SESSION=alice"}},
  {"name": "bob",   "role": "user",  "headers": {"Cookie": "SESSION=bob"}},
  {"name": "admin", "role": "admin", "headers": {"Cookie": "SESSION=admin"}}
]
```

It detects: horizontal **IDOR/BOLA** (one user reads another's object, gated on the
endpoint being ID-sensitive), **unauthenticated access** to sensitive resources,
**vertical privilege escalation / forced browsing** to admin-tier endpoints, and
**role diffing** (a low role reaching high-role functionality). All GET-only and
minimum-impact; write-verb authz is raised as an explicit manual hypothesis.

## LLM agent — plug in your Claude API key

The reasoning agent is **grounded**: it builds a model of the app (actors, objects,
parameters, state/lifecycle endpoints) and reasons about business logic, not just
payloads. Without a key it runs a built-in heuristic reasoner; with a key it uses
your model:

```bash
export BUGHUNTER_LLM_PROVIDER=anthropic         # or: openai
export BUGHUNTER_LLM_MODEL=<your-model-id>       # e.g. a current Claude model id
export BUGHUNTER_LLM_API_KEY=sk-ant-...          # your key — never hard-coded
pip install anthropic                            # (or: pip install openai)
python3 -m bughunter scan app.example.com --llm-agent
```

Get an Anthropic key and check your credit balance at console.anthropic.com
(Settings → API Keys, and the Billing/Usage page). The key is read only from the
environment variable — it is never written to disk, logs, or reports.

## Safety for client engagements

- **Hard scope guard** on every request; nothing outside the declared scope is sent.
- **Reserved-range block**: private/loopback/link-local and the cloud-metadata IP
  (169.254.169.254) are refused unless *explicitly* in scope (stops accidental
  internal scanning / SSRF pivots). Override in a lab with `--allow-private`.
- **Per-host rate limiting with adaptive backoff** (`--rate-limit`, honours
  `Retry-After` and slows down on 429/503) and a **request kill-switch**
  (`--max-requests`) so a scan can never run away against a client.
- **Tamper-evident audit log**: every request is hash-chained to the previous one;
  `bughunter verify-audit <file>` proves the log was not altered after the fact.

## Authenticated scanning

Pass a session cookie / token header so the crawler and checks reach
logged-in pages:

```bash
python3 -m bughunter scan https://app.example.com/ \
  --header "Cookie: session=eyJ...; csrftoken=abc" \
  --header "Authorization: Bearer <token>"
```

In the dashboard, use the **Auth header** field on the New-scan form.

## What it detects today (built-in, pure-Python — 32 checks)

Native coverage spans the OWASP Top 10 (strong on A01/A03/A05/A10; see the
coverage report for what any given scan did *not* reach):

| Check | Severity | Class |
|-------|----------|-------|
| Error-based SQL injection | critical | CWE-89 |
| NoSQL injection | high | CWE-943 |
| XXE (file disclosure + entity expansion) | critical | CWE-611 |
| JWT weaknesses (alg:none, weak secret, no exp) | critical | CWE-347/321 |
| Unrestricted file upload | high | CWE-434 |
| Subdomain takeover (dangling DNS) | high | CWE-350 |
| CSRF (missing anti-CSRF token) | medium | CWE-352 |
| TLS/crypto (cleartext HTTP, bad/expired cert) | medium/high | CWE-319/295 |
| Excessive data exposure (API) | medium | CWE-213 |
| Mass assignment (manual-verify) | medium | CWE-915 |
| Insecure deserialization surface | medium | CWE-502 |
| Web cache poisoning | high | CWE-444 |
| Prototype pollution | medium | CWE-1321 |
| Auth hardening (no rate-limit / user-enum) | medium | CWE-307 |
| Blind SSRF / XSS (out-of-band confirmed) | high | CWE-918/79 |
| IDOR / BOLA (ID enumeration + two-session diff) | high | CWE-639/285 |
| OS command injection (arithmetic-marker, no reflection FPs) | critical | CWE-78 |
| Server-Side Template Injection | high | CWE-1336 |
| Local File Inclusion / path traversal | high | CWE-22 |
| SSRF (candidate + metadata reach) | medium/critical | CWE-918 |
| Reflected XSS (multi-encoding) | high | CWE-79 |
| Exposed secrets (`.env`, `.git`, backups, actuator) | high/critical | CWE-200/527 |
| CORS misconfiguration | medium/high | CWE-942 |
| Open redirect | medium | CWE-601 |
| Host header injection | medium | CWE-644 |
| GraphQL introspection enabled | low | CWE-200 |
| Directory listing | medium | CWE-548 |
| Missing/weak security headers (CSP, HSTS, …) | info–medium | CWE-693 |
| Insecure cookie flags | low | CWE-614 |
| Version/software disclosure | info | CWE-200 |

**Highest-level detection:** boolean-based blind SQLi (TRUE/FALSE response differential), time-based blind SQLi/command injection, and **out-of-band confirmation** of blind command injection and XXE (callback = proof), plus polyglot payloads. 

Plus an **anomaly sweep** that flags parameters behaving oddly as leads for
manual review.

## Tool arsenal (60 adapters)

`python3 -m bughunter tools` shows every integrated tool and whether it's
installed. Each auto-activates when its binary is on `PATH`; results merge into
the same pipeline. 17 have tuned parsers (nuclei, httpx, subfinder, naabu, nmap,
katana, gau, ffuf, arjun, sqlmap, ghauri, dalfox, nikto, gitleaks, trufflehog,
assetfinder, waybackurls); the rest are declared adapters across recon, content/
param/JS discovery, scanning, per-vuln-class exploitation, auth/JWT, API/GraphQL,
secrets and proxy. Install whichever you want — the engine runs fully without any.

## Optional power-ups

```bash
# examples — install whichever you want; BugHunter uses them if on PATH
# nuclei   -> template/CVE scanning        (ProjectDiscovery)
# nmap     -> service/version port scan
# subfinder-> passive subdomain enumeration
# ffuf     -> fast content discovery (with --wordlist)
```

If none are installed, BugHunter falls back to its own implementations.

## Configuration

Flags cover the common cases; for repeatable runs use a config file:

```bash
python3 -m bughunter scan example.com --config config.example.yaml
```

See `config.example.yaml` for every option (threads, rate limit, ports,
crawl depth, enabled/disabled checks, custom wordlist, auth headers…).

## Architecture

```
bughunter/
  core/          models, config, scope guard, storage (SQLite), HTTP client, tool runner
  recon/         subdomains, live hosts, ports, tech fingerprint, crawler, content discovery
  scan/          check framework + checks/ (passive, active, injection) + nuclei
  integrations/  adapter framework + 60 tool adapters (native + declared), registry/doctor
  brain/         techniques (payload mutation), hypotheses, anomaly, chaining, planner, llm hook
  triage/        exploitability verdict + false-positive killing (heuristic + optional LLM)
  learning/      per-target memory that improves each run
  oob/           self-hosted out-of-band interaction server (blind-vuln confirmation)
  authz/         IDOR / BOLA authorization testing
  proxy/         logging proxy + repeater + intruder (manual-testing core)
  monitor/       snapshot + diff for continuous monitoring
  waf.py         WAF detection + adaptive evasion + rate control
  core/cvss.py   CVSS 3.1 scoring
  manual/        manual finding entry
  report/        HTML + JSON report + submission-ready (HackerOne/Bugcrowd) markdown
  web/           zero-dependency local dashboard (stdlib http.server)
  engine.py      recon → WAF → hypotheses → scan → anomaly → OOB → tools → triage → chaining → persist
  cli.py         command-line interface (scan, submit, monitor, serve, tools, list, report)
tests/
  vuln_server.py a deliberately-vulnerable local app (for validation only)
  validate.py    end-to-end test asserting all 14 vuln classes fire
```

### Extending — add a check in ~15 lines

```python
# bughunter/scan/checks/passive.py (or active.py)
from ..base import Check, ScanContext, register
from ...core.models import Severity, Confidence

@register
class MyCheck(Check):
    id = "my-check"
    name = "My new check"
    severity = Severity.MEDIUM
    active = False  # True if it sends payloads

    def run(self, ctx: ScanContext):
        for host in ctx.live_hosts:
            r = ctx.http.get(host["url"])
            if "something bad" in r.body:
                yield self.finding(ctx, title="Found it", target=host["url"],
                                   description="...", remediation="...",
                                   confidence=Confidence.FIRM)
```

It is auto-registered and runs on the next scan.

## Validate it yourself

```bash
python3 tests/validate.py
# starts a local vulnerable app, scans it, asserts all 9 classes are detected
```

## Roadmap (where we take it next)

- Intercepting proxy + request repeater/intruder (mitmproxy-driven manual core)
- Out-of-band confirmation (interactsh) for blind SSRF/SQLi/RCE/XSS
- More native tool parsers (wapiti, arachni, XSStrike, commix, graphw00f)
- Authenticated-session recording & replay; IDOR/authz differential testing
- CVSS vectors, triage workflow, cross-scan dedup, and new-finding diffing
- Deeper LLM reasoning loop (response analysis → next payloads)

---
*For authorised security testing only. You are responsible for ensuring you
have permission to test every target.*
