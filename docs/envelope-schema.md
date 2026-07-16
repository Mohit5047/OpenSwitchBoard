# Envelope schema (draft, `envelope/v0`)

**Status:** Ratified draft — the low-level design companion to [architecture.md §4](architecture.md). This is the "draft envelope schema that feeds the v0 spec" deliverable; the v0 spec gives it the final field-level freeze.
**Ratified basis:** the 11-field contract per architecture decision log #1 (2026-07-12), and the six schema specifics — prefixed UUIDs, priority 0–9 with 0 reserved, 16 KB cap, strict fields, set-once immutability, sha256-only — per decision log #10 (2026-07-15).

## Wire format

JSON, schema-versioned. Every payload that moves through the switchboard is exactly one envelope.

```jsonc
{
  "schema": "envelope/v0",
  "envelope_id": "env_5f0c2b1a-9d3e-4c7b-8a21-6f4e0d9c3b57",   // unique, switchboard-assigned (UUIDv4)
  "thread_id": "thr_a2e8c4d0-1b6f-4e93-b7d5-08c1f2a64e39",     // conversation identity; stable across revise loops
  "session_id": "ses_7c31e9f5-2a84-4d06-9e1b-53b8a0c7d214",    // the handshaken session this travels on
  "verb": "request",                                  // exactly one of the closed set
  "body": "Focus on the rollback path — retry semantics in payments-core changed.",  // optional message text (≤ 12 KB)
  "from": "coding-agent@you.acme",
  "to": "review-agent@varsha.acme",
  "artifact": {
    "type": "diff",                                   // matched against callee manifest at handshake
    "ref": "artifact://acme/pr-4821",                 // reference — bodies live outside the switchboard
    "digest": "sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08"
  },
  "provenance": {
    "workflow_run": "wf:high-impact-pr-review#r-2210", // or "ad-hoc"
    "on_behalf_of": "varsha@acme",                     // the human/team an agent endpoint acts for
    "in_reply_to": "env_0d94b7e2-6c15-4f38-a5d0-b82c1e9f7a46"  // prior envelope on the thread, if any
  },
  "priority": 1,
  "created_at": "2026-07-15T14:02:12Z"
}
```

