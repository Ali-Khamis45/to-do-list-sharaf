# Product Requirements

Status: Planning, rev 3 · Last updated 2026-11-22 · Generated from [plan.md](plan.md); edit plan.md first, then re-split.

Section numbers match `plan.md` (rev 3, the source). Where each section lives: Part A, §1–§4, §10 → product-requirements.md · §5–§7, §11 → architecture-and-ai-pipeline.md · §8–§9 → data-and-api.md · §12–§14, §16 → roadmap-and-testing.md · §15 → decisions.md

---

## Context

- **What it does:** It is a voice-first, Egyptian-Arabic-capable companion. You talk through your day at night, and the app turns what you actually committed to into tomorrow's checklist.
- **Hard rules:** It must not invent tasks, and it must not mistake feelings or wishes for commitments.
- **Who it's for:** Two users, who are also the two builders. Both of us are students and work on it when we're free, about 2 h/day each on average.
- **Stage:** This is still planning. No application code, scaffolding, or dependency installs exist yet.
- **Example clock:** Scenarios assume "now" is **Monday 2026-11-23 22:30, Africa/Cairo** unless stated.

---

## Part A — Analysis before planning

### A.1 Refined core workflow

**Speak → Transcript (editable) → AI reply + suggested changes → One-tap confirm → Checklist → Check off / revise by talking.**

- The AI never writes to the task list. It emits **proposed operations**, which land in a **Plan Tray**.
- **Before apply:** The reply only describes *suggestions*. A server-templated tray summary reads, for example, "اقترحتلك ٤ حاجات، راجعها وأكد".
- **After apply:** A server-templated acknowledgement is built from what was actually committed.
- **Pre-checking:** Definite tasks are pre-checked. Tentative and low-confidence items are unchecked under "Maybe". Changes to existing tasks show as before → after rows.
- **Auto-apply:** An opt-in auto-apply is a V1 option once precision is at least 0.95.

### A.2 Where we pushed back on the brief

| Brief said | Concern | Decision |
|---|---|---|
| AI "automatically organizes" tasks | Silent auto-creation with a sometimes-wrong model destroys trust | Suggestions with one-tap confirm. V1 opt-in auto-apply. |
| 6 primary screens | Today Dashboard and Daily Checklist are the same screen, and Tomorrow is the same view | 3 surfaces: Day view, Talk, Settings & Privacy |
| "Deferred" status | A task moved to Friday is pending on Friday | `pending` / `completed` / `cancelled`, a `rescheduled` event, `defer_count`, null date = Later |
| Heavy Clean Architecture / microservices | Ceremony without payoff at 2 users | A lean Go modular monolith with a pure domain package |
| Streaming STT, TTS, live voice | Large complexity for a nightly monologue | Batch STT and a local waveform. Streaming and TTS in V1/V2. |
| LLM computes dates | Weekday arithmetic errors at midnight and DST | The LLM emits a date expression, and a deterministic resolver turns it into a date |
| LLM confidence decides creation | Poorly calibrated | Policy decides. Confidence only demotes an item to Maybe. |
| LLM reports what it saved | A generative reply can't know a later transaction's result | Suggestion language only. Acknowledgement templated from the apply response. |

### A.3 Hardest technical challenges, ranked

1. **Egyptian Arabic + English code-switching STT.** It can translate, transliterate ("دوكر"), drop terms, or confuse names (أحمد/أمجد).
2. **Commitment classification.** The model has to tell context, task, aspiration, and emotion apart.
3. **Reference resolution.** "خليها الجمعة", "قصدي محاضرة 4", "الغي الجيم", and answers to the app's own questions.
4. **Night-time date boundary.** At 00:45 Tuesday, "بكرة" = Tuesday under the 04:00 rule.
5. **Exactly-once creation under retries.** It must also hold when an apply races a message edit.
6. **Mobile web audio.** iOS Safari MediaRecorder, PWA mic permission, and screen lock.

### A.4 Assumptions

