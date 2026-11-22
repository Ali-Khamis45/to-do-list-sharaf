# Decisions, Open Questions & Risks

Status: Approved for planning (rev 2) · Last updated 2026-11-22
Related: [product-requirements.md](product-requirements.md) · [architecture-and-ai-pipeline.md](architecture-and-ai-pipeline.md) · [data-and-api.md](data-and-api.md) · [roadmap-and-testing.md](roadmap-and-testing.md)

Each entry gives the status, the decision, and the reason for it. A new entry gets added whenever a decision changes. Old entries are kept for history.

---

## 1. Open decisions

| ID | Decision | Status | Default / lean |
|---|---|---|---|
| **D-01** | Backend language: ASP.NET Core, Go, or all-TypeScript | **OPEN. Needs both of us to decide before P0.** | Lean **Go**, if learning it is a real goal. See ADR-D01. |
| D-02 | STT provider | Pending P0 bake-off | Chosen on downstream task F1 first |
| D-03 | LLM provider and model | Pending P0 eval | One adapter first. A second only if the gate is missed. |
| D-04 | Task title language | Accepted (default) | English, with a setting for Arabic or match-speech |
| D-05 | Arabic UI in MVP | Accepted (default) | No. English UI with RTL-safe content. Arabic UI in V1. |
| D-06 | Auto-apply vs review | Accepted | Review with one tap in MVP. Opt-in auto-apply in V1 if precision is ≥ 0.95. |
| D-07 | Transcript retention default | Accepted (default) | 30 days |
| D-08 | Hosting | Accepted (default) | A cheap single host for Next.js and the backend, plus managed Postgres in an EU region |
| D-09 | Accounts | Accepted | 2 seeded accounts. Registration in V1 hardening. |
| D-10 | Day boundary | Accepted (default) | 04:00 local, user-adjustable |

### ADR-D01: Backend language (OPEN)

**Context.** The original preference was ASP.NET Core. We are both learning Go. Rev-2 scope cuts removed much of what ASP.NET's built-ins were buying: Identity registration, OpenTelemetry, If-Match, and idempotency middleware. Full comparison: [architecture-and-ai-pipeline.md §1.2](architecture-and-ai-pipeline.md).

**Options.**
- **Go.** High learning value, a single static binary, and SQL-first via `sqlc`.
  - Costs: more hand-written code, a small civil-date type, and about 20% extra on the backend phases.
- **ASP.NET Core.** Best-in-class local dates through NodaTime, plus mature tooling.
  - Costs: low learning value unless C# is a goal, and the heaviest ceremony.
- **All-TypeScript.** The fastest route to an MVP, with one deployable and shared Zod schemas for API and LLM output.
  - Costs: business logic can drift into the frontend codebase.

**Lean.** Go, if learning it is a real goal and neither of us is fluent in C#. ASP.NET Core, if one of us is already productive in C# and shipping sooner matters more.

**Decision.** _Not yet made._

**Consequence.** The domain core and the eval runner in P0 are written in this language.

---

## 2. Accepted architecture & product decisions

### ADR-001: Proposals with one-tap confirm, not silent auto-creation
- **Decision:** The AI emits proposed operations into a Plan Tray. Definite items are pre-checked, and one tap applies them.
- **Why:** A sometimes-wrong model writing silently to the list destroys trust. One tap still meets "no retyping".
- **Revisit:** In V1, consider opt-in auto-apply with undo once eval precision is ≥ 0.95.

### ADR-002: Three surfaces instead of six screens
- **Decision:** The app has three surfaces: the Day view (Today/Tomorrow/any date), Talk, and Settings & Privacy.
- **Why:** Today Dashboard and Daily Checklist were the same screen. Tomorrow Planner is the same view on another date.
- **Revisit:** History arrives in V1.

### ADR-003: No "deferred" status
- **Decision:** Statuses are `pending`, `completed`, and `cancelled`. Deferral is a `rescheduled` event plus `defer_count`. A null date means "Later".
- **Why:** A task moved to Friday is pending on Friday. A separate status would duplicate the date.

### ADR-004: Lean modular backend, not full Clean Architecture
- **Decision:** The backend is split into `domain`, `app`, `store`, `ai`, `api`, and `cmd/eval`. There is no mediator and no repository-over-ORM layering.
- **Why:** At 2 users, the value is in a pure, testable domain core. The extra layers would add ceremony without payoff.

### ADR-005: Batch STT, no streaming or TTS in MVP
- **Decision:** Recording follows record → upload → transcript. The waveform is drawn locally from the mic.
- **Why:** A nightly monologue tolerates 3–6 s of wait after "stop". Streaming and duplex voice add large complexity.
- **Revisit:** SSE replies in V1. Streaming STT and duplex voice in V2.

### ADR-006: Deterministic date resolution
- **Decision:** The LLM emits a date expression. A pure resolver converts it using the IANA timezone, a 04:00 day boundary, and a Saturday week start. The tray always shows the resolved date.
- **Why:** LLMs get weekday arithmetic wrong around midnight and DST. Egypt reinstated DST in 2023.

### ADR-007: Policy decides, confidence only demotes
- **Decision:** Pre-check status depends on four things: commitment class, a verbatim evidence quote, reference resolution, and dedupe. Confidence < 0.5 demotes an item to "Maybe" with a low-confidence flag. It is never dropped silently.
- **Why:** Self-reported model confidence is poorly calibrated. Dropping items silently hides recall failures.
- **Changed in rev 2:** Low-confidence items were previously dropped.

