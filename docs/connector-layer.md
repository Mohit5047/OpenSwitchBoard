# Connector — reaching endpoints

**Status:** Draft for review. Design-level: the component that turns a `transport` record into an executed call, in both directions.
**Milestone:** M0 — Discovery, Requirements & System Design
**Builds on:** [architecture.md](architecture.md) §3.3 (Exchange), §3.4 (Queues), §7 (security posture), §10 (reference stack), and [registry-data-model.md](registry-data-model.md) (the `transport` column).

---

## 1. Purpose and scope

The Registry stores a typed `transport` — where an endpoint lives and how to reach it. Nothing in the architecture said **who executes it**. Agents were described as pulling from their queue (§3.4), which quietly assumed every endpoint can run a poller. Most can't:

- **Humans** on Slack, email, or mobile can't pull at all. A person will never run a poller, and requiring one turns onboarding into a deployment instead of a configuration.
- **Most agents shouldn't.** Anything that can accept inbound is better dialed, whether it runs on a managed runtime or on the org's own compute.

Reaching an endpoint is also **bidirectional**. When a human answers an escalation in Slack, that reply has to re-enter the switchboard as an envelope carrying a verb on the same `thread_id`. Nothing in the current docs covered ingress at all.

The Connector owns both directions. **In scope:** executing a transport, adapting per-profile mechanics, custody of the credentials that execution needs, and mapping channel events back into envelopes. **Out of scope:** choosing *whom* to call. Callers pick their callee from the directory (§2.1, "no personal routing layer"); the Connector only reaches an already-chosen endpoint.

The distinction the component rests on: the Connector carries **transport competence, not policy competence**. Knowing how to reach every party is what a switchboard operator does; deciding what happens in the conversation is not. "Intelligence at the edges" is untouched.

## 2. Position — sixth component, two callers

The Connector is a sixth logical component with **two egress callers**:

| Caller | What it dials for |
|---|---|
| **Exchange** (§3.3) | Offers during the handshake — first-contact and ad-hoc calls |
| **Queue service** (§3.4) | Envelope delivery for push-profile endpoints, draining the queue on the endpoint's behalf |

It is not inside the Exchange for the same reason the ledger-recorded queue-write API isn't: two independent writers reach it, and decision #3 already established that the Orchestrator bypasses the Exchange entirely for standing-offer calls.

**Ingress needs no new path.** A Slack reply or a runtime callback becomes an ordinary call *from* that endpoint, entering through the Exchange's existing public API exactly as if the endpoint had dialled by itself. There is no privileged back door — the same rule the Console follows (§3.1: a client of the public APIs, not a component).

**No new deployable in v0.** The Connector is a logical boundary that co-locates with the Exchange and Queue services on shared compute, per the §9.6 cost ceiling. The service split stays an architectural seam.

## 3. Delivery profiles

`transport.provider` becomes a discriminated union of **delivery profiles**. The critical framing:

> **Hosting and reach are orthogonal.** Where an agent runs — managed runtime, own server, serverless function, a laptop — does not determine how it is reached. The only question that matters is whether it can accept inbound.

| Profile | Kind | Direction | Credential held |
|---|---|---|---|
| `a2a` | agent | push | scoped, per-endpoint |
| `native` | agent | push | scoped, per-endpoint |
| `webhook` | agent | push | signing key |
| `pull` | agent | pull | endpoint's own token |
| `console` | human | pull (browser) | none |
| `chat` / `email` / `push` | human | push + ingress | org-level channel token |

- **`a2a` is the default for any dialable agent, self-hosted or managed.** One profile covers a Bedrock AgentCore runtime, a Vertex agent, a Foundry agent, and a container on the org's own Kubernetes. The switchboard neither knows nor cares which.
- **`native`** is the shrinking set of managed runtimes that don't speak A2A yet — Claude Managed Agents today. One bespoke adapter each, retired as they converge.
- **`webhook`** is the low-ceremony option for simple agents not worth running an A2A server for. Signed POST, no task lifecycle.
- **`pull` is not the self-hosting profile.** It is the escape hatch for endpoints that cannot accept inbound at all: personal compute behind NAT, air-gapped deployments. Self-hosting does not imply pull, and no other document should suggest it does.