1. **Users:** There are exactly 2 seeded accounts and no registration.
2. **Capacity:** About 2 h/day each, but we work when we're free. **Hour estimates are the real estimate. Calendar weeks are rough.**
3. **Language:** The UI is English with full RTL/bidi for Arabic content. Arabic UI is V1 (D-05).
4. **Task titles:** Titles are **in the language the user spoke**, keeping English technical terms as spoken (D-04). Titles stay editable.
5. **Replies:** The assistant mirrors the user's language. Egyptian Arabic in means Egyptian Arabic out.
6. **Days and weeks:** The day boundary is 04:00 local. Weeks start Saturday. We are the users, so there is no external validation of the boundary convention.
7. **Hosting:** A single cheap host plus managed Postgres, in an EU region (D-08).
8. **Audio:** Never stored server-side beyond transcription.

---

## 1. Product vision

**Vision:** "Talk it out at night, wake up to a plan you trust."

**Positioning:** You capture tasks by talking naturally in your own dialect. The AI acts as a **careful secretary**. It suggests only commitments, asks when unsure, and shows the words it relied on.

**Principles:**
1. **Never invent.** Every AI suggestion cites the user's own words, located by the server.
2. **Never silent, never overclaiming.** Every change is visible. The AI never says something was saved unless the server committed it.
3. **Feelings are heard, not filed.**
4. **Tomorrow should be doable.** The app warns on overload.
5. **Private by default.**

**Core KPIs:** nightly sessions per week, tray acceptance rate, edits after apply, next-day completion rate.

---

## 2. Personas & core journeys

**Personas:**
- **Primary (real users), the two of us:** CS students in Egypt with lectures, a project, and side learning. We speak Egyptian Arabic with English technical terms and plan at night on a phone.
- **Future (informs design only):** A freelancer or junior developer who captures follow-ups by text, and an Arabic-first professional who dislikes typing.

**Journeys:**
1. **J1 Nightly brain-dump.** Tap the mic and talk for 1–3 minutes. The transcript appears. The reply comes with a tray showing "اقترحتلك ٤ حاجات، راجعها وأكد". Answer the Docker question, tap apply, and see the ack "تمام، ضفت ٤ حاجات لبكرة".
2. **J2 Morning check.** The Today view shows progress, and you tick tasks off.
3. **J3 Quick capture.** "فكرني أكلم البنك الخميس" becomes one suggestion, which you confirm.
4. **J4 Revise by talking.** "بكرة مش هعمل Docker، خليها الجمعة" shows a "Tue → Fri" change row, which you confirm.
5. **J5 Roll over.** Yesterday's unfinished tasks move to today or Later.
6. **J6 Privacy.** Deleting a conversation also deletes its transcriptions and evidence. The tasks remain.

---

## 3. MVP scope & explicit non-goals

**MVP goal:** Prove that an Egyptian Arabic or code-switched conversation reliably becomes the correct next-day checklist, end to end, on our phones.

**In MVP:**
- **Accounts:** 2 seeded accounts with password login. Settings cover timezone, day boundary, task-title language, and daily capacity.
- **Recording:** Visible states, a local waveform, a 5-minute cap, and crash-safe local buffering.
- **Input:** Batch STT through **one** adapter (the bake-off winner). Text input is on par with voice.
- **Conversation:** Full history. Editing a sent message re-runs analysis.
- **Turn analyzer:** **One** LLM adapter. It returns a reply, a nullable **clarification object**, and typed ops: `create`, `update_task`, `reschedule_task`, `complete_task`, `cancel_task`, `revise_proposal`, `withdraw_proposal`.
- **Deterministic core:** The validator, evidence locator, date resolver, number-aware dedupe, and policy.
- **Plan Tray:** A templated summary, pre-checked and Maybe items, inline edit, and **all-or-nothing** one-tap apply with a templated acknowledgement.
- **Day view:** Today, Tomorrow, or any date. Checkbox, undo, edit, reschedule, delete, manual add, a workload bar, and "move unfinished".
- **Privacy (full MVP scope):** A retention setting. Deleting a conversation also deletes its transcriptions, runs, and proposal evidence. "Delete all my data". No `raw_output` stored.
- **Quality and ops:** An evaluation harness with dev and held-out splits. Structured logs, plus per-run `cost_usd` and latency.
- **Shell:** An installable PWA shell, online only.

