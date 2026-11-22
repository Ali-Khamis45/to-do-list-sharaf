# Product Requirements: AI Voice Companion & Intelligent To-Do Manager

Status: Approved for planning (rev 2) · Last updated 2026-11-22
Related: [architecture-and-ai-pipeline.md](architecture-and-ai-pipeline.md) · [data-and-api.md](data-and-api.md) · [roadmap-and-testing.md](roadmap-and-testing.md) · [decisions.md](decisions.md)

---

## 1. Context

- **What:** A voice-first, Egyptian-Arabic-capable companion. You talk through your day at night, and the app turns what you actually committed to into tomorrow's checklist.
- **Hard rules:** It must not invent tasks. It must not mistake feelings or wishes for commitments.
- **Real context:** Two users, who are also the two builders. Both are students. Each has about 2 h/day for the project.
- **Example clock:** All scenario examples assume "now" is **Monday 2026-11-23 22:30, Africa/Cairo** unless stated.

## 2. Analysis

### 2.1 Refined core workflow

**Speak → Transcript (editable) → AI reply + Proposed changes → One-tap confirm → Checklist → Check off / revise by talking.**

- The AI never writes to the task list directly. It emits **proposed operations**, which land in a **Plan Tray**. The operations are create, update, reschedule, complete, and cancel.
- Definite tasks are **pre-checked**. Tentative or low-confidence items arrive **unchecked** under "Maybe".
- Changes to existing tasks are always shown as before → after rows.
- One tap applies the tray ("Add 4 to Tuesday"). There is no retyping and nothing silent.
- Opt-in auto-apply is a V1 option once evaluation precision is at least 0.95.

### 2.2 Where we pushed back on the original brief

| Brief said | Concern | Decision |
|---|---|---|
| AI "automatically organizes" tasks | Silent auto-creation with a sometimes-wrong model destroys trust in the list | Pre-checked proposals with one-tap confirm in the MVP. Auto-apply with undo is a V1 opt-in. |
| 6 primary screens | Today Dashboard and Daily Checklist are the same screen. Tomorrow is the same view on another date. | 3 surfaces: Day view (Today/Tomorrow), Talk, and Settings & Privacy |
| "Deferred" status | A task moved to Friday is just pending on Friday | Statuses `pending`, `completed`, and `cancelled`. Deferral is an event plus `defer_count`. A null date means "Later". |
| Heavy Clean Architecture / microservices | Ceremony without payoff at 2 users | A lean modular backend with a pure domain core and thin adapters |
| Streaming STT, TTS, live voice | Large complexity. A nightly monologue tolerates 3–6 s after "stop". | Batch STT. The local waveform from the mic makes it feel live. Streaming and TTS come in V1/V2. |
| LLM computes dates | Weekday arithmetic errors around midnight and DST | The LLM emits a **date expression**. A deterministic resolver produces the date. |
| LLM confidence decides creation | Self-reported confidence is poorly calibrated | A deterministic policy uses commitment class, evidence, reference resolution, and dedupe. Confidence only demotes items to "Maybe". |

### 2.3 Hardest technical challenges, ranked

1. **Egyptian Arabic + English code-switching STT.** Models may translate, transliterate ("دوكر"), or drop English terms. They also confuse names such as أحمد and أمجد. This is measured first, in Phase 0.
2. **Commitment classification.** Telling apart context, task, aspiration, and emotion.
3. **Reference resolution for edits.** "خليها الجمعة", "قصدي محاضرة 4", and "الغي الجيم" need the right target, including answers to the app's own questions from earlier turns.
4. **The night-time date boundary.** At 00:45, "بكرة" means the calendar date that has already started.
5. **Exactly-once creation under retries.**
6. **Mobile web audio.** iOS Safari MediaRecorder, PWA mic permission, screen lock, and never losing a recording.

### 2.4 Assumptions (correctable)