## JSON Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://openswitchboard.dev/schemas/envelope-v0.json",
  "title": "Switchboard envelope v0",
  "type": "object",
  "required": ["schema", "envelope_id", "thread_id", "session_id", "verb", "from", "to", "artifact", "provenance", "created_at"],
  "additionalProperties": false,
  "properties": {
    "schema": { "const": "envelope/v0" },
    "envelope_id": { "type": "string", "pattern": "^env_[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$" },
    "thread_id": { "type": "string", "pattern": "^thr_[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$" },
    "session_id": { "type": "string", "pattern": "^ses_[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$" },
    "verb": { "enum": ["request", "respond", "revise", "approve", "reject", "escalate", "inform"] },
    "body": { "type": "string", "maxLength": 12288 },
    "from": { "$ref": "#/$defs/address" },
    "to": { "$ref": "#/$defs/address" },
    "artifact": {
      "type": "object",
      "required": ["type", "ref", "digest"],
      "additionalProperties": false,
      "properties": {
        "type": { "type": "string", "maxLength": 64 },
        "ref": { "type": "string", "format": "uri", "maxLength": 2048 },
        "digest": { "type": "string", "pattern": "^sha256:[0-9a-f]{64}$" }
      }
    },
    "provenance": {
      "type": "object",
      "required": ["workflow_run"],
      "additionalProperties": false,
      "properties": {
        "workflow_run": { "type": "string", "maxLength": 256 },
        "on_behalf_of": { "$ref": "#/$defs/address" },
        "in_reply_to": { "type": "string", "pattern": "^env_[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$" }
      }
    },
    "priority": { "type": "integer", "minimum": 0, "maximum": 9, "default": 4 },
    "created_at": { "type": "string", "format": "date-time", "pattern": "^\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}(\\.\\d+)?Z$" }
  },
  "$defs": {
    "address": {
      "type": "string",
      "pattern": "^[a-z0-9][a-z0-9-]*@[a-z0-9][a-z0-9-]*(\\.[a-z0-9][a-z0-9-]*)*$",
      "maxLength": 254
    }
  }
}
```

## Semantics (normative notes)

- **Closed verb set.** The seven-verb enum is closed in v0 (decision log #2); workflow joins key off verb emissions.
- **IDs** are switchboard-assigned UUIDv4s with type prefixes (`env_`, `thr_`, `ses_`). Endpoints never mint them. Chronological ordering comes from `created_at`, not the ID.
- **`body`** (optional) — conversational text accompanying the verb, ≤ 12 KB: review remarks, escalation reasons ("needs human judgment on the rollback path"), rejection rationale, or plain chat turns. This is the **message about the work**; the work itself always stays by-reference in `artifact`. Bodies are delivered through the queue and stored in the ledger as part of the interaction record — conversation is switchboard custody, work products are not (ratified 2026-07-16, PR #2 review). Chat-style exchanges are simply sequences of small-bodied envelopes on one `thread_id` — the per-message cap never binds there; message *volume* is governed by the scale targets (architecture §9.1). Anything that doesn't fit in 12 KB is a work product and belongs in an artifact. The body may freely *mention* artifact URIs (or anything else) as plain text — clients like the Console may linkify them — but the switchboard never parses body content (common carrier), so inline mentions carry **no protocol semantics**: no manifest matching, no digest verification, no ledger indexing. The structured `artifact` field is the one canonical, verified reference per envelope. To point at a specific part of that artifact, the body uses **URI fragments** on the canonical ref — `artifact://acme/pr-4821#L40-L60` for diff lines, `#section-name` for docs (ratified 2026-07-16). A fragment doesn't change which resource is referenced, so the artifact digest still covers exactly what the fragment points into; fragment conventions are defined per artifact type in the v0 spec, and interpretation is entirely client/endpoint-side.
- **`artifact`** — the work item this envelope is about. `type` (e.g. `diff`, `design-doc`) is matched against the callee's capability manifest at handshake, so an endpoint is never offered work it can't handle. `ref` is a URI pointing to where the body actually lives — the org's existing systems of record (GitHub for diffs, doc stores for documents), or the org-shared artifact bucket the reference deployment designates for freshly generated content; the switchboard never stores or fetches it, and access control stays with the store. `digest` is the sha256 fingerprint of that content: the recipient hashes what it fetches from `ref` and compares, proving nobody swapped the artifact after the offer.
- **`provenance`** — where the envelope came from. `workflow_run` names the orchestrator run that produced it (`wf:<workflow>#<run>`), or the literal `ad-hoc` for direct dials. `on_behalf_of` is the human or team the sending agent acts for — this is what makes "who did this, for whom" answerable from the ledger alone. `in_reply_to` points at the earlier envelope on the thread this one answers, giving exact conversation ordering.
- **Timestamps are UTC.** `created_at` is RFC 3339 with a mandatory `Z` suffix — the schema rejects local-time offsets. The switchboard assigns it at acceptance; endpoints never set it.
- **`priority`** — 0 (highest, reserved for owner take-over) through 9; default 4. Queues sort by priority, then age.
- **`session_id`** — ad-hoc/first-contact calls get one session per call; standing offers open one long-lived session per workflow–endpoint pair at workflow creation.
- **Envelope size** — the envelope itself is metadata-only and must stay small; artifact bodies are never inline (architecture decision log #4). Cap: **16 KB** per serialized envelope (ratified).
- **Set-once semantics** — every field is immutable after the switchboard accepts the envelope; corrections travel as new envelopes (`in_reply_to` pointing back).

## Open for the v0 spec (not settled here)

1. Error/rejection envelope shape (how the switchboard reports validation failures to the sender).
2. Whether `artifact.digest` ever gains algorithms beyond sha256 (sha256-only is ratified for v0; agility is a future-version question).
3. Per-field length limits (the envelope-level 16 KB cap is ratified; individual field maxima are spec detail).
4. Per-artifact-type **fragment conventions** for sub-artifact references from bodies (`#L40-L60` for diffs, `#section` for docs) — defined alongside the artifact-type registry.
