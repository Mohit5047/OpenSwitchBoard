# Registry — endpoint data model

**Status:** Draft for review. Design-level: one table and the reasoning behind each column.
**Milestone:** M0 — Discovery, Requirements & System Design
**Scope:** how an endpoint is represented. API shape and console UX come later, consumer-first.

---

## The model

An endpoint is one row. There is no separate manifest, no capability projection, no stored live status.

![Entity-relationship diagram of the single endpoint table: id as the opaque primary key, address as a unique identity, org_id for reachability, kind, display_name and description, a three-field IdP maintainer reference, a typed transport, and lifecycle with suspension and deregistration timestamps](diagrams/registry-erd.png)

<sub>Diagram source: [diagrams/registry-erd.mmd](diagrams/registry-erd.mmd)</sub>

| Column | Type | Meaning |
|---|---|---|
| `id` | text, PK | System-minted, opaque (`ep_<uuid>`). Every internal and API reference uses this. |
| `address` | text, unique | The switchboard identity (`review-agent@varsha.acme`). Unique across **all** rows, including tombstoned — never reused. |
| `org_id` | text | Reachability unit. Same org ⇒ reachable. The only access data. |
| `kind` | text | `human` or `agent`. |
| `display_name` | text | Human-readable label. Endpoint-published. |
| `description` | text | What it is / does. Endpoint-published. |
| `maintainer_type` | text | `user` or `group`. Registry-controlled. |
| `maintainer_ref` | text | IdP subject (OIDC `sub`) or group id — opaque, stable. |
| `maintainer_issuer` | text | The IdP that asserts the maintainer. |
| `transport` | json | `{ provider, locator, config }` — where the endpoint lives and how to reach it. `provider` is the **delivery profile** the Connector dispatches on (`a2a`, `native`, `webhook`, `pull`, `console`, `chat`/`email`/`push`); `locator` is the profile-shaped address (A2A base URL, ARN, webhook URL, Slack member id). |
| `lifecycle` | text | `active` \| `suspended` \| `deregistered`. |
| `registered_at` | timestamptz | |
| `suspended_at` | timestamptz | Null unless suspended. Set on suspend, cleared on resume. |
| `deregistered_at` | timestamptz | Null unless deregistered. Set once, terminal. |

## Why each choice

1. **Surrogate primary key; address is a unique attribute.** References point at the opaque `id`, so knowing an address can't forge a reference to its row. It decouples identity from name and matches the ratified prefixed-UUID convention (`env_`, `thr_`, `ses_`). `address` carries a UNIQUE constraint across all rows including tombstones, so a retired address is never reissued — the ledger references addresses permanently.

2. **Maintainer is an IdP principal, registry-controlled.** `{type, ref, issuer}` points at the org's identity provider — OIDC `sub` for a person, group id for a team — the one identity representation every org already has, stable across email or name changes. The human-readable owner (`varsha@acme.com`) is resolved from the IdP for display, never stored as truth. It is **not** endpoint-published: a self-declared maintainer would make accountability meaningless.

3. **Transport is a typed locator, separate from the address.** The address is the stable identity; the transport is where the thing physically lives and how the switchboard reaches it — provider-shaped and mutable (an A2A base URL, a Bedrock AgentCore ARN, an HTTPS webhook, a Slack id). The DNS name-vs-record split. One JSON field now; it becomes a `binding` child table only if an endpoint needs several.

   `provider` names one of the Connector's delivery profiles ([connector-layer.md](connector-layer.md) §3), so the row is what makes an endpoint reachable rather than merely nameable. Two consequences worth stating here: **hosting and reach are orthogonal** — a self-hosted A2A agent and a Bedrock AgentCore agent both carry `provider: a2a`, and this column cannot tell you where either runs — and `pull` marks endpoints that can't accept inbound at all (NAT, air-gapped), never "self-hosted".

4. **No manifest.** The switchboard doesn't enforce which verbs or file types an endpoint accepts — that's the endpoint's call at handshake (intelligence at the edges). A human chooses an endpoint by `description`; a machine reaches it by `transport`. Nothing was left for a manifest to hold. Trade-off: no structured "list every endpoint that takes diffs" — acceptable at pilot scale (search plus decline-at-handshake), and it returns as a real capability feature if agent-to-agent auto-discovery ever needs it.

5. **Reachability is org membership.** Caller and callee in the same org can reach each other; the endpoint still declines fine-grained at handshake. `org_id` is the whole access model. One org in v0; cross-org federation is the north-star, and `org_id` is the only seam left for it — nothing federated is built.

6. **Suspension is reversible and distinct from deregistration.** `suspended_at` marks a reversible pause (queue retained, can return to active); `deregistered_at` marks the terminal tombstone. Opposite meanings, so two columns, not one.

7. **Live status is not stored.** Online/away/saturated and queue depth are the Queue service's facts (queues are switchboard-owned; the Registry never calls endpoints). The console composes them at read time.

## Reconciled with the architecture

This model changed two things the earlier ratified `architecture.md` asserted; the architecture doc has been updated to match (decision-log entry 12, 2026-08-03):

- The manifest is gone — the switchboard no longer stores or matches accepted verbs/artifact types; acceptance is edge-side at handshake, and `artifact.type` is metadata the callee decides on.
- Maintainer moved from manifest content to a registry-controlled field.

One consequence lands on a sibling story: the **Exchange** handshake no longer performs manifest matching — flagged in the architecture decision log for the Exchange design to pick up.

A later change (decision-log entries 13–17, 2026-08-09) gave this table a consumer it previously lacked: the **Connector** ([connector-layer.md](connector-layer.md)) is the component that executes `transport`, dispatching on `provider`. Nothing in the model changes beyond naming the profile set — the row was already the right shape, it just had nobody to read it. Credentials still live outside this table (below); the Connector's custody rules govern them.

## Deliberately not here

- **Credentials / auth material** — a separate concern, designed with the handshake, not the identity row. The custody rules for the credentials the switchboard needs in order to *dial* an endpoint are set in [connector-layer.md](connector-layer.md) §8; neither the secrets nor their references belong in this table.
- **Teams as a table** — group membership belongs to the IdP; `maintainer` references it.
- **Cross-org federation** — north-star only; `org_id` is the seam and nothing more.
- **Console UX** — later, and consumer-first. (API shape is settled: [registry-api.md](registry-api.md).)