1. **Accounts:** Exactly 2 seeded accounts, with no registration in the MVP.
2. **Capacity:** 2 people × 2 h/day × 6 days/week = **24 team-hours/week** (12 per person). Estimates are ±30%.
3. **UI language:** English, with full RTL/bidi support for Arabic content. A full Arabic UI is V1 (D-05).
4. **Task titles:** English by default, with English technical terms preserved. This is a user setting (D-04).
5. **Replies:** The assistant mirrors the user. Egyptian Arabic in means Egyptian Arabic out.
6. **Calendar:** The day boundary is 04:00 local. The week starts on Saturday.
7. **Hosting:** A single cheap host plus managed Postgres, in an EU region (D-08).
8. **Audio:** Never stored server-side beyond transcription.

---

## 3. Product vision

**Vision:** "Talk it out at night, wake up to a plan you trust."

**Positioning:** Capture happens through natural conversation in your own dialect. The AI acts as a **careful secretary**. It writes down only commitments, asks when unsure, and shows the words it relied on.

**Principles:**
1. **Never invent.** Every AI task cites the user's own words.
2. **Never silent.** Every change is visible and reversible.
3. **Feelings are heard, not filed.**
4. **Tomorrow should be doable.** The app warns on overload.
5. **Private by default.**

**Core KPIs:** nightly sessions per week, tray acceptance rate, edits after apply, and next-day completion rate.

## 4. Personas & core journeys

**Personas:**
- **Primary (real users): the two of us.** CS students in Egypt, juggling lectures, a project, and side learning. We speak Egyptian Arabic with English technical terms and plan at night on a phone.
- **Future personas.** These inform design only, not MVP scope:
  - A freelancer or junior developer who captures client follow-ups by text between meetings.
  - A busy Arabic-first professional who dislikes typing.

**Journeys:**
1. **J1 Nightly brain-dump (primary).** Tap the mic, talk for 1–3 min, and see the transcript. The AI reply and tray appear. Answer one follow-up ("Docker?"), then tap "Add 4 to Tuesday".
2. **J2 Morning check.** The Today view shows progress, and tasks get ticked off.
3. **J3 Quick capture.** "Remind me to call the bank Thursday" becomes one tray item, then you confirm.
4. **J4 Revise by talking.** "بكرة مش هعمل Docker، خليها الجمعة" shows a "Tue → Fri" change in the tray, then you confirm.
5. **J5 Roll over.** Yesterday's unfinished items move to today or Later in one tap, or by saying so.
6. **J6 Privacy.** In Settings, view and delete conversations. Tasks remain, and their source quotes are removed.

## 5. MVP scope & explicit non-goals

**MVP goal:** Prove that an Egyptian Arabic or code-switched conversation reliably becomes the correct next-day checklist, end to end, on our phones.

**In MVP:**
- **Accounts:** 2 seeded accounts with password login. Settings cover timezone, day boundary, task language, and daily capacity.
- **Recording:** Visible recording states, a local waveform, a 5-minute cap, and crash-safe local buffering.
- **STT:** Batch STT through **one** adapter, for the bake-off winner.
- **Input:** Text input on par with voice.
- **Conversation:** History is kept. Editing a sent message re-runs analysis.
- **Analysis:** **One** LLM adapter producing structured ops: create, update, reschedule, complete, cancel, revise/withdraw proposal, and clarify.
- **Deterministic core:** A validator, date resolver, number-aware dedupe, and policy.
- **Plan Tray:** Pre-checked and Maybe items, inline edit, and one-tap apply.
- **Day view:** Today/Tomorrow with any date reachable. It supports checkbox, undo, edit, reschedule, delete, manual add, a workload bar, and "move unfinished".
- **Privacy:** A retention setting, view and delete conversations, and "delete all my data".
- **Evaluation:** An evaluation harness and golden set of 30 text cases and 20 audio clips.
- **Logging:** Structured logs, plus a per-run record of `cost_usd` and latency.
- **PWA:** An installable PWA shell that works online only.

**Non-goals for the MVP:**
- **Voice and AI:** streaming STT, TTS, and long-term memory.
- **Task features:** subtasks and dependencies, recurrence, reminders and push, calendar integration, and time-blocking.
- **Platform:** offline editing, sharing, native apps, Arabic UI chrome, and auto-apply.
- **Infrastructure:** Redis, queues, SignalR, a vector DB, and multi-agent orchestration.
- **V1 hardening:** everything on the V1 hardening list ([roadmap-and-testing.md §4.1](roadmap-and-testing.md)).

