# OpenSwitchBoard

The open communication exchange for organizations where AI agents are first-class workflow participants alongside humans. Switchboard handles addressing, routing, queuing, orchestration, and record-keeping for human↔agent and agent↔agent interactions — a neutral common carrier that connects parties, manages congestion, and keeps records, but never injects its own logic into the conversation.

**Driving use case:** a coding agent produces a PR; the switchboard routes it to two senior engineers' review agents; those agents may escalate to their humans; feedback flows back to the coding agent, which revises. Human coordination time is spent only where human judgment is needed.

## Architecture

Five components, one substrate — see the [architecture document](docs/architecture.md):

1. **Registry** — flat namespace of human and agent endpoints, each publishing a capability manifest.
2. **Exchange** — session establishment and delivery via an offer/accept handshake.
3. **Queues** — durable, per-endpoint, switchboard-owned; agents pull, humans see an inbox.
4. **Orchestrator** — declarative workflows over endpoints; states are conversations and can wait days.
5. **Ledger** — append-only record of every session, offer, and envelope. No side channels.

Every payload carries an envelope: an artifact reference, a thread ID, provenance, and one verb from a small universal set — `request`, `respond`, `revise`, `approve`, `reject`, `escalate`, `inform`. Intelligence lives at the edges; the switchboard stays lean and neutral.

## Status

Pre-build (milestone M0 — discovery, requirements, and system design). The overall architecture is documented and ratified; the v0 specification is next. An interactive console mock (Directory, Queues, Workflows, Ledger) lives in `mock_ui/`.

## License

See [LICENSE](LICENSE).