**Non-goals:** Everything below is out of the MVP.
- **Voice:** Streaming STT and TTS.
- **Memory:** Long-term memory.
- **Task features:** Subtasks, recurrence, reminders, calendar sync, and time-blocking.
- **Platform:** Offline editing, sharing, native apps, and Arabic UI chrome.
- **Automation:** Auto-apply.
- **Infrastructure:** Redis, queues, SignalR, a vector DB, and agents.
- **Hardening:** Everything on the V1 hardening list (§13.1).

**Anti-goals:** A cluttered dashboard. A chatbot that turns every sentence into a task. A therapist persona. Stored voice recordings.

---

## 4. Requirements

### 4.1 Functional (MVP)

| ID | Requirement |
|---|---|
| FR-01 | **Recording:** Record voice with visible state and elapsed time. Auto-stop at 5 min, with a warning at 4:30. |
| FR-02 | **Local buffering:** A recording stays on the device until its transcript is saved. Failed uploads can be retried. |
| FR-03 | **Editable transcript:** The transcript appears as the user's message. Editing it re-runs analysis and supersedes that message's still-pending items. Apply and edit race safely (FR-18). |
| FR-04 | **Text input:** Text runs the same pipeline as voice. |
| FR-05 | **Reply rules:** The reply is in the user's language. **Before apply it describes suggestions only.** It never states counts and never claims anything was saved. It asks at most one clarifying question per turn, through the clarification object. |
| FR-06 | **Pre-check and Maybe:** Definite creates are pre-checked. Tentative items, concrete aspirations, and confidence < 0.5 go to unchecked "Maybe", with a low-confidence flag where it applies. |
| FR-07 | **Evidence display:** Every suggestion shows its resolved date and an expandable evidence quote. The **server cuts the quote from the message text using offsets it computed itself**. Between 00:00 and the day boundary, relative-date items show a line such as "بكرة = التلات 24 نوفمبر". |
| FR-08 | **Tray editing:** Items can be edited in the tray, and each edit bumps the item's revision. Apply is **all-or-nothing**. Any conflict commits nothing and shows item-level conflicts. |
| FR-09 | **Existing tasks:** References to existing tasks become modify ops, never duplicate creates. |
| FR-10 | **Corrections:** Corrections and clarification answers revise pending items rather than adding new ones. |
| FR-11 | **Dedupe:** Number-aware dedupe (§7.5.1) shows "already on your list" and blocks the insert. Re-proposals of rejected or withdrawn items from earlier messages are dropped. |
| FR-12 | **Day view actions:** Checkbox, undo, edit, reschedule, delete, manual add. |
| FR-13 | **Workload:** A workload bar compares estimates with daily capacity. |
| FR-14 | **Move unfinished:** A past day's pending tasks move to today or Later. |
| FR-15 | **Date resolution:** Dates resolve in the user's IANA timezone with the day boundary. |
| FR-16 | **Privacy settings:** A retention setting. Deleting a conversation removes its messages, transcriptions, runs, proposal items, and evidence. "Delete all my data". |
| FR-17 | **Acknowledgement:** After apply, the assistant acknowledgement is a **server template** built from the apply response, never from LLM output. |
| FR-18 | **Apply vs edit race:** When apply and message-edit act on the same proposal, exactly one wins inside its transaction. The loser gets a 409 or a no-op result. |

### 4.2 Non-functional