**Anti-goals:**
- A cluttered dashboard.
- A chatbot that turns every sentence into a task.
- A therapist persona.
- Stored voice recordings.

## 6. Requirements

### 6.1 Functional (MVP)

| ID | Requirement |
|---|---|
| FR-01 | Record voice with visible state and elapsed time. Auto-stop at 5 min, with a warning at 4:30. |
| FR-02 | A recording stays on the device until its transcript is saved on the server. Failed uploads can be retried. |
| FR-03 | The transcript appears as the user's message. Editing it re-runs analysis and supersedes that message's pending items. |
| FR-04 | Text input runs the same pipeline as voice. |
| FR-05 | The reply is in the user's language, with at most one clarifying question per turn. |
| FR-06 | Definite creates are pre-checked. Tentative items, aspirations with a concrete action, and items with confidence < 0.5 are unchecked "Maybe" items. Low-confidence items also carry a flag. |
| FR-07 | Every proposal shows its resolved date (for example "Tue 24 Nov") and an expandable evidence quote. |
| FR-08 | The user can edit title, date, priority, and estimate in the tray, then apply the selected items with one tap. |
| FR-09 | References to existing tasks yield modify ops, never duplicate creates. |
| FR-10 | Corrections and answers to clarifying questions revise pending tray items rather than adding new ones. |
| FR-11 | Number-aware duplicate detection shows "already on your list" and blocks the insert. |
| FR-12 | The Day view supports checkbox, undo, edit, reschedule, delete, and manual add. |
| FR-13 | The workload bar compares estimates with daily capacity and warns when over. |
| FR-14 | "Move unfinished" moves a past day's pending tasks to today or to Later. |
| FR-15 | Dates resolve in the user's IANA timezone, respecting the day boundary. |
| FR-16 | Settings cover retention, viewing and deleting conversations, and deleting all data. |

### 6.2 Non-functional

| Area | Target |
|---|---|
| Latency | **Stop → tray on a 60 s clip:** p50 ≤ 5 s, p95 ≤ 10 s. **Text → tray:** p50 ≤ 3 s. **Toggle:** ≤ 150 ms, using an optimistic update. |
| Quality | Meets the gate in [roadmap-and-testing.md §2.2](roadmap-and-testing.md). The gate is directional while the golden set has 30 cases. |
| Reliability | **Zero duplicate tasks from retries.** This is a hard requirement and is tested. No recording is ever lost silently. |
| Privacy | **Audio:** never retained. **Transcripts:** kept per the retention setting, 30 days by default, and never written to logs. **Providers:** only under no-training API terms. |
| Security | **MVP controls:** per-user data isolation (tested), an HttpOnly session cookie, upload limits, and a spend cap. **OWASP ASVS:** a guideline in the MVP and a release gate in V1 hardening. |
| Accessibility | **MVP:** a labelled mic control, reduced-motion support, and an RTL-correct layout. **WCAG 2.2 AA:** a guideline in the MVP and an audit gate in V1 hardening. |
| Mobile | Installable PWA on Chrome for Android and iOS Safari 17+. |
| Cost | **Tracking:** `cost_usd` recorded per analysis and per transcription. **Controls:** a monthly spend alert at the provider and a hard per-user monthly audio-minute cap. |
| Portability | STT and LLM sit behind ports, and any swap is re-evaluated. |

---

## 7. Screens & UX plan

### 7.1 Information architecture

There are three surfaces: the Day view, Talk, and Settings & Privacy.

```
┌───────────────────────────────┐
│  Today ▸ Tomorrow ▸ 📅         │  ← Day view
│  Mon 23 Nov · 3 of 5 done ███░ │
│  ☐ Finish AI project section ⓗ │
│  ☑ Reply to Ahmed              │
│  ☐ Review OS lecture 1   ~1h   │
│  + Add task                    │
│  ↺ 2 unfinished from yesterday │
├───────────────────────────────┤
│   Today      ( 🎙 )     ⚙      │  ← central mic opens Talk
└───────────────────────────────┘
```

