# Switchboard — Overall System Architecture (v0)

**Status:** Draft v3 — decisions ratified in stakeholder walkthrough 2026-07-12 and PR #1 review (see §13 Decision log); pending remaining sign-off · feeds the v0 specification
**Milestone:** M0 — Discovery, Requirements & System Design
**Asana:** [Overall system architecture](https://app.asana.com/1/1216510645821595/task/1216512588498291)
**Audience:** stakeholders reviewing the M0 design gate; authors of the per-component design stories (Registry, Exchange, Queues, Orchestrator, Ledger, Connector, Console)

---

## 1. Purpose and scope

This document defines how Switchboard's six components fit together before any build begins. It locks the component boundaries, the data flow between them, the envelope contract, and the cross-cutting concerns (identity/endpoint model, security posture, observability). Every other M0 design story plugs into this backbone, and the frozen v0 spec consolidates it.

**In scope:** logical architecture, contracts and invariants, end-to-end flows, a proposed reference stack (default-but-revisable).
**Out of scope:** per-component API surface detail (owned by the sibling M0 stories), visual/console design, cross-org federation (explicitly deferred — see §11).

## 2. System overview

Switchboard is a communication exchange for organizations where AI agents are first-class workflow participants alongside humans. It handles **addressing, routing, queuing, orchestration, and record-keeping** for human↔agent and agent↔agent interactions.

The guiding metaphor is the telephone switchboard operator: a **neutral mediator** that connects parties, manages congestion, and keeps records — but never injects its own logic into the conversation. Switchboard never knows what "review" means; it only knows that an envelope carrying the verb `respond` arrived on a thread.

**Driving use case.** A coding agent produces a PR. The switchboard routes it to two senior engineers' review agents; those agents may escalate to their humans; feedback flows back to the coding agent, which revises. Human coordination time is spent only where human judgment is needed.

### 2.1 Design principles (settled — don't relitigate casually)

These were settled through iteration and constrain every component design below.

| Principle | Consequence for architecture |
|---|---|
| **Intelligence at the edges** | Consent, prioritization logic, and escalation live in endpoints, built at agent development time. The switchboard stays lean and neutral — a common carrier. |
| **No personal routing layer** | Callers see the directory and choose whom to dial. Preference-based resolution is a possible later customization, not core. |
| **Slack-style participation** | Referencing an endpoint in a workflow requires no approval — visibility plus the right to decline via handshake is enough. Approval gates don't scale. |
| **Self-managing under normal load, human-controllable under pressure** | Queue tooling exists for exceptions (saturation), not routine management. |
| **Handshake over policy DSL** | A lightweight offer/accept protocol beats switchboard-hosted declarative rules. The switchboard never *authors* an endpoint's acceptance policy — and evaluates one only as a console-configured default for endpoints that supply no webhook of their own, which is the only way human endpoints are reachable at all (amended 2026-08-09, decision #16). |
| **No side channels** | Every interaction between endpoints flows through the switchboard and is ledger-recorded — including an agent escalating to its own human. |

## 3. Component architecture — six components, one substrate

Switchboard is six logical components on one substrate, deployed in v0 as **four services**: the **Exchange service**, the **Queue service**, and the **Registry service** each stand alone, and a single **control-plane service** hosts the Orchestrator and Ledger behind separate API paths (`/workflows`, `/ledger`) — see §10. The sixth component, the **Connector**, adds no deployable of its own: it co-locates with the Exchange and Queue services that call it (§3.7). Component boundaries are logical contracts, not necessarily process boundaries.

### 3.1 Topology

![System topology: endpoints at the edges; inside the switchboard the Exchange, Queue, and Registry services stand alone, the control-plane service hosts the Orchestrator and Ledger API paths, and the Connector sits outside all four services, dialled by both the Exchange and the Queue service to reach endpoints and carrying channel replies back inbound; the Console is a client](diagrams/topology.png)

<sub>Diagram source: [diagrams/topology.mmd](diagrams/topology.mmd)</sub>

There are exactly **two delivery paths** into an endpoint's queue: the Exchange (ad-hoc and first-contact calls, after a handshake) and the Orchestrator (workflow calls covered by standing offers accepted at workflow creation — no per-call handshake, so no per-call Exchange hop). Both paths converge on the same queue-write API, which is ledger-recorded — that chokepoint, not "everything flows through the Exchange", is what keeps the record complete.

Getting an envelope *out* of the queue and into the endpoint is the Connector's job (§3.7) for every endpoint that can't pull — which is most of them, and all humans. The Connector likewise has exactly **two egress callers**: the Exchange (offers) and the Queue service (push-profile delivery). Registry, Orchestrator, and Ledger never reach endpoints. The Console is a client of the same public APIs endpoints use, not a component.

### 3.2 Registry

**Owns:** the flat namespace of endpoints — one record per endpoint (see [registry-data-model.md](registry-data-model.md)).

- Humans and agents are **separate, co-equal endpoints**: `varsha@acme` (human), `review-agent@varsha.acme` (her agent). Agents can be personal or team-owned infrastructure (`payments-review@acme`).
- Each endpoint record carries **identity and reachability, not capabilities**: address, `org_id`, kind, a human-readable `display_name` and `description`, a registry-controlled `maintainer` (an IdP principal — user or group), and a typed `transport` (where the endpoint lives and how to reach it). `transport.provider` names one of the Connector's **delivery profiles** — `a2a`, `native`, `webhook`, `pull`, `console`, `chat`/`email`/`push` ([connector-layer.md](connector-layer.md) §3) — which is what the Connector dispatches on. There is **no capability manifest**: the switchboard does not store, or match against, an endpoint's accepted verbs or artifact types.
- The registry is the **discovery surface**: callers browse/search by description and choose whom to dial (no personal routing layer). Live status and queue depth are **composed at read time** from the Queue service, not stored here.

**Does not:** route, enforce fine-grained call graphs, or record what an endpoint can do. Reachability is coarse **org membership** (§6); the endpoint's right to decline at handshake does the rest.

**Interfaces:** register/update endpoint, resolve address (→ transport + reachability), query directory. The resolve is on the hot path of every offer.

### 3.3 Exchange

**Owns:** session establishment and envelope delivery.

- **Offer/accept handshake:** to open a session, the Exchange sends the callee *offer metadata only* — caller address, verb, artifact type (never the artifact itself), dialled out through the Connector (§3.7). The endpoint replies `accept`, `decline`, or `defer` (with optional retry-after). **No response within the handshake timeout = implicit decline.** How an endpoint decides is its own business.
- **Hosted default acceptance policy:** an endpoint that supplies no acceptance webhook gets a console-configured default the Exchange evaluates on its behalf — accept in-org, accept from an allowlist, or auto-defer while away. Without this, the 60 s timeout plus implicit decline means *every* offer to a human declines, since nobody answers a handshake while asleep or on leave. The endpoint still owns the policy; the switchboard only executes it when none is supplied. This amends the §2.1 principle "handshake over policy DSL" — see decision log #16 and [connector-layer.md](connector-layer.md) §9.
- On `accept`, the Exchange opens a session, delivers the envelope into the callee's queue, and streams every subsequent envelope on that session to the ledger.
- On `defer`, the Exchange schedules a retry (visible in the ledger, e.g. "queue saturated · retry scheduled 12:20").
- Emits **verb events** consumed by the Orchestrator for workflow joins.
- The Exchange carries **ad-hoc and first-contact** traffic. It is *not* the sole delivery path: workflow calls covered by standing offers are enqueued directly by the Orchestrator (§3.5). The completeness guarantee lives at the ledger-recorded queue-write chokepoint (§3.4), not here.

**Does not:** inspect artifact content, author acceptance policy, prioritize, or retry beyond the defer contract. It is handshake plus session, nothing else — the dialing itself belongs to the Connector. It evaluates an acceptance policy only in the hosted-default case above, and never one it wrote.

### 3.4 Queues

**Owns:** durable, per-endpoint delivery buffers. **Switchboard-owned** — endpoints do not host their own queues. Backed by **AWS SQS**, one queue per endpoint (see §10 for the abstraction seam).

- The queue-write API is the **single chokepoint** shared by both delivery paths (Exchange and Orchestrator). Every enqueue writes its ledger entry in Postgres *first*, then sends to SQS with an idempotency key; delivery is at-least-once and consumers deduplicate on `envelope_id`. This ordering is what makes the ledger complete by construction even with two writers.
- The ledger row doubles as a **transactional outbox** entry carrying a delivered-to-queue marker. If the SQS send fails — or the process dies between the Postgres commit and the send — the **sweeper** re-drives unmarked rows until SQS acknowledges; the sender only sees enqueue success after that acknowledgment. Duplicate sends from re-drives are harmless because consumers dedupe on `envelope_id`.
- The sweeper is a **background job owned by the Queue service**: it continuously scans the outbox for committed rows without the delivered marker (older than a short grace window, e.g. 30 s) and retries their SQS sends with backoff. Its lag is observable (§8) — a growing unmarked backlog is an alert condition.
- **Queues drain two ways.** Pull-profile agents drain their own queue and autoscale to a budget cap; for every push profile — which is most agents and all humans — the Queue service drains on the endpoint's behalf and hands each envelope to the Connector to dial (§3.7). Same queue, same ordering, same ledger treatment; only who initiates the last hop differs.
- **Humans see the same queue as a Slack-like inbox** — one queue abstraction, several presentations (console, chat, email, mobile).
- Entries carry priority, age, originating workflow (or `ad-hoc`), caller, verb, and artifact reference.
- **Human intervention only under saturation:** bump (reprioritize), pause, and "I'll take this call" (owner takes over an item destined for their agent). These are exception tools, not routine workflow.

**Does not:** decide acceptance (that already happened at handshake) or transform payloads.

### 3.5 Orchestrator

**Owns:** declarative, Step-Functions-style workflows over endpoints. Runs on **Temporal** (self-hosted, backed by the same Postgres — see §10).

- Shape: **trigger → fan-out → join on verb emissions → continuation.** A workflow never encodes what work means — joins key off verbs (e.g. "wait until both callees emit `respond` or any emits `reject`").
- **States are conversations.** Waits are durable across deploys and restarts and cost nothing while idle (Temporal). Each state carries a **wait cap declared by the workflow author (default 24 h, §9.4)**; a state exceeding its cap times out, fails the run fast, and is ledger-recorded.
- **Standing offers:** when a workflow is created, the Orchestrator performs the handshake with every referenced endpoint once, up front. Runs then execute without per-call friction and **fail fast on declines** at creation time rather than mid-run.
- **Direct enqueue:** because standing offers were already accepted, workflow calls skip the per-call Exchange hop — the Orchestrator writes straight to the callee's queue via the ledger-recorded queue-write API (§3.4).
- **Ad-hoc mode:** any endpoint can dial any other directly ("New call"), producing a single-session micro-workflow that goes through the full Exchange handshake, queueing, and ledger treatment.

**Does not:** host endpoint logic, or bypass the ledger-recorded queue-write chokepoint.

### 3.6 Ledger

**Owns:** the append-only record of every session, offer, and envelope — including agent→own-human escalations. **No side channels** means the ledger is complete by construction: every handshake is recorded by the Exchange, and every queue write — from either delivery path — is recorded at the queue-write chokepoint (§3.4). Entries are never edited; an **org-configurable retention window** may expire old entries per policy (corrections are new entries, never rewrites).

- Records: offers (with accept/decline/defer outcome and latency), session open/close, every envelope (metadata + artifact reference, not artifact bodies), workflow run lineage.
- **Queryable by decision, endpoint, or workflow** — "show every `approve` on payments-core this month", "everything `review-agent@mohit.acme` deferred".
- The ledger is the audit/governance surface and, per the business framing, part of the moat: the stateful, trust-heavy layer peer-to-peer agent protocols can't replicate.

**Does not:** store artifact bodies (references + digests only — envelope message bodies, being part of the conversation record, *are* stored), and is never consulted for routing decisions.

### 3.7 Connector

**Owns:** executing an endpoint's `transport` — reaching it, in both directions. See [connector-layer.md](connector-layer.md).

- **Two egress callers:** the Exchange dials offers through it (§3.3); the Queue service dials envelope delivery through it for push-profile endpoints (§3.4). It sits inside neither, for the same reason the queue-write chokepoint doesn't: two independent callers.
- **Delivery profiles.** `transport.provider` selects one — `a2a` (any dialable agent, self-hosted or managed), `native` (managed runtimes that don't speak A2A yet), `webhook`, `pull`, `console`, or `chat`/`email`/`push`. **Hosting and reach are orthogonal**: where an agent runs never determines how it's reached, and `pull` is the escape hatch for endpoints that can't accept inbound at all — not the self-hosting profile.
- **Ingress.** A human's Slack reply or a runtime callback maps back to an envelope — structured actions to `approve`/`reject`, free text to `respond` carrying the inline `body`. It re-enters through the Exchange's normal public API as a call from that endpoint, so there is no privileged inbound path.
- **Backpressure becomes `defer`.** A backend 429 or quota signal is surfaced as the endpoint's defer with retry-after, reusing the existing contract rather than adding a retry model.
- **Credential custody.** Dialing needs credentials: per-endpoint, scoped to one invoke action, short-lived where the provider supports it, never in the ledger (§7).

**Does not:** choose recipients (no routing, no capability matching — callers pick from the directory), inspect artifact content, own the ledger record (it emits; the Exchange and the queue-write chokepoint still record), bypass the queue, or store endpoint state.

**Interfaces:** `dial(endpoint, payload)` returning accepted / deferred-with-retry-after / failed, plus per-channel ingress adapters.

## 4. The envelope — the shared contract

Every payload that moves through the switchboard is wrapped in an envelope. This is the one schema all six components share, and it is what keeps Switchboard workflow-agnostic.

The product brief settles four fields: **artifact reference, thread ID, provenance, and verb**. Seven supporting fields were ratified in the 2026-07-12 review as *the proposal*. This doc pins only the field list and what each is for; the concrete wire schema is low-level design, drafted separately in [envelope-schema.md](envelope-schema.md) and finalized in the v0 spec.

| Field | Purpose | Origin |
|---|---|---|
| Verb | Exactly one of the universal set (§4.1) | Brief |
| Thread ID | Conversation identity; stable across revise loops | Brief |
| Artifact reference + type | Pointer to the work item; `type` is metadata the callee uses to decide at handshake — the switchboard neither stores a manifest nor matches on it. Bodies live outside the switchboard | Brief |
| Provenance (workflow run, on-behalf-of) | Which workflow run (or `ad-hoc`) produced it; the human/team an agent acts for | Brief |
| Envelope ID | Unique, switchboard-assigned identifier per message; dedupe and ledger reference | Proposed |
| From / To | Sender and recipient endpoint addresses | Proposed |
| Session ID | The handshaken session the envelope travels on (see invariants below) | Proposed |
| Artifact digest | Integrity check — recipients verify the fetched content matches what was offered | Proposed |
| In-reply-to | The prior envelope on the thread this one answers | Proposed |
| Priority | Queue ordering; surfaced in the console and adjustable under saturation | Proposed |
| Created-at | Timestamp; powers queue age and latency metrics | Proposed |
| Message body (optional) | Conversational text accompanying the verb (≤ 12 KB) — remarks, escalation reasons, rejection rationale, chat turns. The conversation is part of the record; the work product stays by-reference in the artifact | Ratified 2026-07-16 (PR #2) |

### 4.1 The verb set

One verb per envelope, from a small universal set. Workflow joins key off verb emissions; the console color-codes them; the ledger indexes them.

| Verb | Meaning (protocol-level, not domain-level) |
|---|---|
| `request` | Asks the callee to act on the artifact. Opens or continues a thread. |
| `respond` | Returns the callee's output for a `request`. |
| `revise` | Sends an updated artifact on an existing thread (e.g. after feedback). |
| `approve` | Terminal positive judgment on the thread's artifact. |
| `reject` | Terminal negative judgment. |
| `escalate` | Hands the thread up — typically agent → its own human. Recorded like any envelope; no side channels. |
| `inform` | One-way notification; no response expected, no join semantics. |

**Invariants:**

- The verb set is **closed in v0**. Extending it is a spec change, not a configuration option — every component and every workflow join depends on its stability. Channel events arriving through the Connector's ingress adapters (§3.7) map *onto* this set — a Slack button to `approve`, a prose reply to `respond` with a `body` — and never extend it.
- The switchboard never interprets artifact content; `artifact.type` is offer metadata the callee uses to decide, not something the switchboard matches or enforces.
- `thread_id` is the unit of conversation (a PR review across five turns is one thread); `session_id` is the unit of handshaken delivery. Ad-hoc and first-contact calls open a session per call; a standing offer opens one long-lived session per workflow–endpoint pair at workflow creation, and all direct-enqueued envelopes for that pair travel on it. A thread may span multiple sessions (e.g. an escalation opens a new session to the human).

## 5. End-to-end data flow

### 5.1 The handshake (session establishment)

![Handshake sequence: caller dials, Exchange resolves the callee's transport profile and org via the Registry, the Connector dials the offer out, and the callee's accept, decline, or defer — including host backpressure surfaced as a defer — is recorded to the Ledger](diagrams/handshake.png)

<sub>Diagram source: [diagrams/handshake.mmd](diagrams/handshake.mmd)</sub>

The offer carries **metadata only**. The artifact reference is delivered after acceptance, into the queue. Endpoint-side acceptance logic (checking load, maintainer rules, caller allowlists) is invisible to the switchboard by design.

### 5.2 Driving use case: high-impact PR review

`wf:high-impact-pr-review` — trigger: coding agent emits a PR diff; fan-out to two review agents; join on both emitting a terminal verb; continuation returns the bundle to the coding agent.

![PR-review workflow sequence: the coding agent's request triggers the run, the Orchestrator direct-enqueues to both review agents via standing offers, one agent escalates to its human through the Exchange, and the approve verb completes the join](diagrams/pr-review-flow.png)

<sub>Diagram source: [diagrams/pr-review-flow.mmd](diagrams/pr-review-flow.mmd)</sub>

Points to notice:

- The workflow calls took the **direct-enqueue path**: standing offers were accepted at workflow creation, so no per-call handshake — but every enqueue was still ledger-recorded at the queue-write chokepoint.
- The **escalation is a normal envelope** on the same thread — as a first-contact call it goes through the Exchange handshake before landing in Varsha's inbox. No side channels.
- The Orchestrator only saw **verbs**, never diff content or review semantics.
- If the run had instead produced `revise` feedback, the coding agent's revised diff re-enters on the **same `thread_id`**, and the workflow's next state waits on the next terminal verb — up to that state's declared wait cap (default 24 h, §9.4), after which the run times out and is ledger-recorded.

### 5.3 Ad-hoc call

Any endpoint can dial any other directly. An ad-hoc call is the degenerate workflow: one handshake, one session, same envelope contract, same ledger treatment, `provenance.workflow_run = "ad-hoc"`. The "New call" console flow (open question in the brief) is UI over this path, not a new mechanism.

### 5.4 Saturation and human intervention

When an agent endpoint's queue saturates (e.g. `review-agent@mohit.acme` at replicas 4/4, depth 31):

1. New offers to it get `defer` (endpoint decision) — the Exchange schedules retries and the ledger shows the defer trail.
2. The endpoint's **observed status** reads `saturated` (composed from Queue-service depth); callers and the Orchestrator's standing-offer checks see it.
3. The owning human intervenes **only if needed**: bump an item's priority, pause the queue, or take an item over ("I'll take this call"), which reassigns delivery to their own inbox. All three are ledger-recorded interventions.

### 5.5 Reaching a human

The escalation in §5.2 ends at a human's queue. Getting it to the person, and their answer back, is the round trip the Connector exists for.

![Escalation round trip: the review agent escalates, the envelope is enqueued and ledger-recorded, the Connector dials Varsha's chat profile and posts a notification with action buttons, their Approve tap returns through the Connector as an ingress event, re-enters via the Exchange as an approve envelope on the same thread, and is delivered back to the agent over its A2A profile](diagrams/human-escalation.png)

<sub>Diagram source: [diagrams/human-escalation.mmd](diagrams/human-escalation.mmd)</sub>

Points to notice:

- The human never installs anything. Their endpoint is a **configuration** — a channel and a hosted acceptance policy — not a deployment. This is what keeps the directory dense enough to be worth browsing.
- **Ingress is not a back door.** The reply re-enters through the Exchange's ordinary public API as a call from `varsha@acme`, gets a switchboard-minted envelope, and lands in the ledger like any other. The *no side channels* principle (§2.1) survives contact with Slack.
- The verb set never stretched: a button became `approve`, and a prose reply would have become `respond` carrying the inline `body`.
- Both hops out of a queue went through the same Connector — one to a `chat` profile, one to an `a2a` profile. The switchboard treats a person and a hosted agent as the same kind of thing, which is the claim the whole product rests on.

## 6. Cross-cutting: identity and endpoint model

- **Address = identity.** Flat, org-scoped namespace: `name@org` for humans and team agents, `agent-name@owner.org` for personal agents. Addresses are stable; the endpoint record behind them (transport, description, maintainer) changes freely. Internally the record is keyed by an opaque system-minted `id`, not the address (see [registry-data-model.md](registry-data-model.md)).
- **Humans and agents are co-equal** endpoint kinds with the same protocol surface. The only difference is presentation and delivery profile (§3.7); what each accepts is decided at handshake by the endpoint's own logic — or, for endpoints that supply none, by the console-configured default the Exchange executes on their behalf (§3.3).
- **Channel identity binds to the address.** A human reachable on chat or email has that channel identity (Slack member id, mail address) recorded in their `transport`, so an inbound reply resolves to exactly one switchboard address. The binding is registry-controlled like `maintainer` — self-asserted channel identity would let anyone speak as anyone.
- **Ownership and accountability:** every agent endpoint has a `maintainer` — a registry-controlled **IdP principal** (user or group), not self-declared — and every envelope an agent sends can carry `on_behalf_of` provenance. "Who did this and for whom" is always answerable from ledger + registry alone.
- **Authentication:** each endpoint authenticates to the switchboard with its own credential (per-endpoint token/key issued at registration; mTLS as hardening later). Endpoints never authenticate to each other — the switchboard is the trust broker.
- **Authorization model is deliberately thin:** reachability is coarse **org membership** — any endpoint may dial any other in the same org — plus the handshake right-to-decline. There are no finer switchboard-enforced call graphs or approval gates in v0; cross-org is out of scope (§11), and `org_id` is the only seam for it.

## 7. Cross-cutting: security posture

| Concern | Position |
|---|---|
| Trust boundary | Endpoints are untrusted edges; the switchboard substrate is the trusted core. Everything an endpoint asserts (its self-description, acceptance, envelopes) is recorded as *its* assertion, attributable via its credential. |
| Artifact custody | The switchboard carries **references + digests**, not bodies. Bodies live in the org's existing systems of record (GitHub for diffs, doc stores for documents), which the single-org pilot's endpoints already share access to; for freshly generated artifacts the reference deployment designates **one org-shared object-store bucket** — a deployment convention, not a switchboard component — so agents don't each bring their own storage. Artifact stores enforce their own access control; a leaked ledger never leaks artifact content. Digest lets recipients verify what they fetched is what was offered. |
| Credential custody | Dialing endpoints means holding credentials to reach them (§3.7). Scoped **per-endpoint** and to **one invoke action** — never a general-purpose cloud key — and short-lived where the provider supports it (role-assume with external ID, OAuth grants) in preference to static secrets. Human channels are the exception: `chat`/`email`/`push` use one org-level channel token, because that credential belongs to the org's messaging infrastructure, not to the person. Credentials never enter the ledger; it records that a dial happened, never what authenticated it. This is a **scope** addition to the trusted-core posture above, not a change to it. |
| Tenancy | v0 is single-org. All addresses live in one org namespace; cross-org federation is deferred (§11) and must not be accidentally half-built. |
| Auditability | Append-only ledger, complete by construction (no side channels). Ledger writes are part of the delivery path, not best-effort telemetry. Entries are never edited; an org-configurable retention window may expire old entries per policy. The ledger contains **message bodies** (the conversation) but never **artifact bodies** (the work); role-gated queries and the retention window bound that exposure. |
| Least privilege in the console | Directory is org-visible; queue intervention is owner-only; ledger queries are role-gated (governance roles can query across endpoints). |
| Abuse pressure valve | Unknown/noisy callers are handled by endpoint-side decline logic today; a switchboard-side "quarantine pen" tier is an open question (§11), explicitly not in v0. |

## 8. Cross-cutting: observability

Correlation is built into the contract: `thread_id` (conversation), `session_id` (delivery), `workflow_run` (orchestration) appear on every envelope, so traces need no separate correlation scheme.

- **Metrics (the console's lamps are views over these):** handshake latency and accept/decline/defer rates per endpoint; queue depth, age of oldest item, drain rate; workflow run counts by state; escalation rate per agent; autoscale replica utilization vs. budget cap; **Connector dial latency and outcome mix per profile, plus ingress volume per channel** — a profile whose dials start failing is an alert condition, since the endpoint behind it goes silently unreachable.
- **Ledger vs. telemetry:** the ledger is the durable record of *what happened between parties*; metrics/logs/traces are operational exhaust of *how the substrate performed*. The ledger is never sampled; telemetry can be.
- **Health of the edges:** endpoint status (online/away/saturated) is **observed, not stored on the endpoint** — composed from Queue-service depth plus liveness signals (missed handshakes, heartbeats). Saturation is a first-class observable state, since the human-intervention model depends on noticing it.
- **Human presence needs different signals.** Queue depth is agent-shaped. For human endpoints, status composes from working hours, calendar out-of-office, an open console session, and channel presence — "saturated" for a person means a deep unread inbox nobody has opened in days, and feeds the auto-defer branch of their hosted acceptance policy (§3.3).

## 9. Non-functional requirements (v0)

Anchored to the ratified v0 scale target: **pilot — one team** (~50 endpoints, low thousands of messages/day, single region, single org). All parameters below were ratified in the 2026-07-15 NFR walkthrough (§13, entries 8–9). They size the pilot — not the eventual product.

### 9.1 Scale

| Dimension | v0 target |
|---|---|
| Registered endpoints | 50 (humans + agents), design ceiling 200 without rework |
| Envelope volume | 5,000/day sustained; bursts of 10/s during workflow fan-out |
| Concurrent open sessions | 500 total across the switchboard (not per endpoint) |
| Workflow runs in flight | 100 total across all workflow definitions |
| Regions / orgs | 1 / 1 |

These are sizing targets, not quotas. **Fairness limits (ratified in PR #2 review):** each endpoint may hold at most **25 concurrent sessions as caller** (default; org-configurable per endpoint), and each workflow definition carries a **max-concurrent-runs limit declared in the definition** (default 10). The switchboard `defer`s the excess with standard retry semantics, so a noisy caller or hot workflow degrades only itself. Below those limits, fairness is structural: queues are per-endpoint (a flooded endpoint slows only its own queue) and each queue orders by priority then age.

### 9.2 Latency

Human-in-the-loop turnarounds are minutes-to-days, so the system optimizes for durability over raw speed. Budgets below are the switchboard's own overhead (endpoint think-time excluded), p95:

| Path | Budget |
|---|---|
| Registry resolve (address → transport + reachability; hot path of every offer) | ≤ 100 ms |
| Handshake transit (offer out + reply in, excluding callee decision time) | ≤ 400 ms |
| Enqueue: verb emission → visible in callee's queue (ledger write + SQS send) | ≤ 1 s |
| Connector egress: switchboard-side overhead of one dial, excluding the backend's own processing | ≤ 500 ms |
| Handshake timeout (silence = implicit decline) | 60 s default, per-offer configurable; **human profiles take a longer default or bypass the handshake** (§11 open question #10; [connector-layer.md](connector-layer.md) §10) |
| Console / ledger queries (endpoint- or workflow-scoped) | ≤ 4 s |

### 9.3 Durability

- **No accepted envelope is ever lost.** The ledger row commits in Postgres *before* the SQS send (§3.4); if the ledger write fails, the enqueue fails and the caller/orchestrator retries. If the ledger commit succeeds but the SQS send fails, the entry is re-driven from the ledger (transactional outbox, §3.4) until the send is acknowledged. Delivery is at-least-once; consumers dedupe on envelope ID.
- **Queue items must survive waits.** SQS retention is capped at 14 days; items still undelivered at that horizon (e.g. a paused queue) are automatically re-driven by the queue service rather than expiring silently.
- **Postgres** runs with point-in-time-recovery backups: RPO ≤ 5 minutes, RTO ≤ 4 hours for the pilot.

### 9.4 Long-lived state

- Workflow waits are capped **per state, declared by the workflow author, defaulting to 24 hours**. A state that exceeds its cap times out, fails fast, and is ledger-recorded; re-triggering is explicit. Workflows that legitimately wait longer (leave coverage, multi-day reviews) declare a longer cap up front — the default keeps stuck runs surfacing within a day without forbidding the brief's days-long conversations (§13, entry 9). Waits survive deploys and restarts and cost no compute while idle (Temporal durable timers).
- Threads have no expiry in v0. Ledger retention defaults to **1 year** (org-configurable; the v0 spec sets the floor — see §11).
- Endpoint restarts or redeploys never lose queue position or workflow state.

### 9.5 Availability

- **99.5% monthly** for the pilot (~3.6 h/month error budget); no cross-region DR in v0.
- Degradation order is fixed: under pressure the switchboard may fail new handshakes, but never compromises durability of already-accepted envelopes.

### 9.6 Cost

- Infrastructure spend is capped at **≤ $50/month** at pilot volume. Any design choice that busts the ceiling needs a written reason.
- Consequence: the four services (§3) keep their logical and API boundaries but **co-locate on shared small compute** in the pilot deployment (one small host/cluster running all four plus Temporal, one small Postgres, SQS pay-per-use ≈ $0). The service split is an architectural seam, not four dedicated instances.

## 10. Proposed reference stack (default-but-revisable)

Locked above: the logical boundaries, the envelope contract, the handshake, and the invariants. The stack below was **ratified in the 2026-07-12 review** as the default for the per-component stories and M1 seeding — revisable with a written reason, without reopening the architecture.

| Choice | Default | Rationale |
|---|---|---|
| Deployment shape | **Four services**: Exchange, Queues, and Registry each stand alone; one control-plane service hosts Orchestrator (`/workflows`) and Ledger (`/ledger`) as separate API paths | Each hot-path concern (handshaking, delivery) and the read-heavy Registry scale independently; the lower-traffic Orchestrator and Ledger share one deployable behind distinct paths. |
| System of record | **PostgreSQL** for registry, ledger (append-only tables with retention policy), and workflow metadata | One durable substrate for state; the ledger-write-before-enqueue ordering (§3.4) keeps "complete by construction" honest. |
| Queue mechanics | **AWS SQS**, one queue per endpoint | Managed, pay-per-use (effectively free at v0 volume), no broker to operate. Couples the reference deployment to AWS — accepted for v0; abstraction seam for self-hosters is an open question (§11). |
| Endpoint protocol | HTTPS + JSON. **A2A is the standard we ask direct hosters to implement** — expose an A2A server on any compute and you are a first-class endpoint, dialled exactly like a managed runtime | Bedrock AgentCore, Vertex AI Agent Engine, and Azure AI Foundry already speak A2A, so self-hosters get parity for free; a rival wire protocol against a Linux Foundation standard would lose. A2A standardizes protocol, not reach and auth — that residue is the Connector's (§3.7). The handshake, `defer`, ledger, and verb joins layer *above* A2A, which has none of them. |
| Connector — agent egress | **LiteLLM** | One surface over generic self-hosted A2A (by URL, with agent-card discovery) plus Bedrock AgentCore, Vertex AI Agent Engine, Azure AI Foundry, LangGraph, and Pydantic AI, with task polling, cancellation, and streaming. Accepted gaps: no Claude Managed Agents provider, and its auth model covers keys *to* LiteLLM rather than credentials for each backend — see [connector-layer.md](connector-layer.md) §12. |
| Connector — human channels | **Novu** | Open-source and self-hostable, with multi-channel egress, an in-app inbox, and inbound normalization across chat and email. The ingress half is what no agent-side tooling provides. |
| Orchestrator engine | **Temporal**, self-hosted, backed by the same Postgres | Durable execution for restart-proof waits (per-state caps, 24 h default, §9.4) and fan-out/join; self-hostable so the OSS story holds. |
| Envelope wire format | JSON per §4, schema-versioned (`envelope/v0`) | The envelope is the compatibility surface; version it from day one. |
| Console | React SPA (evolves from the existing mock) over the same APIs endpoints use | The console is a client, not a component; keeps the substrate honest about its API. |

## 11. Open questions and deferred decisions

Explicitly **not** blocking M0 close; owners should land these in the v0 spec or later milestones.

1. **"New call" (ad-hoc dialing) flow design** — UI/UX over the §5.3 path; owned by the Console design story.
2. **Cross-org federation model** — out of v0 entirely; the flat org namespace must not grow ad-hoc cross-org semantics before this is designed.
3. **Richer acceptance tiers** — whether accept/decline/defer ever needs e.g. a quarantine pen for unknown callers. Default position: keep the triple; endpoint-side logic absorbs this need until proven otherwise.
4. **Envelope schema freeze** — field-level schema, error envelopes, and size limits belong to the v0 spec ("envelope schema, handshake protocol, registry API").
5. **Budget-cap semantics for autoscaling agents** — who sets the cap (maintainer vs. org policy) and what happens at the cap besides `defer`.
6. **Queue abstraction seam for self-hosters** — the SQS default couples the reference deployment to AWS (accepted for v0 in the 2026-07-12 review). Define the narrow queue interface so non-AWS adopters can slot in an alternative backend.
7. **Ledger retention defaults** — the retention window is org-configurable (§3.6); the v0 spec should propose a sane default and floor.
8. **Per-channel ingress fidelity** — how faithfully a Slack thread, an email reply chain, and a mobile action each map onto one `thread_id`, and what happens to a reply that arrives on a closed thread. Owned by the Connector story; the verb mapping (§3.7) is settled, the per-channel conventions aren't.
9. **Human presence sources** — which of working hours, calendar OOO, console session, and channel presence the pilot actually wires up (§8), and which are deferred.
10. **Human handshake semantics** — longer timeout vs. bypassing the handshake for human profiles (§9.2). Both work; the v0 spec picks one. Note the hosted default (decision #16) already narrows this: it is evaluated **instantly**, so "longer timeout" can't mean the synchronous Exchange→Connector `dial()` blocks for hours — for hosted-policy humans the default *is* the handshake answer. The live choice is instant hosted evaluation vs. bypass for endpoints that supply their own async policy.

## 12. How this feeds the v0 spec

The v0 spec ("v0 specification frozen", the M0 exit gate) consolidates, from this document: the envelope schema (§4) and verb set semantics (§4.1), the handshake protocol (§5.1) including timeout and defer/retry semantics, the registry API shape (§3.2, §6), the delivery profiles and ingress verb mapping (§3.7), the NFR targets (§9), and the invariants in §2.1/§4. The six per-component M0 stories inherit their boundaries and non-responsibilities from §3 and must flag any contradiction with this document rather than silently diverging.

**Acceptance criteria mapping (from the Asana task):**

- *Reviewed and agreed by stakeholders* — all seven decision areas ratified in the 2026-07-12 walkthrough, plus the PR #1 review amendments and the v0 scale anchor (§13); remaining stakeholders to co-sign.
- *No unresolved cross-component contradictions* — §3 assigns every responsibility to exactly one component and lists explicit non-responsibilities; contradictions found in review go in §11 or get resolved here.
- *Explicitly feeds the v0 spec* — §12.

## 13. Decision log

Ratified in the stakeholder walkthrough on **2026-07-12**, with amendments from the PR #1 review and the v0 scale decision on **2026-07-15** (reviewer: Mohit Gupta). "Confirmed" = the draft's proposal stands; "Changed" = the draft was revised to the outcome below.

| # | Area | Outcome |
|---|---|---|
| 1 | Envelope schema | **Confirmed** — all 11 fields (brief's four + seven proposed extras) stay as the proposal; field-level freeze at v0 spec. |
| 2 | Verb set | **Confirmed** — seven verbs, closed in v0; extension is a spec change. |
| 3 | Delivery topology | **Changed** — the Orchestrator enqueues directly for standing-offer workflow calls; ledger completeness enforced at the queue-write chokepoint (§3.4) instead of "Exchange is sole path". |
| 4 | Artifact custody | **Confirmed** — references + digests only; bodies never stored in the switchboard. |
| 5 | Security | **Confirmed** (a) per-endpoint credentials, switchboard as trust broker; (b) single-org v0. **Changed** (c) ledger append-only *plus* org-configurable retention window, replacing "never deleted, ever". |
| 6 | Reference stack | **Changed** — AWS SQS for queues; Temporal for workflows. Postgres system of record and HTTPS/JSON + signed-webhook protocol unchanged. Deployment shape initially two services (hot path / control plane); **amended 2026-07-15 in PR #1 review** to four services — Exchange, Queues, and Registry standalone, control plane hosting Orchestrator + Ledger as API paths. |
| 7 | Console | **Confirmed** — a client of the public APIs, not a sixth component; no privileged back door. |
| 8 | v0 scale anchor | **Ratified 2026-07-15** — NFRs target pilot scale: one team, ~50 endpoints, low thousands of messages/day, single region/org. |
| 9 | NFR parameters | **Ratified 2026-07-15** — latency budgets relaxed to 100 ms / 400 ms / 1 s / 4 s p95; handshake timeout 60 s; durability RPO ≤ 5 min, RTO ≤ 4 h, auto-re-drive past SQS's 14-day cap; workflow wait caps are **per-state, author-declared, defaulting to 24 h** (initially a hard 24 h cap; softened same day to the configurable default so days-long workflows declare their needs up front — reconciles with the brief's "can wait days"); ledger retention 1 year default; availability 99.5% monthly, no cross-region DR; **cost ceiling $50/month** (four services co-locate on shared compute in the pilot). Amended 2026-07-16 in PR #2 review: fairness via **per-endpoint session limits** (default 25) and **per-workflow-definition run limits** (default 10), replacing the briefly proposed 20%-of-global guardrail. |
| 10 | Envelope schema specifics | **Ratified 2026-07-15** — IDs are switchboard-minted prefixed **UUIDv4s** (ordering via `created_at`); priority **0–9, lower = urgent, default 4, 0 reserved for human take-over**; **16 KB** envelope cap; **strict fields** (unknown fields rejected, no extension bag); **set-once immutability** (corrections are new envelopes); **sha256-only** digests. See [envelope-schema.md](envelope-schema.md). |
| 11 | Message body vs artifact body | **Ratified 2026-07-16 (PR #2 review)** — envelopes gain an optional inline **`body` ≤ 12 KB** for conversational text (including chat-style exchanges), delivered via queues and stored in the ledger as part of the record. Artifact bodies remain by-reference only. Rationale: the conversation is exactly what the ledger must capture — forcing prose into artifact stores (or Slack) would recreate side channels. Sub-artifact references from bodies use **URI fragments** on the canonical artifact ref (`#L40-L60`); per-type conventions land in the v0 spec; interpretation is client-side only. |
| 12 | Registry model & no manifest | **Changed 2026-08-03** — the Registry stores **one endpoint record** (opaque `id`; `address`; `org_id`; `kind`; `display_name`/`description`; an IdP-principal `maintainer`; typed `transport`; lifecycle with reversible suspension) and **no capability manifest**. The switchboard no longer stores or matches accepted verbs/artifact types — acceptance is entirely edge-side at handshake, reinforcing "intelligence at the edges"; `artifact.type` is offer metadata the callee decides on, not a switchboard gate. Status and queue depth are **observed** (composed from the Queue service), not published fields; `maintainer` moves from manifest content to a registry-controlled field (self-declared accountability is worthless). Reachability is coarse **org membership**. Consequence flagged for the **Exchange** story: the handshake no longer performs manifest matching. See [registry-data-model.md](registry-data-model.md). |
| 13 | Connector as a sixth component | **Changed 2026-08-09** — the architecture stored a typed `transport` but assigned **no component the job of executing it**, and assumed endpoints pull. Humans can't pull at all and most agents shouldn't, so a sixth logical component, the **Connector**, owns reaching endpoints in both directions. Two egress callers (Exchange for offers, Queue service for push-profile delivery) put it inside neither, the same reasoning as the queue-write chokepoint. Ingress re-enters through the Exchange's public API, so no privileged inbound path. **No new deployable** — it co-locates, preserving the §9.6 cost ceiling. Component count moves five → six; the Console remains a client, not a component. See [connector-layer.md](connector-layer.md). |
| 14 | A2A as the self-hosting standard | **Ratified 2026-08-09** — the standard asked of direct hosters is **A2A**, adopted rather than invented: expose an A2A server on any compute and you are a first-class endpoint. **Hosting and reach are orthogonal** — where an agent runs never determines how it's reached, and `pull` is the escape hatch for endpoints that can't accept inbound (NAT, air-gapped), *not* the self-hosted profile. Rationale: Bedrock AgentCore, Vertex, and Foundry already speak A2A so self-hosters get parity free; competing with a Linux Foundation standard backed by AWS, Google, and Microsoft would lose; and the switchboard's differentiators (handshake, `defer`, ledger, verb joins) layer above A2A, which has none of them. A2A standardizes protocol but **not reach and auth** — per-vendor work shrinks to endpoint resolution plus a request signer, which is why convergence shrinks the Connector without removing it. |
| 15 | Connector reference stack | **Ratified 2026-08-09** — **LiteLLM** for agent egress, **Novu** for human channels, default-but-revisable per §10. Four gaps accepted, not open: no Claude Managed Agents provider in LiteLLM (one `native` adapter we write, retired when it speaks A2A); LiteLLM's auth covers keys *to* LiteLLM, not per-backend credentials (custody is ours regardless); neither tool has a queue-drain path (`pull` and `webhook` are ours); neither owns `defer` or `envelope_id` dedupe (switchboard semantics stay in the Connector). Recorded so a gap is never later mistaken for a bug. |
| 16 | Hosted default acceptance policy | **Changed 2026-08-09** — amends §2.1 ("handshake over policy DSL") and §3.3 ("never evaluates an endpoint's acceptance policy"). Those two, applied to a human, meant **every offer to every human implicitly declined**: nobody answers a 60 s handshake while asleep, in a meeting, or on leave, and nobody runs a webhook to answer it for them. The endpoint still **owns** its policy; the switchboard **executes** a console-configured default (accept in-org, allowlist, or auto-defer while away) only when the endpoint supplies no webhook. Any endpoint wanting real logic registers its own and nothing changes. Consequence for the **Exchange** story: it picks up hosted-policy evaluation and a human-shaped handshake timeout. |
| 17 | Credential custody | **Changed 2026-08-09** — extends §7, which previously addressed artifact custody only. Dialing endpoints requires credentials: scoped per-endpoint and to one invoke action, short-lived where the provider supports it, never in the ledger; human channels use one org-level channel token since that credential belongs to the org's messaging infrastructure. A **scope** addition to the trusted-core posture, not a change to it — §7 already had the switchboard holding per-endpoint credentials and the full conversation record. |
