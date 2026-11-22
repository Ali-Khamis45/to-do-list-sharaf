# Architecture review: AI Voice Companion & Intelligent To-Do Manager

**Status:** Review for discussion and approval; no implementation authorized.  
**Source:** [initial-plan1](initial-plan1), read in full. The requested `initial-plan.md` is empty (0 bytes); `initial-plan1` is the populated original plan. Section references below refer to that file. Its repository-state statements are historical.  
**Priority:** Critical = resolve before architecture approval; High = resolve before the affected milestone or beta; Medium = resolve during MVP planning; Low = defer or simplify.

## Assessment and decisions worth preserving

The plan identifies the right product risk: a companion must understand a user's actual commitments and earn trust. Preserve conversational replies, editable transcripts, text as a first-class input, evidence-backed suggestions, explicit review before task mutation, and visible change previews. Preserve one structured analysis as the starting hypothesis, deterministic date resolution and lifecycle validation, server-side user scoping, transactional application, retry protection, minimal conversation context, and a representative evaluation set. Retain the preference for a thin Next.js client and ASP.NET Core backend as a **preference**, not an approved irreversible decision; the original's D-01 and other open decisions remain open. Batch speech, no long-term memory, no autonomous tools, no queues or vector database are sensible defaults. The privacy intent is sound but its specification is not yet implementable.

## Critical findings

### C1. Semantic deduplication blocks unrelated work

**Evidence:** §7.5 step 7 declares a duplicate when title similarity ≥ 0.55 **or** dates are within ±1 day **or** the model marks `duplicate_of`; FR-11 says to block insertion.

**Problem and impact:** Any two tasks on the same or adjacent days may be suppressed, including unrelated commitments. A fuzzy title or model assertion can also suppress separate repetitions such as two lectures or recurring bank calls. This can silently lose intended work.

**Fix:** Separate exact request idempotency from semantic matching. Use text similarity, model suggestions and dates only to retrieve candidate matches, never as proof. Compare action, object/person, intended instance, relevant qualifiers and date in context. Classify as `distinct`, `possible_duplicate`, or `same_instance` with evidence; uncertain matches remain visible and user-resolvable. Never block a create solely on date proximity, similarity threshold, or an LLM flag. Provide an explicit “create anyway” and link or discard option. A repeated mention of the same commitment can be linked only when supported by context and confirmed by the user. Recheck candidates at apply time without silently changing the user's selection. Calibrate on labeled false-merge and false-split cases, including same title on different days and different tasks on the same day.

### C2. Deletion promises conflict with stored transcript derivatives

**Evidence:** §7.9 and S24 promise conversation deletion removes evidence while preserving tasks; §8 stores transcription text, message text, raw model output, validation reports, proposal payload/evidence, task evidence, task-event before/after JSON and cached idempotency responses. §11 says emotional content exists only in transcripts, although replies and model output may repeat it. Transcriptions have no deletion linkage or retention field.

**Problem and impact:** Deleting messages alone leaves identifiable or verbatim content elsewhere, including orphaned transcriptions, events, cached responses, logs and backups. The stated privacy guarantee is untrue and deletion cannot be reliably verified.

**Fix:** Adopt the data inventory and deletion matrix in the revised plan. Make one canonical owner and expiry for each transcription, message revision and derivative; avoid persisting raw model output and raw evidence in event snapshots and cached responses. Store only minimal operational metadata. On delete or expiry, atomically remove or redact all live derived content, clear task provenance/evidence while preserving user-approved task content, invalidate replay caches, and propagate a deletion tombstone to asynchronous work. Specify treatment of local audio, provider-side processing, telemetry, exports, backups and restore. Do not claim immediate deletion from systems outside application control. Test each store, crash/retry and a backup restore. Resolve `none`, pending proposals and previously exported files explicitly before implementation.

### C3. Reply asserts a task was saved before application

**Evidence:** §7.1 says the call can say “I added 4 tasks,” and §9's response says “حطيتلك ٤ حاجات” although the sequence applies only after confirmation.

**Problem and impact:** The companion can promise completed work that was never saved, failed, or was rejected. A generative reply cannot know the result of a subsequent transaction.

**Fix:** Before apply, refer to *suggestions* in natural language, e.g. “اقترحت ٤ مهام، راجعها واضغط تأكيد”. After apply, generate or template a separate acknowledgement exclusively from the committed server result and report rejected/conflicting items accurately. A single analysis call remains acceptable if the reply is constrained to proposal language and checked after validation; use a neutral fallback if consistency cannot be established. Never let an LLM report a persistence outcome from its own output.

