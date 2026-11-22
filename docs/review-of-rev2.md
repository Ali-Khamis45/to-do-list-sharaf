# Review of rev 2

The five rev 2 documents are the source; the earlier review supplies finding IDs only. Status reflects the two-builder scope. “Fixed” means the original concern is addressed, not that every related risk is gone. Proposed fixes below are planning recommendations.

**Fixed decisions:** Go; 04:00 logical day for “بكرة” (Tuesday 00:45 → Tuesday); one-tap confirmation before persisted task changes; text-first Milestone 0 with voice required for MVP; one structured LLM call plus deterministic validation; 30 text cases, 20 audio clips and 10 held-out real nights. These override conflicting rev 2 text. Preserve the companion experience, two seeded accounts and lean design.

## Critical

- **C1 — fixed-in-rev2:** ADR-008 removes date-only duplicate blocking. Residual risk: a model hint or similar unnumbered title can still suppress a distinct task. Show possible matches with “create anyway”; test adjacent-day repeats and distinct same-day tasks.
- **C2 — still-open:** Transcript text can remain in transcriptions, validation reports, proposal payloads, task evidence and event snapshots after deletion. Link transcriptions to messages or clear their text after message creation; on expiry/deletion remove runs and proposals and clear transcript-derived task evidence and event content while keeping tasks. Expire orphan transcriptions. Test deletion and “delete all my data” with recognizable text in each store.
- **C3 — still-open:** The API example claims tasks were added before confirmation. Say “اقترحت ٤ مهام” before apply; acknowledge only committed results afterward. Test failure and rejection.
- **C4 — still-open:** Re-analysis assumes it will correctly find already-applied tasks. Link the original proposal to its applied task; supersede only pending proposals on correction. Leave applied tasks unchanged and present an explicit change proposal, or ask if the target is unclear. Test corrections before and after apply and after a manual edit.

## High

- **H1 — fixed-in-rev2:** The agreed 04:00 interpretation is documented. Keep dates visible; test before and at 04:00, year wrap and DST. Do not change the meaning of “بكرة.”
- **H2 — still-open:** Fuzzy evidence matching can confuse names, numbers and negation. Require exact quotes from the referenced `U-0`/`U-1`/`U-2` message; keep the three-message window. Evidence alone does not prove intent: ask or show an unchecked Maybe when uncertain.
- **H3 — still-open:** The illustrative operation schema leaves required fields and task/proposal aliases inconsistent. Define required and forbidden fields per operation, make clarification separate, specify `T`/`P` references and reject invalid combinations. Align API examples.
- **H4 — still-open:** Atomic apply conflicts with `skipped` results, and one STT call cannot be guaranteed after a crash. Validate all selected items, commit task changes and guarded proposal transitions together, or commit none and return conflicts. Retry returns stored outcomes; test concurrent apply and lost responses. Permit a repeated provider call after a crash while retaining one transcription row per upload ID. No generic idempotency table or If-Match is needed.
- **H5 — still-open:** A hard-deleted task cannot retain an ordinary required-FK event; proposal ownership is indirect. Delete its events with the task for MVP and scope proposals through their owning conversation/user. Test isolation between both users.
- **H6 — fixed-in-rev2:** The fixed text-first milestone and single model adapter resolve the earlier multi-adapter concern. Align the P0 checklist: move the 20-clip STT comparison after the text gate; voice remains required for MVP.
- **H7 — still-open:** The small set is directional, and held-out nights and Arabic error checks are undefined. Keep 30 text/20 audio; reserve 10 real nights from prompt tuning and specify their overlap with those sets. Manually inspect names, numbers, negation, code-switching and wrong targets; report counts and compare STT with final task errors.
- **H8 — still-open:** “No audio is stored” conflicts with IndexedDB. Disclose temporary on-device buffering; delete after transcript save or discard, with retry on failure. Keep secure cookie, CSRF header, ownership checks, upload limits and redacted logs; verify provider no-training terms before sending data.
- **H9 — still-open:** Tests omit correction/apply races and deletion during analysis. Add cases for stale proposals, applied corrections, ambiguous targets, deletion during analysis and lost-response retry. A late result must not recreate a deleted conversation.

## Medium

- **M1 — fixed-in-rev2:** FR-05 and S15 preserve empathetic conversation without forced tasks. Keep S15 and add one mixed-language clarification follow-up.
- **M2 — fixed-in-rev2:** Registration, multiple production adapters and major infrastructure are deferred. If schedule slips, defer waveform, themes or workload polish; keep voice and the core conversation/review loop.
- **M3 — out-of-scope-for-2-users:** English titles are an accepted default with an Arabic/match setting. Keep editable titles; both users can flag mistranslations.
- **M4 — still-open:** “Don't keep” can retain unresolved drafts indefinitely. Agree a finite expiry, warn before discarding pending items, and never turn expired drafts into tasks. Test expiry and late analysis completion.
- **M5 — still-open:** A deadline time has no defined timezone, and past completion conflicts with the global past-date rule. Use date-only deadlines in MVP; validate past dates according to operation. Keep the 04:00 rule.
- **M6 — still-open:** A 20-second analysis timeout conflicts with the 10-second end-to-end p95 target; local audio recovery is not guaranteed. Treat latency as a measured target, show progress/retry, and describe recovery as best effort. Measure on both phones.
- **M7 — fixed-in-rev2:** Heavy layers, fallback chains and telemetry stack were removed. Keep Go modules and single adapters small.

## Low

- **L1 — out-of-scope-for-2-users:** Themes, haptics and wake lock are polish. Detect optional browser capabilities and test the two actual phones; defer unsupported polish.

**Documentation alignment:** Rev 2 still labels Go open, places the STT bake-off in P0, and allows a future split LLM call. Update those statements when the team next revises the plan. This review changes none of the source documents.
