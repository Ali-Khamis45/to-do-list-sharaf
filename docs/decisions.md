# Decisions, Open Questions & Risks

Status: Planning, rev 3 · Last updated 2026-11-22
Related: [plan.md](plan.md) (source) · [product-requirements.md](product-requirements.md) · [architecture-and-ai-pipeline.md](architecture-and-ai-pipeline.md) · [data-and-api.md](data-and-api.md) · [roadmap-and-testing.md](roadmap-and-testing.md)

Each entry gives the decision, its status, and the reason for it. When a decision changes, add a new entry and keep the old one for history.

---

## 1. Decision status

| ID | Decision | Status | Value |
|---|---|---|---|
| **D-01** | Backend language | **Decided (rev 3)** | **Go**: pgx v5 + sqlc, goose, net/http (1.22+ routing) or chi. P1 picks between net/http and chi. |
| D-02 | STT provider | Pending P0 bake-off | Chosen on downstream task F1 first |
| D-03 | LLM provider and model | Pending P0 eval | One adapter. A second only if the gate is missed. |
| **D-04** | Task title language | **Decided (rev 3)** | The language the user spoke, with English technical terms kept as spoken. Editable. |
| D-05 | Arabic UI in MVP | Accepted | No. English UI with RTL-safe content. Arabic UI in V1. |
| D-06 | Auto-apply vs review | Accepted | Review with one-tap confirm. Opt-in auto-apply in V1 once precision ≥ 0.95. |
| D-07 | Transcript retention default | Accepted | 30 days |
| D-08 | Hosting | Accepted (default) | A cheap single host plus managed Postgres, in an EU region |
| D-09 | Accounts | Accepted | 2 seeded accounts |
| **D-10** | Day boundary | **Decided (rev 3)** | 04:00. At 00:45 Tue, "بكرة" means Tue, and the tray shows the night date line. No external validation. |

### ADR-D01: Backend language = Go (decided)

**Context:** The original preference was ASP.NET Core. We are both learning Go. The rev 2 cuts removed most of what ASP.NET's built-ins were buying.

**Decision:**
- **Language:** Go.
- **Data:** `pgx` v5 + `sqlc` for data access, `goose` for migrations.
- **HTTP:** `net/http` with 1.22+ pattern routing. Switch to `chi` only if middleware grouping gets awkward.
- **Libraries:** `scs` for sessions, `slog` for logs, and `testcontainers-go` for tests.
- **API contract:** A spec-first `openapi.yaml` with `oapi-codegen` types.

**Consequences:**
- The domain core and the eval runner are written in Go from P0.
- Backend estimates include about 20% learning overhead.
- A small civil-date type is needed.
- Trigram scores come from Postgres, so the P0 eval runner uses Compose Postgres.

---

## 2. Accepted architecture & product decisions

### ADR-001: Suggestions with one-tap confirm, not silent auto-creation
The AI emits proposed operations into a Plan Tray. Definite items are pre-checked, and one tap applies them. The reason is that a sometimes-wrong model writing silently to the list destroys trust.

### ADR-002: Three surfaces instead of six screens
The app has a Day view, Talk, and Settings & Privacy. Today Dashboard and Daily Checklist were the same screen, and Tomorrow is the same view on another date.

### ADR-003: No "deferred" status
Statuses are `pending`, `completed`, and `cancelled`. A move is a `rescheduled` event that increments `defer_count`. A null date means Later.

### ADR-004: Lean modular backend
A Go modular monolith with a pure `internal/domain` and thin adapters. No mediator and no repository-over-ORM layering.

### ADR-005: Batch STT, no streaming or TTS in MVP
Record, upload, then transcribe. The waveform is drawn locally.