### C4. Transcript correction can duplicate or silently alter applied work

**Evidence:** FR-03, §7.8 and S18 supersede pending items but assume a re-analysis will infer that already-applied items should become updates. The schema has no immutable revision history, dependency graph or correction reconciliation protocol; §7.5 mutates pending items directly.

**Problem and impact:** A corrected utterance can create a second task, update the wrong one, or overwrite a later manual edit; an in-flight apply may race a re-analysis. The user cannot know which earlier conclusions are being revised.

**Fix:** Use immutable message revisions and immutable analysis runs, with revision IDs and causation links from applied proposal to task/event. On correction, atomically mark only old *pending* items superseded and retain a minimal applied ledger. Compare the new interpretation with the previous applied outcome and current task versions. Already-applied tasks are never silently reversed or rewritten: present explicit update/cancel proposals or ask which task, preserving subsequent manual changes. If mapping is ambiguous, do not create a replacement automatically. Reject stale apply requests via proposal revision and target version; serialize relevant state changes in a transaction. A later conversational correction follows the same target and confirmation rules. Failed analysis leaves the edited message saved and old pending items visibly superseded; allow retry without duplicating committed effects.

## High findings

### H1. Logical day is incorrectly treated as the meaning of “بكرة”

**Evidence:** A.3 and S12 interpret “بكرة” at Tuesday 00:45 as Tuesday, while §7.6 applies `logicalToday + 1` to all relative days and uses it for weekdays.

**Problem and impact:** The plan assumes a contested conversational convention. After midnight some speakers mean the next calendar date, while others refer to the upcoming waking day. The same rule also makes “النهاردة,” “بعد بكرة,” weekday, dashboard and automatic defaulting interact unexpectedly.

**Fix:** Separate `calendarToday` (user-zone civil date), optional `planningDay` (04:00 grouping) and a `referenceInstant` captured when the message was sent. Proposed default: explicit relative words use civil dates; the boundary affects grouping/default views only. At 00:45 Tuesday “بكرة” tentatively resolves Wednesday; show the weekday and absolute date, and ask rather than guess when the conversational context suggests Tuesday. Treat this as an **open product decision**, validate with Egyptian speakers, and provide correction. Define “النهاردة/النهارده”, “امبارح”, “بعد بكرة”, named weekdays, “الخميس الجاي”, omitted dates, travel/timezone changes, DST, year wrap, past references and date-only versus timed deadlines. Store the input zone and anchor with the analysis. Never automatically invent a scheduled date from a deadline (S3); suggest it as an editable choice. See revised-plan decision D2.

### H2. Evidence matching and confidence do not prove intent

**Evidence:** §7.5 accepts fuzzy quote similarity ≥ 0.8 as the main invention guard, drops confidence < 0.5, and §7.4 treats “المفروض” as definite. §7.3 calls evidence verbatim while §7.7 normalizes lossy forms.

**Problem and impact:** A real quote can be negated, hypothetical, spoken by someone else, past, or merely an obligation; fuzzy matching can match a different person or number. Self-scored confidence is uncalibrated, and normalization can collapse different words. Silent drops hide missed commitments.

**Fix:** Capture exact offsets and exact quote against the immutable user revision; validate exact Unicode substring after a well-defined representation, without lossy matching as proof. Use normalization only for retrieval. Evidence is necessary, not sufficient: explicitly evaluate negation, speaker, commitment, temporal context and referent; uncertain cases require clarification or an unchecked suggestion. Preserve an auditable reason code for rejection without raw text. Never rely on the model's confidence as an automatic drop threshold; evaluate any future threshold empirically. A failed quote should produce a recoverable analysis warning rather than quietly presenting a complete-looking result.

### H3. Operation schema and API do not define one executable contract

**Evidence:** §7.3 has an illustrative object with nullable fields for all operation types; `clarify` appears in scope but is outside its enum; `duplicate_of` permits P aliases while §8 stores only a task ID; `deadline` is a placeholder; aliases are described as preventing hallucination, but invalid aliases are still possible. §9 uses different names and types, and excludes expected versions.

**Problem and impact:** Invalid combinations, ambiguous nulls, broken proposal references and stale updates reach the boundary between model and application. Generated OpenAPI types cannot enforce an unspecified model contract.

