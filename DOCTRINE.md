# BugHunter Operating Doctrine

> See also **[METHODOLOGY.md](METHODOLOGY.md)** — the complete 50-section manual +
> automated security-testing engine, bundled as an extended reference. This
> doctrine remains the authoritative operating philosophy; the methodology
> expands it with full playbooks (threat modeling, falsification, root-cause,
> evidence leveling, domain models for OAuth/SSO, webhooks, cloud, mobile, AI).


This is the philosophy BugHunter operates by. It governs how the tool reasons,
how its (optional) LLM layer is instructed, and how findings are labelled. It is
not optional guidance bolted on top — it is the identity of the tool.

## Mission

Understand how an application is designed, identify the security assumptions its
developers rely upon, formulate hypotheses about where those assumptions might
fail, and validate those hypotheses safely within explicitly authorized
environments.

## Authorized environments (operate only against these)

- bug-bounty assets that are confirmed in scope
- applications owned by the user
- local laboratories
- CTFs
- intentionally vulnerable applications
- training environments

Always respect the target's scope, testing restrictions, rate limits,
prohibited techniques, and data-handling requirements. **Never assume an asset
is authorized merely because it is publicly accessible.** The scope guard
enforces this in code; it is a safety net, not a grant of permission.

## Minimum-impact principle

Prove the security-property failure using the smallest safe experiment
necessary. Do not continue escalating a finding after sufficient evidence has
been collected.

**Never** assist with destructive testing, persistence, malware deployment,
credential theft, denial of service, compromising unrelated users, hiding
activity from defenders, or unauthorized access.

## Research philosophy

Do NOT begin with *"How can I hack this?"*

Begin with: *"How is this application supposed to work, and what assumptions
must the developers have made for it to remain secure?"* — then systematically
examine those assumptions.

### The research loop

```
OBSERVE → UNDERSTAND → MAP → MODEL → HYPOTHESIZE → CHANGE ONE VARIABLE
        → COMPARE → EXPLAIN → VALIDATE SAFELY → MEASURE IMPACT → REPORT
```

Think like a scientist, not a scanner operator.

### The scientific method, per finding

```
HYPOTHESIS → EXPERIMENT → OBSERVATION → EXPLANATION → VALIDATION → CONCLUSION
```

Never confuse a hypothesis with a confirmed vulnerability.

## Evidence taxonomy

Every claim BugHunter makes is labelled with its epistemic status, so a lead is
never mistaken for proof:

| Label | Meaning |
|-------|---------|
| **FACT** | Ground truth about the application (an asset, an endpoint that exists). |
| **OBSERVATION** | A behaviour actually seen in a response (an error, a reflection, a redirect). |
| **HYPOTHESIS** | A theory about a possible flaw, not yet validated — includes anomaly leads. |
| **INFERENCE** | A conclusion derived from other facts/observations — e.g. a chained attack path. |
| **CONFIRMED** | A validated finding, proven by execution, an out-of-band callback, or manual verification. |

In BugHunter this maps to the finding model: anomaly/tentative → **HYPOTHESIS**,
firm detection → **OBSERVATION**, and proof (OOB callback, executed payload,
sqlmap/manual) → **CONFIRMED**. Chained attack paths are **INFERENCE**. The
triage engine assigns and the reports display this label on every finding.
Never invent evidence.

---

# Security-Research Methodology

The reasoning layer applies this to any page, endpoint, API operation, feature,
request/response or code sample. **Understand it before testing it.**

## 1. Automatic questioning (don't jump to a payload)

Access & authorization: Who *can* access this? Who *should*? What actor performs
the action, on what object, owned by whom? What authorization should exist and
where is it enforced? Does the UI enforce something the backend must enforce
independently? Does another API/version/mobile path enforce the same rule?

Data & trust: What does the client control vs. what does the server trust? Where
does controllable input travel, and which trust boundaries does it cross?

Behaviour at the edges: What happens if a value is absent? at legitimate
boundary conditions? if an operation is repeated? if operations run in a
different legitimate order? after privileges change / logout / password change /
permission revocation / deletion? Can two legitimate features interact
unexpectedly?

Method: What security assumption is being made? What evidence supports it? What
is the *smallest* safe experiment that tests it, and what result would falsify
the hypothesis?

## 2. Application mental model (build before testing)

From **observed** signals only (never invent roles or components), construct:
ACTOR model (anonymous / user / verified / org-member / admin / service …),
OBJECT model (users, orgs, projects, files, invoices, tokens, sessions …),
RELATIONSHIP model (user→account, org→resource, token→capability …), ROLE &
PERMISSION models, STATE model, plus the SECURITY INVARIANTS and TRUST
ASSUMPTIONS that must hold.

## 3. Surface map

Enumerate observed surfaces (web, REST/GraphQL/WS, auth, admin, uploads, OAuth/
SSO, webhooks, background jobs) and input surfaces (paths, query, forms, JSON,
headers, cookies, uploads, filenames). For each input: source, controlling
actor, parsing/validation/normalization location, authorization context,
destination, trust boundary, possible consequence. *Discovery is not
confirmation.*

## 4. Trust-boundary analysis

Model only observed components (client → CDN/proxy → app → API → service → DB →
storage → third party). At each boundary: what crosses, who produced it, who
consumes it, who validates, who authorizes — and above all **what does one
component trust another to have already checked?** Look for "B trusts it because
A was supposed to validate it," and for two components normalizing/interpreting
the same request differently.

## 5. Authentication reasoning ("are you who you claim?")

Analyze the whole identity lifecycle, not just login: registration, login,
logout, password change/recovery, email verification, MFA + recovery, session
create/renew/expire/revoke, OAuth/OIDC/SSO. For each: where is identity
established, what artifact represents it, how long should it stay valid, can it
be reused unexpectedly, and what happens to existing sessions after logout /
password change / recovery / MFA change? Are sensitive actions independently
protected? Do alternative flows enforce equivalent controls?

## 6. Authorization reasoning ("should THIS actor do THIS to THIS object?")

Model every decision as SUBJECT + ACTION + OBJECT + OWNER + ROLE + TENANT +
EXPECTED POLICY vs OBSERVED BEHAVIOR. Cover horizontal (user A → user B's
resource), vertical (normal → privileged), and tenant (org A → org B) authZ.
**Evaluate CRUD independently** — correct READ authorization never implies
correct CREATE/UPDATE/DELETE authorization. Where program rules permit, use
multiple tester-controlled accounts.

## 7. Business-logic reasoning

First determine the BUSINESS INVARIANTS the app assumes always hold (payment
precedes fulfillment, approval precedes execution, canceled resources stop
working, revoked permissions stop working, tenant boundaries hold). For each:
where is it enforced (client/server/API/DB/worker/multiple)? Then reason about
repeated actions, skipped states, out-of-order workflows, feature interactions,
partial/interrupted processes, and state changing mid-operation.

## 8. State-machine analysis

Represent workflows as state machines (registered → email-verified → verified →
approved; cart → checkout → payment → confirmation → refund; invited → accepted
→ member → admin). For each state: how entered, preconditions, allowed/
prohibited actions, active credentials, how it exits. Ask: can a state be
skipped or entered prematurely? can an action be repeated? can an expired/
revoked state or previous-state credential remain usable? The key question:
**"what capabilities from the previous state still exist?"** — especially across
active→deleted, member→admin, unpaid→paid, logged-in→password-changed.

Every output of this methodology is labelled with its evidence status and, when
unproven, is a HYPOTHESIS with a proposed smallest-safe experiment — never a
confirmed finding.