- **Day view:** One component, parameterized by date. Before 18:00 the app opens on Today. After 18:00 it opens on Tomorrow, because the evening is planning time.
- **Talk:** A full-screen sheet opened from the central mic. It holds the conversation, the recorder, and the Plan Tray.
- **Settings & Privacy:** Profile, task language, day boundary, capacity, retention, the conversations list with delete, and "delete all my data".

### 7.2 Talk screen

```
┌───────────────────────────────┐
│ ✕  Tonight's check-in          │
│  You (voice) ✎  "بص أنا…"      │
│  Companion  "…Docker؟"         │
│ ┌ Plan for Tue 24 Nov ────────┐│
│ │☑ Finish AI project section  ││
│ │☑ Review OS lecture 1  ~1h   ││
│ │☑ Review OS lecture 2  ~1h   ││
│ │☑ Reply to Ahmed       ~10m  ││
│ │ Maybe                       ││
│ │☐ Start learning Docker      ││
│ │ ~4h10m of 6h  ✓ fits        ││
│ │ [ Add 4 to Tuesday ]        ││
│ └─────────────────────────────┘│
│   ∿∿∿∿∿  (  ●  )  ⌨            │
└───────────────────────────────┘
```

**Recorder state machine.** Each state has distinct visuals and screen-reader text.
- **Main flow:** `idle → requesting_permission → recording → stopping → uploading → transcribing → analyzing → reviewing`.
- **Error states:**
  - `error_permission`
  - `error_upload`, where the audio is kept and can be retried
  - `error_analysis`, which can be retried

**Recording details:**
- **Input:** Tap to toggle rather than hold-to-talk, because monologues are long.
- **Feedback:** A waveform driven by the WebAudio AnalyserNode. A Screen Wake Lock is held while recording. Haptic feedback plays on start and stop.
- **Crash safety:** MediaRecorder writes 1 s chunks to IndexedDB, so a crash never loses audio.
- **Formats:** `audio/webm;codecs=opus`, falling back to `audio/mp4` for iOS.

**Tray interactions:**
- **Edit:** Tap a row to edit it inline. The date chip offers Today, Tomorrow, a weekday list, and a picker.
- **Reject:** Tap ✕ or swipe the row away.
- **Change rows:** Edits to existing tasks show a before → after diff, such as "Tue → Fri".
- **Notices:** A duplicate notice row reads "Already on Tuesday". Low-confidence items carry a badge.
- **Clarification:** Answering the question updates the item in place. For example, the Docker row moves from Maybe into the pre-checked list, with its new date highlighted.
- **Continuing:** If you keep talking, the tray updates in place and changed fields flash.
- **Pending banner:** Items not yet applied show as a banner on the Day view: "4 suggestions waiting".

### 7.3 Visual direction

- **Look:** Calm and premium. A warm neutral palette, one accent color, and generous spacing.
- **Typography:** A type pairing that renders Arabic well, for example IBM Plex Sans Arabic with Inter.
- **Motion:** Subtle springs for checkbox completion and tray items sliding in. `prefers-reduced-motion` is respected.
- **Themes:** Dark and light. Dark is likely the default at night.
- **Text direction:** `dir="auto"` on every user- or AI-generated string. Logical CSS properties are used throughout, so an Arabic UI in V1 is a translation job, not a re-layout.

### 7.4 Frontend structure

- **Routes:** `app/(app)/{day/[date],talk,settings}` and `app/login`.
- **Features:** `features/recorder` covers the state machine, MediaRecorder adapter, IndexedDB buffer, and waveform. `features/conversation` covers messages and the tray. `features/tasks` covers the Day view, task row, and edit sheet.
- **API layer:** `lib/api` holds the client generated from `openapi.yaml` and the TanStack Query hooks. Checkbox toggles are optimistic and roll back on error.
- **Tests:** Vitest with Testing Library for components, and Playwright with fake media streams for E2E.