### ADR-006: Deterministic date resolution, 04:00 rule kept (updated rev 3)
- **Resolver:** The LLM emits a date expression. A pure resolver turns it into a date using the IANA timezone, the 04:00 day boundary, and a Saturday week start.
- **Night date line:** Between 00:00 and 04:00, items resolved from a relative expression show a server-templated line, for example "بكرة = التلات 24 نوفمبر".
- **Why:** LLMs get weekday arithmetic wrong. We are the users, and this is the convention we use.

### ADR-007: Policy decides, confidence only demotes
- **Inputs:** Pre-check status depends on the commitment class, the server-located evidence, reference resolution, and dedupe.
- **Low confidence:** Confidence < 0.5 sends an item to "Maybe" with a low-confidence flag. It is never dropped silently.

### ADR-008: Dedupe metric and threshold (updated rev 3)
- **Score:** `s = GREATEST(similarity(a,b), word_similarity(a,b), word_similarity(b,a))`, computed by pg_trgm on normalized titles.
- **Rule:** Two items are duplicates if either:
  - `s ≥ 0.84`, AND the dates are within ±1 day or one is Later, AND the digit tokens match; or
  - `duplicate_of` is validated AND the digit tokens match.
  A number mismatch overrides `duplicate_of`.
- **Why:** Plain `similarity()` gives 0.484 for "reply to ahmed" vs "reply to ahmed about the project", so a 0.55 threshold fails a required case.
- **Threshold source:** 19 pairs measured in Postgres 17; the table is in architecture-and-ai-pipeline.md §7.5.1. The highest distinct pair scored 0.815 and the lowest true duplicate 0.862.
- **Known limits:**
  - The margin between those two is thin.
  - A one-word title scores 1.0 against any longer title that contains it.
  - Arabic trigrams depend on the database locale, so the fixture is re-run on the real managed Postgres in P1.
- **History:** Rev 2 fixed the OR rule and made dedupe number-aware. Rev 3 replaced plain similarity.

### ADR-009: Evidence window and re-proposal guard (updated rev 3)
- **Window:** Evidence must be located in `U-0`, `U-1`, or `U-2`.
- **Re-proposal guard:** A create whose evidence is only in `U-1`/`U-2` is checked against every item from those messages, including **rejected and withdrawn** ones. The check is on title only: score ≥ 0.84 AND numbersMatch.
- **On a match:** The create is dropped with `reproposal_of_earlier_item` and no notice. Covered by S30.
- **Why:** Clarification answers depend on earlier turns. Without the guard, a rejected item could come back.

### ADR-010: Idempotency via client IDs and unique constraints
Each of these is enforced by a unique constraint: `uploadId`, `clientMessageId`, the client-generated task `id`, and `tasks.source_proposal_item_id`. Proposal `revision` handles stale edits. There is no generic `idempotency_keys` table.

### ADR-011: Same-origin via Next.js rewrites
`/api/*` is rewritten to the Go API, so cookies stay first-party.

### ADR-012: One STT adapter and one LLM adapter
- **STT:** The bake-off is a throwaway Python script. Only the winner gets a Go adapter.
- **LLM:** A second adapter is built only if the first misses the gate after 2 prompt iterations.

### ADR-013: MVP scope sized for 2 builders who work when free
- **V1 hardening list:** registration, export, OpenTelemetry, an idempotency table, If-Match, Caddy, provider fallbacks, ASVS/WCAG gates, and a wider beta.
- **Estimates:** Hours are the real estimate. Calendar weeks are rough.

### ADR-014: No Redis, queues, vector DB, microservices, or agent orchestration

### ADR-015: Suggestion-only replies and templated acknowledgements (rev 3, C3)
- **Before apply:** The LLM reply describes suggestions only. It contains no counts and no "saved/added" claims. The tray summary is a server template, for example "اقترحتلك ٤ حاجات، راجعها وأكد".
- **After apply:** The acknowledgement is a `template_ack` message built from the apply response, for example "تمام، ضفت ٤ حاجات لبكرة (التلات 24 نوفمبر).".
- **Why:** A generative reply can't know the result of a later transaction.