**Fix:** Define separate versioned, discriminated model-output and API schemas with explicit mappings. The model may propose `create`, `update_task`, `reschedule_task`, `complete_task`, `cancel_task`, `revise_proposal`, or `withdraw_proposal`; clarification is a separate nullable object, not an operation. Require an allowlisted alias of the correct type for changes, exact evidence, typed date expressions, and operation-specific fields. Distinguish field omitted, set and clear; reject unknown properties and invalid combinations. Resolve aliases against the exact user-scoped context snapshot, including expiry and versions. Publish an explicit API example and errors, and generate contract tests for both schemas. `duplicate_of` is an optional candidate reference only, not a decision.

### H4. Apply atomicity, concurrency and idempotency conflict

**Evidence:** §9 calls batch apply atomic yet returns `skipped`; §7.8 defines guarded transitions and a unique source proposal only for creates. If-Match says 409 in §9. Provider retries, apply retries and retention TTLs are not fully specified.

**Problem and impact:** Partial updates, duplicate changes/events or an incorrect success reply can result after network loss or simultaneous edits. An expired replay cache can rerun a mutation; DB atomicity cannot guarantee exactly one external STT call after a crash.

**Fix:** Choose one all-or-nothing transaction per apply request: validate ownership, pending status, proposal revision, target version and lifecycle for *all* selected and rejected IDs; on any conflict commit nothing and return item-level conflicts. Persist mutation, events, proposal decisions and idempotency outcome together. Same user/key plus same request returns the original outcome; same key plus a different canonical request returns a conflict. Durable proposal state prevents replay after cache expiry; a new intent needs a new key. Use strong opaque version tokens and HTTP 412 for failed `If-Match`, 409 for domain/state conflict. Recheck concurrent access at commit and document transaction isolation. External provider calls are at-least-once across crash recovery unless provider idempotency is available; record states and avoid claiming exactly-once billing. Lost responses must be recoverable without returning deleted private content.

### H5. Schema has incompatible deletion, ownership and history rules

**Evidence:** §8 promises hard task deletion while retaining a `task_events` row with a task foreign key; proposals have no explicit user_id; transcription/message relationships and multi-revision storage are undefined; `xmin` is exposed implicitly; events contain full before/after content.

**Problem and impact:** The proposed constraints either block deletion, cascade away the retained event, or require undocumented exceptions. Cross-user foreign keys and target aliases can be misused if an endpoint bypasses filters. A restore or concurrent update can invalidate opaque version assumptions.

**Fix:** Define ownership and foreign-key behavior in migrations and API authorization, not global filters alone. Every owned entity has an owner, and relationships enforce matching ownership or are validated in the same transaction; return a consistent not-found response for another user's objects. Use immutable revisions and typed lifecycle enums. For MVP, hard-delete the task and its events, or retain a content-free tombstone without a task FK if an audit obligation is approved; do not promise both unspecified behaviors. Prefer explicit version columns as opaque ETags; define changes on updates. Add required indexes, unique constraints, checks, revision links and a retention strategy before implementation.

### H6. Phase 0 tests too many systems at once

**Evidence:** §14 requires 60 cases, 40 clips, two LLM adapters, three STT adapters, domain services and full validation; §16 estimates about a week and says only dialect extraction can sink the product.

**Problem and impact:** Provider plumbing and audio collection delay proof of task understanding; language, conversation, privacy and UX may fail independently. The schedule has no staffing or annotation assumptions.

**Fix:** Make Milestone 0 a text-only, reproducible extraction and companion evaluation using one selected candidate, fixed inputs and a small harness. Include deterministic date, correction and proposal lifecycle cases without building the production stack. Start a consented audio feasibility sample only after the text gate; use one STT candidate first and add comparisons only if measured failures justify them. Set time ranges conditional on staffing and access, record actuals, and require a gate review rather than an automatic Phase 1 start.

### H7. Egyptian Arabic and code-switching are not adequately measured

**Evidence:** §5.3 relies on 40 clips/4–6 speakers and uses task-title prompts unconditionally; §12.2 allows LLM judging and does not define matching, sample denominators or slice gates. S17 assumes transliteration can always be expanded correctly.

**Problem and impact:** A small, non-independent set can conceal failure on names, dialect, gender, negation, digits, mixed scripts and noise. Vocabulary hints may bias the transcriber toward old tasks and send extra personal data to another provider. An LLM judge can miss wrong intent or date.

**Fix:** Use independently human-labeled, consented, de-identified cases; separate prompt-development and held-out sets by conversation and speaker where audio exists. Define one-to-one matching and per-operation/date/target metrics and error buckets. Report counts and confidence intervals, plus slices for dialect, code-switching, negation and ambiguity; never infer general population performance from a small sample. Include transliterated technical terms, similar Arabic names, ١/1 and ٣/4, pronouns, competing tasks, no-task venting and corrections before/after apply. Request clarification when normalization or STT is uncertain; preserve original transcript and allow edits. Test vocabulary hints as an optional privacy-reviewed experiment; send minimum necessary context and never assume improvement.