Every profile carries the same envelope, the same verbs, and the same ledger treatment. Only reach mechanics and presentation differ.

## 4. Egress contract

One shape across every profile:

```
dial(transport, payload) -> accepted | deferred(retry_after) | failed(reason)
```

`transport` is the endpoint's **already-resolved** transport record — the caller (Exchange for offers, Queue service for deliveries) does the single Registry `resolve` and hands the record in; the Connector never calls the Registry itself (one resolve per offer, §9.2). `payload` is either offer metadata (from the Exchange) or an envelope (from the Queue service). The Connector reads the delivery profile from the transport record, applies the profile's auth, executes, and normalizes the result into those three outcomes.

**Guarantees are profile-dependent, and the Connector reports which apply** rather than pretending uniformity:

| Capability | `a2a` | `native` | `webhook` | `pull` |
|---|---|---|---|---|
| Task polling | yes | varies | no | n/a |
| Cancellation | yes | varies | no | n/a |
| Streaming | yes | varies | no | n/a |

This capability data is **operational, about the host**. It is not a capability manifest and does not reopen decision #12: what verbs and artifact types an endpoint accepts stays edge-side, decided at handshake, and the switchboard still never matches on it.

## 5. Ingress contract

![Escalation round trip: the review agent escalates, the envelope is enqueued and ledger-recorded, the Connector dials Varsha's chat profile and posts a notification with action buttons, their Approve tap returns through the Connector as an ingress event, re-enters via the Exchange as an approve envelope on the same thread, and is delivered back to the agent over its A2A profile](diagrams/human-escalation.png)

<sub>Diagram source: [diagrams/human-escalation.mmd](diagrams/human-escalation.mmd)</sub>

Channel and runtime events map back into envelopes:

| Event | Verb | Carries |
|---|---|---|
| Structured action (Slack button, mobile action) | `approve` / `reject` | — |
| Free-text reply (Slack message, email body) | `respond` | the text in `body` |
| Runtime completion callback | `respond` | result reference in `artifact` |

The inline `body` (≤ 12 KB, decision #11) is what makes this work without an artifact store. A human replying in prose from Slack produces a conforming envelope directly. That decision was ratified for conversational text generally; the human-channel path is what makes it load-bearing.

Ingress envelopes are switchboard-minted like any other — the endpoint never supplies `envelope_id`, `session_id`, or `created_at`. Thread continuity comes from the notification's correlation, and `provenance.workflow_run` carries the originating run or `ad-hoc`.

## 6. Backpressure maps to `defer`

A backend that signals saturation — HTTP 429, a quota error, a runtime concurrency cap — becomes the endpoint's `defer` with the backend's retry-after. No new retry model: this reuses the Exchange's existing defer contract (§3.3), so the ledger shows the same defer trail whether the endpoint declared it or its host did.

Failures that are not backpressure surface as `failed` and do not silently retry beyond the profile's transport-level retries.

## 7. Idempotency

**Egress.** Both egress callers can re-drive: the Queue service's sweeper re-sends unmarked outbox rows (§3.4), and the Exchange retries deferred offers. Delivery is at-least-once, so the Connector passes `envelope_id` as the idempotency key wherever the profile supports one, and endpoints dedupe on it — the same contract §3.4 already sets for queue consumers. Nothing here weakens the ledger-write-before-send ordering that makes the record complete by construction.

**Ingress is different — there is no `envelope_id` yet.** An inbound channel event mints a *new* envelope (§5), so it can't dedupe on one. Channel platforms re-deliver: Slack retries an unacknowledged interaction or event, and a naïve mint would turn one Approve tap into two ledger records and two envelopes on the thread. The Connector therefore dedupes on the **channel-native event id** (Slack `event_id` / interaction token, the provider's message id) **before minting** — one channel event maps to at most one minted envelope, and a retry resolves to the already-minted one rather than a second. This runs ahead of the ledger write, so ledger-write-before-send still holds and the record stays complete by construction.

## 8. Credential custody

Dialing requires credentials the switchboard did not previously hold. The rules:

1. **Per-endpoint, not per-org** for agent profiles. One endpoint's credential never reaches another's transport.
2. **Scoped to one action** — invoke this runtime. Not a general-purpose cloud key.
3. **Short-lived where the provider supports it** — role-assume with an external ID, OAuth grants — in preference to static secrets.
4. **Human channels are the exception**: `chat`/`email`/`push` use one org-level channel token, because the credential belongs to the org's Slack or mail infrastructure, not to the person.
5. **Never in the ledger.** Credentials are operational secrets; the ledger records that a dial happened, never what authenticated it.

This is a **scope** addition to §7, not a posture change. §7 already establishes the switchboard substrate as the trusted core holding per-endpoint credentials and the full conversation record. Holding a narrowly scoped invoke credential is smaller than what it already holds.

## 9. Hosted default acceptance policy

**The problem.** §3.3 sets a 60 s handshake timeout where silence is an implicit decline, and §2.1 says the switchboard never evaluates an endpoint's acceptance policy. Apply both to a human and **every offer to every human implicitly declines**. Someone in a meeting, asleep, or on leave cannot answer a handshake — and cannot run a webhook to answer it for them.

**The fix.** The endpoint still *owns* its acceptance policy; the switchboard *executes* a console-configured default when the endpoint supplies no webhook of its own:

- accept from anyone in the org (default for humans),
- accept from an allowlist,
- auto-defer with retry-after while away.

Any endpoint that wants real logic registers its own webhook and nothing changes for it. Agents that don't want to implement handshake logic may use the same hosted defaults.

**This is an amendment to §2.1 and §3.3, not a reading of them** — logged as decision #16. The principle survives if scoped precisely: the endpoint owns the policy, the switchboard may execute it when none is supplied. Out-of-office is the case that makes it undeniable.

## 10. Human handshake and presence

- **Timeout.** The 60 s default is agent-shaped. Human profiles take a longer default, or bypass the handshake entirely on the grounds that an inbox is unbounded and a human declines by acting rather than by handshaking. Settled in the v0 spec — but note it can't mean the Exchange→Connector `dial()` blocks for hours: a **hosted default policy (§9) is evaluated instantly**, so for hosted-policy humans the default *is* the handshake answer, returned at once. The genuinely open branch (architecture §11 #10) is therefore instant hosted evaluation vs. bypassing the handshake for endpoints that supply their own async policy — not a multi-hour synchronous block.
- **Presence.** §8 composes status from queue depth and liveness, which is agent-shaped. Human presence composes from working hours, calendar out-of-office, an open console session, and channel presence. "Saturated" for a person means 37 unread and no inbox opened in two days — and should feed the hosted auto-defer above.

## 11. The self-hosting standard is A2A

The open question this component grew out of was: *what do we ask direct hosters to implement?* The answer is an existing standard, not one OpenSwitchBoard invents.

**Expose an A2A server on whatever compute you like and you are a first-class endpoint.** Server, serverless, on-prem, a homelab with an inbound route — the switchboard cannot tell them apart, and a self-hosted agent gets exactly the treatment a hyperscaler-hosted one does.

Why adopt rather than invent:

1. **It costs nothing.** Bedrock AgentCore, Vertex AI Agent Engine, and Azure AI Foundry already speak A2A natively, so self-hosters get parity for free rather than joining a smaller ecosystem.
2. **Competing would lose.** A2A is a Linux Foundation project with AWS, Google, and Microsoft behind it. A rival wire protocol from this project would not win.
3. **The differentiator sits above it.** A2A standardizes protocol — message shape, task lifecycle, cancellation, discovery. It has no `defer`, no handshake with a right to decline, no org-wide append-only ledger, and no verb-joined workflows. Those are the switchboard, and they layer cleanly on top.

**What A2A does not standardize: reach and auth.** An AgentCore-hosted A2A server is still an AWS endpoint behind AWS authentication; a Vertex one is still behind Google's. Per-vendor work drops from a full adapter — bespoke invoke semantics, its own polling model, its own error taxonomy — to endpoint resolution plus a request signer. That residue is exactly what §8 owns, and it is why A2A convergence shrinks this component without eliminating it.

## 12. Reference implementations and their gaps

Following the §10 default-but-revisable pattern:

| Half | Default | Why |
|---|---|---|
| Agent egress | **LiteLLM** | Generic self-hosted A2A by URL with agent-card discovery, plus Bedrock AgentCore, Vertex AI Agent Engine, Azure AI Foundry, LangGraph, and Pydantic AI behind one surface, with task polling, cancellation, and streaming. |
| Human channels | **Novu** | Open-source, self-hostable, multi-channel egress with an in-app inbox, and inbound normalization across chat and email — the ingress half nothing in the agent tooling provides. |

**Accepted gaps.** These are known and accepted, not open questions — recorded so nobody later mistakes a gap for a bug:

| Gap | Consequence |
|---|---|
| Claude Managed Agents is absent from LiteLLM's providers | one `native` adapter we write, retired when it speaks A2A |
| LiteLLM's auth model covers keys *to* LiteLLM, not credentials for each backend | credential custody (§8) is ours regardless |
| Neither tool has a queue-drain path | the `pull` and `webhook` profiles are ours |
| Neither owns `defer` semantics or `envelope_id` dedupe | switchboard semantics stay in the Connector (§6, §7) |

The dependency boundary is the point: LiteLLM is the agent-egress engine *inside* the Connector, Novu the human-channel engine. Neither is the Connector.

## 13. Does not

- **Choose recipients.** No routing, no capability matching, no preference resolution. The caller picked the callee from the directory.
- **Resolve addresses.** Its caller passes the already-resolved transport record (§4); the Connector never calls the Registry. Resolve stays a caller concern, one per offer (§9.2).
- **Evaluate artifact content.** It carries references and digests like everything else.
- **Own the record.** It emits; the Exchange records handshakes and the queue-write chokepoint records deliveries (§3.4). Ledger completeness still lives there.
- **Bypass the queue.** Push-profile delivery drains a queue; it does not replace one.
- **Store endpoint state.** Presence is composed at read time, consistent with §3.2 and §8.

## 14. Reconciled with the architecture

This component changes three things the ratified `architecture.md` asserted; that document has been updated to match (decision-log entries 13–17, 2026-08-09):

- **Five components became six.** The Connector is a logical component with two callers, not a sub-responsibility of the Exchange.
- **The switchboard holds invoke credentials** (§8) — a scope addition to §7's custody model, which previously addressed artifacts only.
- **The switchboard may execute a hosted acceptance policy** (§9) — an amendment to §2.1's "handshake over policy DSL" and §3.3's "never evaluates an endpoint's acceptance policy," narrowly scoped to the case where the endpoint supplies none.

One consequence lands on a sibling story: the **Exchange** design picks up hosted-policy evaluation and a human-shaped handshake timeout.

## 15. Deliberately not here

- **Concrete channel adapters** — per-channel ingress fidelity (threading in Slack vs. email reply parsing) is build detail for the v0 spec.
- **Credential rotation mechanics** — the rotation design lands with the credential/auth story the Registry docs already defer; §8 sets the custody rules, not the lifecycle.
- **Console UX for endpoint onboarding** — the "add endpoint" flow that makes this a configuration rather than a deployment belongs to the Console story.
- **Cross-org reach** — federation is deferred entirely (§11); profiles carry no cross-org semantics.