### ADR-016: Clarification object and a discriminated op union (rev 3, H3)
- **Clarification:** `clarification` is a separate nullable object, not an op. "clarify" is removed from the op list.
- **Op variants:** Each op is a variant with its own required and optional fields and `additionalProperties: false`. The variants are `create`, `update_task`, `reschedule_task`, `complete_task`, `cancel_task`, `revise_proposal`, and `withdraw_proposal`.
- **Field changes:** `update_task` and `revise_proposal` use explicit `set` and `clear`. Omitting a field leaves it unchanged. Putting the same field in both is rejected.
- **Why:** Ambiguous nulls must never clear data, and invalid combinations must fail at the boundary.

### ADR-017: All-or-nothing apply with revisions (rev 3, H4 + C4)
- **Request:** Apply carries `{itemId, revision, decision}` for every item.
- **Transaction:** One transaction locks the conversation row, checks every item, and commits all or nothing.
- **Conflicts:** Any conflict returns 409 with item-level conflicts. There is no "skipped" result. A stale revision is a 409.
- **Retry:** If every item is already in the requested state, the server returns the stored outcome.
- **Race with edits:** Message edit takes the same lock, so apply and edit on the same proposal have exactly one winner. S31 runs this 100× concurrently.

### ADR-018: Deletion scope and stored derivatives (rev 3, H5 + C2)
- **Task delete:** Hard-deleting a task deletes its `task_events` through an FK cascade.
- **Events:** They store field names and before/after values for an allowlist of task fields only. No evidence and no quotes.
- **Model output:** `analysis_runs.raw_output` is **not stored**. Only per-op validation reason codes are kept.
- **Conversation delete:** It deletes its transcriptions, runs, and proposal items with their evidence. It also sets `tasks.source_message_id` and `tasks.evidence_ref` to null.
- **Orphans:** Orphan transcriptions are purged after 24 h.
- **Scope:** This is the full MVP privacy scope.

### ADR-019: Server-located evidence offsets (rev 3, H2)
- **Model:** It returns quote text only, never offsets.
- **Locating:** The server normalizes the quote and the referenced message (§7.7). It looks for an exact substring first, then an approximate match with edit-distance ratio ≥ 0.8.
- **Storage:** The match is mapped back to original code-point offsets and stored as `{message_id, message_revision, start, end}`.
- **Display:** The displayed quote is always cut from the original message text.

### ADR-020: Titles in the spoken language (rev 3, M3)
D-04: titles are in the language the user spoke, and English technical terms stay in Latin script as spoken. The scenario titles in §12.3 have been updated to match.

### ADR-021: Golden set with a real-night held-out split (rev 3)
- **Held-out set:** At least 10 of the 30 text cases come from our real nights. They are recorded and labeled before prompt v1, then frozen and reported as a held-out split.
- **Gate:** It must pass on both the dev and held-out splits.
- **Isolation:** The prompt author never reads held-out cases. A held-out case opened for debugging moves to dev and is replaced.

---

## 3. Codex review of rev 1: disposition

The review is in [architecture-review.md](architecture-review.md).

**Accepted in rev 3:**
- C3 → ADR-015
- H3 → ADR-016
- H4 and C4 → ADR-017
- H5 and a small part of C2 → ADR-018
- H2 → ADR-019
- M3 → ADR-020
- H1, as "keep the 04:00 rule and add a night date line" → ADR-006

**Rejected:**
- **An 80-turn evaluation set with independent annotators and confidence intervals:** Rejected. We are the only two users. 30 text cases with a real-night held-out split and a directional gate are enough.
- **A backup, processor, and legal retention matrix:** Rejected. The MVP privacy scope is ADR-018 plus the retention setting.
- **An incident response process:** Rejected. We are two users who are also the operators.
- **Provider-terms procurement:** Rejected. We check each provider's no-training API terms ourselves when we create the accounts.
- **Full-time-engineer estimates:** Rejected. We plan by scope and hours, and calendar weeks are rough.