### H8. Privacy and security controls lack an executable boundary

**Evidence:** §11 says no audio is stored while §10 requires IndexedDB; no-training/limited-retention terms and EU hosting are assumptions, not verified. Conversation list/export/deletion, cookies and prompt injection protections are only sketched.

**Problem and impact:** Users may misunderstand local recovery, third-party processing, retention and access; shared devices and backups keep sensitive recordings. Same-origin and a global query filter alone do not provide full security.

**Fix:** Explain temporary client buffering and device-loss/shared-device limits; delete local audio after confirmed transcript persistence or user discard, with an expiry, storage-failure handling and a visible delete action. Before beta verify provider terms, regions, subprocessors and actual retention, and disclose them; don't send sensitive data to a fallback automatically without disclosure. Define authentication/session rotation, cookie/CSRF and origin controls, input limits, attachment validation, rate and spending limits, least privilege, encryption in transit/at rest and secret rotation. Treat all messages, transcripts and retrieved task text as untrusted prompt content; enforce authorization and operation validation outside the model. Redact telemetry and test user isolation across every endpoint and background job. Resolve backup deletion and account deletion processes before beta.

### H9. Acceptance tests omit task changes and transactional failure modes

**Evidence:** §12.2 uses small unspecified denominators, a ≥90% create precision measure and ≥90% modify accuracy, while wrong-target denominator and task matching are unspecified. The scenarios do not cover simultaneous edit/apply, stale versions, correction after manual edits, delete during processing, or incompatible apply selections.

**Problem and impact:** Aggregate scores can pass with unsafe failures hidden in small slices, and a test suite based on canned model output cannot establish real model performance.

**Fix:** Define unit-level deterministic and live-model acceptance separately. Track wrong-target numerator against all proposed modify operations and its upper confidence bound, and give every slice counts. Require zero wrong-target *applied* operations in controlled end-to-end tests, plus ambiguity abstention. Test retries after lost responses, concurrent apply, stale ETags, pending and applied corrections, user-edited tasks, deletion and provider failure. Freeze a held-out human-reviewed benchmark and require release-gate review with confidence intervals; targets are initial planning thresholds, not claims of achieved performance. See revised plan for Milestone 0 thresholds.

## Medium findings

### M1. Product experience could collapse into speech-controlled CRUD

**Evidence:** §1 says “talk to a friend,” but the pipeline and roadmap primarily score task extraction; §7.1 makes a single call responsible for reply and actions.

**Problem and impact:** The assistant can become an extraction form, respond mechanically to venting or continually push tasks. Extraction metrics alone miss the core value.

**Fix:** Preserve conversational response with no task required, acknowledge context without diagnosing, respect “don't organize,” remember only bounded current-session context, and ask at most one useful question when needed. Add human-rated language/tone, relevance and non-pushy behavior cases. The assistant must not claim therapeutic competence or retain inferred emotional memory. Decide whether an extraction and reply call should be split only after evaluation.

### M2. MVP includes too many coupled capabilities

**Evidence:** §3 includes workload with inferred estimates, multi-provider fallback, crash-safe chunking, full export, installable PWA, account settings, recurrence-adjacent rollover and numerous task operations alongside voice and companion quality.

**Problem and impact:** Coupled delivery hides the core validation and stretches a private-beta release. Inferred duration may create false workload certainty; PWA and mobile browser recovery involve device-specific constraints.

**Fix:** Keep a minimal conversational text/voice loop, explicit review, essential manual task management and privacy controls. Defer inferred estimates/workload, bulk rollover, fallback chains, PWA installation, themes and automatic repair unless evidence requires them. Account deletion and understandable privacy/deletion behavior remain beta prerequisites, not optional polish. Local audio recovery is best effort with explicit limits, verified on actual devices.

### M3. Default English titles may distort meaning

**Evidence:** D-04 defaults to translated English, S17 assumes a correct English expansion, while source evidence is Arabic.

**Problem and impact:** Translation can lose names, intent and code-switched terms and makes confirmation harder for Arabic-first users. English-only chrome may limit testing.

**Fix:** Keep original-language titles as the proposed default, preserve code-switched words and provide editable titles. Treat title language and Arabic UI as open product decisions and validate with target users; do not assume a translation is correct. Ensure bidi rendering and accessible labels regardless of UI language.