| Area | Target |
|---|---|
| Latency | Stop on a 60 s clip until the tray shows: p50 ≤ 5 s, p95 ≤ 10 s. Text to tray: p50 ≤ 3 s. Toggle: ≤ 150 ms with an optimistic update. |
| Quality | The §12.2 gate on **both** the dev and held-out splits. It is directional at this size. |
| Reliability | Zero duplicate tasks from retries (hard requirement). Zero partial applies. No recording lost silently. |
| Privacy | No audio retained. No `raw_output`. Transcripts kept per the retention setting, 30 days by default. No transcripts in logs. Providers under no-training API terms. |
| Security | Per-user isolation, tested. HttpOnly session cookie. Upload limits. A spend cap. ASVS is a guideline in the MVP and a V1-hardening gate. |
| Accessibility | A labelled mic, reduced motion, RTL-correct layout. WCAG AA is a guideline in the MVP and a V1-hardening gate. |
| Mobile | Installable PWA on Chrome Android and iOS Safari 17+. |
| Cost | `cost_usd` per call, a provider spend alert, and a per-user monthly audio-minute cap. |
| Portability | STT and LLM sit behind interfaces. Re-evaluate on any swap. |

---

## 10. Screens & UX plan

```
┌───────────────────────────────┐
│  Today ▸ Tomorrow ▸ 📅         │  ← Day view
│  Mon 23 Nov · 3 of 5 done ███░ │
│  ☐ أخلص جزء مشروع الـ AI   ⓗ   │
│  ☑ أرد على أحمد                │
│  ☐ أراجع محاضرة 1 OS    ~1h    │
│  + Add task                    │
│  ↺ 2 unfinished from yesterday │
├───────────────────────────────┤
│   Today      ( 🎙 )     ⚙      │  ← central mic opens Talk
└───────────────────────────────┘
```

```
┌───────────────────────────────┐
│ ✕  Tonight's check-in          │
│  You (voice) ✎  "بص أنا…"      │
│  Companion  "باين يومك كان…"   │
│ ┌ Tue 24 Nov ─────────────────┐│
│ │ اقترحتلك ٤ حاجات، راجعها وأكد ││
│ │☑ أخلص جزء مشروع الـ AI  ~2h ││
│ │☑ أراجع محاضرة 1 OS     ~1h  ││
│ │☑ أراجع محاضرة 2 OS     ~1h  ││
│ │☑ أرد على أحمد بخصوص المشروع ││
│ │ Maybe                       ││
│ │☐ أبدأ أتعلم Docker          ││
│ │ ? Docker: بكرة ولا يوم تاني؟ ││
│ │ ~4h10m of 6h  ✓ fits        ││
│ │ [ Confirm 4 ]               ││
│ └─────────────────────────────┘│
│   ∿∿∿∿∿  (  ●  )  ⌨            │
└───────────────────────────────┘
```

- **Day view:** One component, parameterized by date. Before 18:00 it opens on Today, after 18:00 on Tomorrow.
- **Talk:** A full-screen sheet with three parts.
  - **Order:** The LLM reply comes first. Under it is the tray card with the templated summary, the items, and the clarification question in its own row.
  - **Apply:** After apply, the templated acknowledgement appears as an assistant bubble.
  - **On 409:** The tray re-fetches, highlights the conflicted items, and shows "في حاجة اتغيرت، راجع تاني". Nothing is saved.
- **Night line:** Between 00:00 and 04:00, relative-date rows show "بكرة = التلات 24 نوفمبر" under the date chip.
- **Recorder state machine:** `idle → requesting_permission → recording → stopping → uploading → transcribing → analyzing → reviewing`, plus `error_permission`, `error_upload` (audio kept), and `error_analysis` (retry).
- **Recording:** Tap to toggle. AnalyserNode waveform. Wake Lock. Haptics. 1 s chunks to IndexedDB. webm/opus with an mp4 fallback.
- **Tray rows:** Inline edit (sends `revision`), reject with ✕ or swipe, before → after change rows, a duplicate notice, a low-confidence badge, and in-place updates as the user keeps talking.
- **Visuals:** A warm neutral palette with one accent. Arabic-capable type, for example IBM Plex Sans Arabic + Inter. Subtle springs. Reduced-motion support. Dark and light themes.
- **Text direction:** `dir="auto"` on every generated string, since titles are mostly Arabic with Latin terms. Logical CSS properties throughout.
- **Code structure:** `web/app/(app)/{day/[date],talk,settings}`, `web/app/login`, `web/features/{recorder,conversation,tasks}`, `web/lib/api`. Tests use Vitest + Testing Library, and Playwright with fake media.
