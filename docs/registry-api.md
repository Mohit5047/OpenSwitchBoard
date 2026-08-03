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

## Surface

| Operation | Shape | Consumer | Use case |
|---|---|---|---|
| Register | `POST /endpoints` | owner (console) / provisioning | claim an address, mint credential, create the record |
| Resolve | `GET /endpoints/{address}` | Exchange (hot path), agents | before dialing: transport + org + display |
| List / search | `GET /endpoints?q=&kind=&maintainer=&lifecycle=` | console, agents (MCP) | look up a known endpoint, or discover by purpose |
| Update | `PATCH /endpoints/{address}` | owner / agent | change `display_name`, `description`, `transport` (redeploy → new webhook/ARN) |
| Suspend / resume | `POST /endpoints/{address}:suspend` · `:resume` | owner / admin | reversible pause |
| Deregister | `DELETE /endpoints/{address}` | owner / admin | terminal tombstone (address stays reserved) |
| Transfer maintainer | `POST /endpoints/{address}:transfer` | admin / owner | change accountability, address unchanged |
| Rotate credential | `POST /endpoints/{address}/credential:rotate` | owner | auth hygiene (credential is a separate concern) |

This matches the interfaces architecture §3.2 already names — register/update, resolve, query directory — with nothing added that contradicts it.

## Search — the substance of the list API

`GET /endpoints?q=…` runs **Postgres full-text search** over `display_name` + `description`: tokenized, stemmed, ranked, so "payments review" matches "reviews payment diffs." Facets (`kind`, `maintainer`, `lifecycle`) are plain filters. **One endpoint serves both modes** — lookup (a known name/address falls out of the same query) and discovery (purpose-based). It's built into the Postgres we already run: no external engine, no added cost, and at 50–200 rows match *quality*, not speed, is the whole game — which is exactly what full-text gives over dumb substring match, without the infrastructure of semantic/vector search.

**One trade to keep in view:** with the manifest gone, agent discovery is *also* text-over-`description` — an agent can no longer query "endpoints that accept `diff`" structurally. So `description` is dual-purpose: human-readable **and** machine-discoverable. Acceptable at pilot scale (agents usually know their target, or the org is small enough to scan), but it means the description field is the discovery surface for both audiences, not just a label.

## Lifecycle as action sub-routes

`:suspend`, `:resume`, and `:transfer` are POST actions, not a `PATCH {lifecycle: …}`. A state machine reads as verbs, and each transition is its own auditable operation with its own preconditions — you can't resume what isn't suspended, and a transfer needs the receiving maintainer's consent. `DELETE` tombstones (the address is never reused).

## MCP — the agent façade

Agents reach the system over **MCP tools that wrap the same core operations** — `resolve_endpoint`, `search_directory`, `register_endpoint`, `update_endpoint`. One implementation, two façades: REST for the console and service-to-service, MCP for agents. Neither door has behavior the other lacks.

## Authorization (sketch, not the spec)

- **Reads** — any authenticated principal in the same org (reachability is org membership).
- **Register** — owner or provisioning.
- **Mutations and lifecycle** — the endpoint's maintainer, or an org admin.

The full authorization model is its own concern; this is the shape it takes against these operations, not its definition.

## Deferred

- OpenAPI spec and generated code — this is M0 design, not implementation.
- Pagination mechanics (trivial at pilot scale; offset vs cursor is a build call).
- Rate limiting, credential mechanics, and the full authz model.
- The concrete MCP tool schemas — the wrapper principle is settled; exact definitions land with the dual-consumer (console + MCP) phase.
