# Envelope schema (draft, `envelope/v0`)

**Status:** Draft — the low-level design companion to [architecture.md §4](architecture.md). This is the "draft envelope schema that feeds the v0 spec" deliverable; the v0 spec gives it the final field-level freeze.
**Ratified basis:** the 11-field contract (four brief-settled fields + seven proposed extras) per architecture decision log #1, 2026-07-12.

## Wire format

JSON, schema-versioned. Every payload that moves through the switchboard is exactly one envelope.

```jsonc
{
  "schema": "envelope/v0",
  "envelope_id": "env_01J8Z3K7Q2M4N6P8R0T2V4X6Y8",   // unique, switchboard-assigned (ULID)
  "thread_id": "thr_01J8Z3K7Q2M4N6P8R0T2V4X6Y8",     // conversation identity; stable across revise loops
  "session_id": "ses_01J8Z3K7Q2M4N6P8R0T2V4X6Y8",    // the handshaken session this travels on
  "verb": "request",                                  // exactly one of the closed set
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
    "in_reply_to": "env_01J8Z2A1B3C5D7E9F1G3H5J7K9"    // prior envelope on the thread, if any
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
    "envelope_id": { "type": "string", "pattern": "^env_[0-9A-HJKMNP-TV-Z]{26}$" },
    "thread_id": { "type": "string", "pattern": "^thr_[0-9A-HJKMNP-TV-Z]{26}$" },
    "session_id": { "type": "string", "pattern": "^ses_[0-9A-HJKMNP-TV-Z]{26}$" },
    "verb": { "enum": ["request", "respond", "revise", "approve", "reject", "escalate", "inform"] },
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
        "in_reply_to": { "type": "string", "pattern": "^env_[0-9A-HJKMNP-TV-Z]{26}$" }
      }
    },
    "priority": { "type": "integer", "minimum": 0, "maximum": 9, "default": 4 },
    "created_at": { "type": "string", "format": "date-time" }
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
- **IDs** are switchboard-assigned ULIDs with type prefixes (`env_`, `thr_`, `ses_`). Endpoints never mint them.
- **`priority`** — 0 (highest, reserved for owner take-over) through 9; default 4. Queues sort by priority, then age.
- **`session_id`** — ad-hoc/first-contact calls get one session per call; standing offers open one long-lived session per workflow–endpoint pair at workflow creation.
- **Envelope size** — the envelope itself is metadata-only and must stay small; artifact bodies are never inline (architecture decision log #4). Proposed cap: 16 KB per serialized envelope.
- **Set-once semantics** — every field is immutable after the switchboard accepts the envelope; corrections travel as new envelopes (`in_reply_to` pointing back).

## Open for the v0 spec (not settled here)

1. Error/rejection envelope shape (how the switchboard reports validation failures to the sender).
2. Whether `artifact.digest` supports algorithms beyond sha256 (agility vs. simplicity).
3. Exact size cap and per-field length limits (16 KB proposed above).
4. Extension mechanism, if any, for org-private metadata (leaning: none in v0 — `additionalProperties: false`).