### M4. Transcript and task lifecycles lack an expiration policy for unresolved work

**Evidence:** §7.9 `none` purges only after proposals resolve; §8 daily job is unspecified for pending items, edits, in-flight jobs and orphan uploads.

**Problem and impact:** A pending tray can retain sensitive text forever; deleting during an in-flight provider request can resurrect data. A daily job in a sleeping API is not a time guarantee.

**Fix:** Set a finite pending-proposal TTL as an open policy decision; warn before expiry and discard unresolved drafts on expiry/deletion rather than silently create tasks. The durable worker must run while the app is idle, use leases and be idempotent; define deletion SLO and monitoring. Validate a deletion generation/tombstone before committing any async result. Cover provider data and backup restore in C2.

### M5. Date and deadline model misses timed and ambiguous cases

**Evidence:** §8 separates `deadline_date` and `deadline_time` without deadline timezone; §7.6 says past dates become clarification even though “completed yesterday” is valid; S3 conflates deadline and schedule.

**Problem and impact:** Travel, DST and partial times make deadlines ambiguous, and valid historical statements can be rejected or changed incorrectly.

**Fix:** MVP can use date-only tasks and deadlines. If timed deadlines are included, define local date/time, IANA zone, DST overlap/gap handling and derived instant, and how timezone edits affect them. Validate past dates by operation and user intent, not globally. Clarify ambiguous relative weekdays and retain original expression for the lifetime allowed by retention.

### M6. Reliability and operational targets are assertions, not measured budgets

**Evidence:** §4 sets stop-to-tray p95 ≤10s, §7.5 allows a 20s analysis timeout plus retries, and §14 assumes exactly one provider call and never-lost audio.

**Problem and impact:** These targets contradict timeout behavior and mobile browser eviction/crashes. Fallbacks can increase costs, retention and latency.

**Fix:** Start with measured stage-level p50/p95, timeout and cost budgets, sample size and network/device profile. Set a single end-to-end deadline and bounded retries; expose a saved-message/retry state. Validate realistic mobile performance before promising latency or recovery guarantees. A timeout is an uncertainty state, not proof that the provider did no work.

### M7. Operational layers are premature

**Evidence:** §5 mandates four .NET projects, `IAppDbContext`, two AI abstractions, Polly, OpenTelemetry stack, reverse proxy, pg_trgm, optional dual fallbacks and code generation before evidence of need.

**Problem and impact:** Extra boundaries, packages and infrastructure slow small-team delivery; trigram and provider portability introduce complexity without improving safety by themselves.

**Fix:** Keep a modular monolith with ordinary modules and EF directly unless boundaries demonstrate value. One narrowly scoped model adapter and one STT integration suffice initially; add `IClock` only where useful for tests. Use simple structured metrics/logs and minimal same-origin hosting; add search indexes, resilience libraries, OpenTelemetry exports and fallback adapters when measured needs justify them. Keep OpenAPI if used to define the API; generation is optional until it saves maintenance. Stack and host remain open.

## Low findings

### L1. Product presentation and delivery assumptions need validation

**Evidence:** §10 mandates two themes, haptics, wake locks, waveform, auto-opening Tomorrow after 18:00, and installability. Device support is asserted broadly.

**Problem and impact:** These features add test combinations and permissions without proving core conversational value; the day switch can surprise users.

**Fix:** Start with a responsive accessible interface and explicit day selection. Make enhanced visuals and installation optional follow-ups; test actual target devices and capability-detect optional browser APIs. Label accessibility targets as release criteria with manual checks in addition to automated scans.

## Approval-sensitive questions

1. Does the populated `initial-plan1` represent the authoritative original, and should it be renamed by its owner? Its contents were reviewed; neither source file was changed.
2. Approve the provisional civil-date interpretation of explicit “بكرة,” with confirmation when contextual meaning conflicts, and separately decide whether a 04:00 planning-day grouping is needed.
3. Choose stack and host (original D-01/D-08), title language and UI language (D-04/D-05), and target devices; these remain open.
4. Approve explicit-confirmation scope and whether conversational changes to already applied tasks require the same confirmation (recommended: yes).
5. Decide retention defaults, pending-draft expiry, local-audio expiry, backup deletion SLO and provider conditions before private beta.
6. Approve benchmark composition, provisional thresholds and staffing assumptions before treating Milestone 0 as a commitment.

See [revised-plan.md](revised-plan.md) for the coherent proposed contract, retention matrix and milestone gates. None of the recommendations above is represented as user-approved.