**Other findings:** These were either addressed in rev 2 already or are not adopted for the MVP.

---

## 4. Review log

### Rev 1 (2026-11-22)
- Plan drafted.

### Rev 2 (2026-11-22)
- Dedupe AND rule and number awareness (ADR-008).
- Evidence window of the last 3 messages (ADR-009).
- Low confidence goes to Maybe (ADR-007).
- Scope cuts (ADR-012, ADR-013).
- 4 docs plus this file.

### Rev 3 (2026-11-22)
- **D-01 = Go** (ADR-D01).
- Dedupe metric is `GREATEST(similarity, word_similarity both ways)`, with a measured threshold of 0.84 (ADR-008).
- The re-proposal guard covers rejected and withdrawn items, with new scenario S30 (ADR-009).
- Held-out split of ≥ 10 real nights, and the gate must pass on it (ADR-021).
- Suggestion-only replies and templated acknowledgement (ADR-015).
- Clarification object and op union with set/clear (ADR-016).
- All-or-nothing apply with revisions, and exactly one winner against message edit. New scenarios S31–S33 (ADR-017).
- Task events cascade on delete and hold no evidence. No `raw_output`. Conversation delete removes transcriptions and evidence (ADR-018).
- Server-located evidence offsets (ADR-019).
- D-04: titles in the spoken language (ADR-020).
- The 04:00 rule is kept, with the night date line (ADR-006).
- Deleted the empty `initial-plan.md`.

---

## 5. Risk register & decision table (plan §15)

| Risk | L / I | Mitigation |
|---|---|---|
| STT weak on Egyptian code-switching | Med / High | Bake-off judged on downstream F1. Vocabulary biasing. Editable transcript. |
| Over-extraction | Med / High | Server-located evidence, commitment classes, Maybe, review before apply |
| Wrong modify target | Med / High | Typed aliases, the clarification object, change rows, explicit confirm |
| Re-proposal of resolved items | Med / Med | Re-proposal guard covers every status of earlier-message items (S30) |
| Dedupe threshold margin is thin (0.815 vs 0.862), plus the containment effect | Med / Med | Fixture table in CI. Add every real false merge or split and re-tune. Dates and numbers gate too. |
| Arabic trigrams depend on DB locale | Low / High | The P1 fixture runs on the real managed Postgres |
| Apply/edit races | Med / High | Conversation row lock, revisions, all-or-nothing, S31 at 100× |
| Night/DST date bugs | Med / Med | Pure resolver with table tests. Explicit date plus the night line. |
| iOS PWA audio quirks | High / Med | Device tests in P3, mp4 fallback, text input |
| Small golden set hides regressions | High / Med | Directional gate. Held-out split from real nights, refreshed during daily use. |
| Go learning curve | Med / Med | Included in the hour estimates. The pure domain package is the learning ground. |
| Schedule slips (part-time) | High / Low | Hours are the plan and weeks are rough. The P0 go/no-go comes before UI work. |
| Provider changes | Med / Med | Interfaces, pinned versions, eval gate |

| ID | Decision | Status |
|---|---|---|
| D-01 | Backend language | **Decided: Go** (pgx + sqlc, goose, net/http or chi) |
| D-02 | STT provider | Pending the P0 bake-off |
| D-03 | LLM provider | One adapter, chosen in the P0 eval |
| D-04 | Task title language | **Decided: as spoken**, with English technical terms kept as spoken. Editable. |
| D-05 | Arabic UI in MVP | No (V1) |
| D-06 | Auto-apply vs review | Review in the MVP. Opt-in auto-apply in V1 once precision ≥ 0.95. |
| D-07 | Retention default | 30 days |
| D-08 | Hosting | Cheap single host plus managed Postgres, in an EU region |
| D-09 | Accounts | 2 seeded accounts |
| D-10 | Day boundary | **Decided: 04:00** (so "بكرة" at 00:45 Tue means Tue), plus the night date line. No external validation. |
