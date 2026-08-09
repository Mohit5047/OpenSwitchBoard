# Registry — API

**Status:** Draft for review. Design-level: the operations, their shape, and the reasoning. No OpenAPI, no code.
**Milestone:** M0 — Discovery, Requirements & System Design
**Builds on:** [registry-data-model.md](registry-data-model.md) (the one endpoint table) and architecture §3.2 (Registry interfaces), §9.2 (resolve ≤ 100 ms hot path), §10 (HTTPS + JSON default).

---

## Style: REST + JSON

The surface is one resource — endpoints — and CRUD-shaped, which is REST's home. The GraphQL alternative had exactly one real claim here: federating a Directory row across Registry + Queue + Orchestrator in a single query. That need **dissolved** once the v0 directory row was scoped to Registry-only — identity + description + maintainer is all that choosing an endpoint requires; live status and "referenced-by N workflows" were console-mock features, not requirements. With no cross-service composition left:

- the hot-path **resolve is a cacheable `GET`** (GraphQL's POST-everything throws HTTP caching away);
- the surface is **one resource**, not a graph worth a query language;
- **MCP maps to RPC-shaped tools**, not GraphQL — agents reach the system as tool calls;
- it matches the **ratified HTTPS + JSON** default (§10), so there's no departure to justify.

## Addressing

Public paths key on **`address`** — the directory is public and the address *is* the identity. The opaque `id` stays internal (FKs, cross-service references); that's where the surrogate-PK benefit lives (no forgeable internal handles, no coupling to the name). A public directory keyed by address doesn't undermine that, because the address is public by design.

The primary key stays the opaque `id` — pathing on `address` doesn't change that. `address` carries a **UNIQUE constraint across all rows** (registry-data-model.md), so it backs a unique index and `GET /endpoints/{address}` is a single-key lookup the database serves directly — no PK change required. And `org_id` is **not** part of the path: the address is globally unique, so resolution never needs an org qualifier. `org_id` enters as an **authorization filter** (reads are same-org — see *Authorization* below), not as routing. There is **one shared `endpoints` table**, not a table per org; `org_id` is a column, and one org in v0.

## Surface

| Operation | Shape | Consumer | Use case |
|---|---|---|---|
| Register | `POST /endpoints` | owner (console) / provisioning | claim an address, mint credential, create the record |
| Resolve | `GET /endpoints/{address}` | Exchange and Connector (hot path), agents | before dialing: transport (including the delivery profile the Connector dispatches on) + org + display |
| List | `GET /endpoints?kind=&maintainer=&lifecycle=` | console, agents (MCP) | filter the directory by structured facets, org-scoped |
| Update | `PATCH /endpoints/{address}` | owner / agent | change `display_name`, `description`, `transport` (redeploy → new webhook/ARN) |
| Suspend / resume | `POST /endpoints/{address}:suspend` · `:resume` | owner / admin | reversible pause |
| Deregister | `DELETE /endpoints/{address}` | owner / admin | terminal tombstone (address stays reserved) |

This matches the interfaces architecture §3.2 already names — register/update, resolve, query directory — with nothing added that contradicts it.

## List — structured filters only

The list endpoint filters, it does not free-text search. The query surface is a **fixed allowlist of structured facets** — `kind`, `maintainer`, `lifecycle` — and every list is implicitly **org-scoped** (you only ever see your org's directory). Each facet is an indexed column, so the query is a plain, cheap, bounded index scan; results are capped (pagination is a build detail, deferred below). There is no open-ended `q=` parameter, no cross-org listing, and no arbitrary-field matching — the only things you can filter on are the facets named here.

**Trade recorded — no free-text purpose discovery in v0.** Dropping `q=` means you can't yet ask "find me an endpoint that reviews payment diffs" in words; you filter by facet, or you already know the `address`. `description` stays a **human-read label**, not a machine query surface. This is acceptable at pilot scale — an agent usually knows its target, and an org's directory is small enough to list and scan. Purpose-based discovery returns as a real capability feature (structured, or search-backed) if agent-to-agent auto-discovery ever needs it — the same deferral the data model records for the dropped manifest.

## Lifecycle as action sub-routes

`:suspend` and `:resume` are POST actions, not a `PATCH {lifecycle: …}`. A state machine reads as verbs, and each transition is its own auditable operation with its own preconditions — you can't resume what isn't suspended. `DELETE` tombstones (the address is never reused).

## MCP — the agent façade

Agents reach the system over **MCP tools that wrap the same core operations** — `resolve_endpoint`, `list_endpoints`, `register_endpoint`, `update_endpoint`. One implementation, two façades: REST for the console and service-to-service, MCP for agents. Neither door has behavior the other lacks.

## Authorization (sketch, not the spec)

- **Reads** — any authenticated principal in the same org (reachability is org membership).
- **Register** — owner or provisioning.
- **Mutations and lifecycle** — the endpoint's maintainer, or an org admin.

The full authorization model is its own concern; this is the shape it takes against these operations, not its definition.

## Deferred

- OpenAPI spec and generated code — this is M0 design, not implementation.
- Pagination mechanics (trivial at pilot scale; offset vs cursor is a build call).
- Rate limiting and the full authz model.
- **Credential mechanics, including rotation.** Credentials are a separate concern designed with the handshake, not the identity row (registry-data-model.md, "Deliberately not here") — so there's no `credential:rotate` on the Registry surface; rotation lands with that auth design. Note this now covers two distinct sets: the credential an endpoint uses to authenticate *to* the switchboard, and the scoped credential the switchboard holds to *dial* the endpoint ([connector-layer.md](connector-layer.md) §8). Neither is a Registry resource.
- **Free-text / purpose-based discovery.** The list endpoint is facet-only (above); a words-in, endpoints-out discovery surface is a later capability feature if agent auto-discovery needs it.
- The concrete MCP tool schemas — the wrapper principle is settled; exact definitions land with the dual-consumer (console + MCP) phase.