### ADR-008: Number-aware dedupe with AND semantics
- **Decision:** An item is a duplicate if (trigram ≥ 0.55 **AND** date within ±1 day or Later **AND** digit tokens match) **OR** (a validated `duplicate_of` **AND** digit tokens match). A number mismatch overrides the model's `duplicate_of`.
- **Why:**
  - With OR, every same-day task would get flagged.
  - Without number awareness, "OS lecture 1" and "OS lecture 2" merge, which broke S1.
- **Changed in rev 2:** This replaced the earlier OR rule.

### ADR-009: Evidence window is the last 3 user messages
- **Decision:**
  - Evidence quotes must fuzzy-match (≥ 0.8) a span in `U-0`, `U-1`, or `U-2`.
  - A create cited only by `U-1` or `U-2` must survive dedupe against everything already proposed or applied from those messages.
- **Why:** Clarification answers ("خليها الخميس") and corrections depend on earlier turns. The dedupe guard stops the model re-proposing old content.
- **Changed in rev 2:** Evidence was previously limited to the current message only. Scenario S26 was added for this.

### ADR-010: Idempotency via client IDs and unique constraints, no generic table
- **Decision:** Retry safety comes from four keys:
  - `uploadId`, `clientMessageId`, and the client-generated task `id`, each enforced by a unique constraint.
  - A guarded `pending → applied` transition, plus a unique `tasks.source_proposal_item_id`.
- **Why:** These cover every create path with zero duplicate inserts, without an `idempotency_keys` table.

### ADR-011: Same-origin via Next.js rewrites
- **Decision:** The browser calls `/api/*` on the Next.js origin, and Next.js rewrites those requests to the backend.
- **Why:** Cookies stay first-party, there is no CORS, and no reverse proxy is needed in the MVP.

### ADR-012: One STT adapter and one LLM adapter
- **Decision:**
  - The STT bake-off is a throwaway Python script, and only the winner gets a production adapter.
  - There is one LLM adapter. A second is built only if the first misses the gate after 2 prompt iterations.
- **Why:** 2 people × 2 h/day can't afford to maintain adapters that aren't used.

### ADR-013: MVP scope sized for 2 users at 2 h/day
- **Decision:** Several items move to the V1 hardening list:
  - registration and invite codes
  - account deletion
  - JSON export
  - OpenTelemetry
  - the `idempotency_keys` table
  - If-Match
  - Caddy
  - a provider fallback chain
  - ASVS/WCAG as release gates
  - the 5–20 user beta gate
- **Why:** None of these change whether the core loop works for the two of us. Account deletion moved because it goes with registration. The MVP keeps "delete all my data".
- **Also:** The golden set is 30 text cases and 20 audio clips. At that size the gate is directional and grows from real failures.

### ADR-014: No Redis, queues, vector DB, microservices, or agent orchestration
- **Decision:** None of these are used in the MVP.
- **Why:** The pipeline is synchronous and under 10 s. Task context is small, and `pg_trgm` covers similarity search. One structured call plus deterministic code is more reliable than an agent loop.

---

## 3. Review log

| Date | Change |
|---|---|
| 2026-11-22 | Rev 1 plan drafted. |
| 2026-11-22 | Rev 2 after review:<br>• Dedupe AND fix and number-aware dedupe (ADR-008)<br>• Evidence window of the last 3 messages (ADR-009)<br>• Low confidence goes to Maybe (ADR-007)<br>• Scope cuts and estimates at 2 h/day (ADR-012, ADR-013)<br>• Docs consolidated to 4 files + this one<br>• D-01 reopened with a Go option |

---

## 4. Risk register

| Risk | Likelihood / impact | Mitigation |
|---|---|---|
| STT weak on Egyptian code-switching | Med / High | Bake-off judged on downstream F1. Vocabulary biasing. Editable transcript. The LLM tolerates transliteration. |
| Over-extraction (invented tasks, aspirations promoted) | Med / High | Evidence check, commitment classes, the Maybe section, review before apply, and false-task rate in the gate |
| Wrong modify target | Med / High | Aliases, a clarification when ambiguous, change rows, and an explicit tap |
| Multi-turn evidence re-proposes old tasks | Med / Med | The last-3 window, plus mandatory dedupe for creates cited only in earlier turns (S26) |
| Night and DST date bugs | Med / Med | A pure resolver with table tests, and explicit dates in the tray |
| iOS PWA audio quirks | High / Med | Device tests in P3, an mp4 fallback, and text input |
| Small golden set hides regressions | High / Med | Treat the gate as directional. Add every real failure. Target 60+ cases by V1. |
| Go learning curve (if chosen) | Med / Med | +20% in the estimates. The pure domain core in P0 serves as the learning ground. Keep libraries simple. |
| 2 people at 2 h/day: schedule slips | High / Med | Parallel tracks. The P0 go/no-go comes before UI work. V1 hardening stays parked. |
| Provider deprecation or price change | Med / Med | Ports, pinned versions, and an eval gate on every swap |
| Privacy perception ("it listens to my feelings") | Med / High | No audio storage, a clear retention control, deletion, and no long-term memory in the MVP |
