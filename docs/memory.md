# Agent Memory

## Purpose

Each agent maintains a per-member memory store that persists across sessions. At the start of a session, relevant memory is injected as context. At the end of a session, a Haiku call extracts memory-worthy facts and updates the store.

This gives agents continuity without storing raw conversation history. The agent does not replay past queries because it works from a curated, structured summary of what it knows about the member.

---

## Storage - `member_agent_memory`

Table in MSSQL. One row per memory entry so entries can be retrieved selectively and expired individually.

| Column | Type | Notes |
|---|---|---|
| `id` | PK | |
| `member_id` | FK | |
| `agent_id` | GCF / TOF / MOF | |
| `memory_type` | enum | see per-agent types below |
| `content` | text | extracted fact or summary |
| `updated_at` | datetime | upserted on each session end |
| `expires_at` | datetime / null | null = no expiry; set for time-sensitive entries |

---

## Per-Agent Memory Types

### Huginn (GCF)

| Type | Example content |
|---|---|
| `language_preference` | `"is"` |
| `aircraft_preference` | `"Typically asks about ASK 21"` |
| `recurring_topic` | `"Recency requirements - asked 4 times"` |
| `session_summary` | `"Member is preparing for a cross-country flight and asked about airspace and weather minima"` |

### Nölva (TOF)

| Type | Example content |
|---|---|
| `language_preference` | `"en"` |
| `weak_topic` | `"Weather minima - answered incorrectly in 3 of last 5 attempts"` |
| `exam_history_summary` | `"Consistently strong on airspace, weak on meteorology and navigation"` |
| `session_summary` | `"Student reviewed 12 questions, 8 correct. Struggled with SIGMET interpretation"` |

`weak_topic` entries complement the structured scores in `StudentProgressQuery` - the C# progress tracker holds numerical scores; memory holds the narrative summary and pattern context.

### Völundur (MOF)

| Type | Example content |
|---|---|
| `language_preference` | `"is"` |
| `aircraft_interest` | `"Regularly queries TF-SAC and TF-GRO"` |
| `open_items_context` | `"Has been tracking SIB 2024-17 applicability for TF-SAC across multiple sessions"` |
| `session_summary` | `"Reviewed open compliance items for two aircraft, asked about SIB 2024-17 deadline"` |

---

## Session Flow

### Session start

1. Retrieve all `member_agent_memory` entries for this `member_id` + `agent_id` from MSSQL
2. Format as a compact context block
3. Inject before the first user message in the conversation

```
[Member context]
Language preference: Icelandic
Aircraft: Typically asks about ASK 21
Recent sessions: Preparing for cross-country flight, asked about airspace and weather minima
```

### Session end

1. Session timeout or user closes chat: BFF signals end of session
2. Python agent retrieves full conversation from Redis (`SimpleChatStore`)
3. Haiku call with the full conversation: extracts memory-worthy facts per type
4. Upsert entries in `member_agent_memory` (update if type exists, insert if new)
5. Clear Redis session

---

## Haiku Extraction

A lightweight Haiku call at session end. The prompt instructs Haiku to extract structured facts from the conversation in JSON format. One entry per memory type that was evidenced in the session. Types with no evidence are omitted.

No LLM call is made if the session had fewer than 3 exchanges (nothing worth extracting).

---

## Retention Policy

| Memory type | Retention behaviour |
|---|---|
| `language_preference` | Upsert in place - one entry per agent per member |
| `aircraft_preference` / `aircraft_interest` | Upsert in place |
| `recurring_topic` / `open_items_context` | Upsert in place - content updated each session |
| `weak_topic` | Keep all entries; older entries weighted lower during context injection |
| `session_summary` | Keep last 5 per agent per member; oldest deleted on overflow |
| `exam_history_summary` | Upsert in place |

Hard limit: all entries deleted 1 year after `updated_at` with no new session activity.

---

## GDPR

`member_agent_memory` is personal data. It is:
- Included in the Article 15 member data export report
- Deleted as part of the member data deletion policy (active members: retained; post-membership: deleted after the agreed retention period)
- Mentioned in the privacy policy

No raw query text is stored in memory with only Haiku-extracted summaries and structured facts.
