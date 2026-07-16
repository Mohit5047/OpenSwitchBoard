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
- **IDs** are switchboard-assigned UUIDv4s with type prefixes (`env_`, `thr_`, `ses_`). Endpoints never mint them. Chronological ordering comes from `created_at`, not the ID.
- **`priority`** — 0 (highest, reserved for owner take-over) through 9; default 4. Queues sort by priority, then age.
- **`session_id`** — ad-hoc/first-contact calls get one session per call; standing offers open one long-lived session per workflow–endpoint pair at workflow creation.
- **Envelope size** — the envelope itself is metadata-only and must stay small; artifact bodies are never inline (architecture decision log #4). Cap: **16 KB** per serialized envelope (ratified).
- **Set-once semantics** — every field is immutable after the switchboard accepts the envelope; corrections travel as new envelopes (`in_reply_to` pointing back).

## Open for the v0 spec (not settled here)

1. Error/rejection envelope shape (how the switchboard reports validation failures to the sender).
2. Whether `artifact.digest` ever gains algorithms beyond sha256 (sha256-only is ratified for v0; agility is a future-version question).
3. Per-field length limits (the envelope-level 16 KB cap is ratified; individual field maxima are spec detail).
