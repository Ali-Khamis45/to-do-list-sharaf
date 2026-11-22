# STATUS

This is the only file both agents update during implementation. **Change one line at a time, and only your own sprint's line.** Humans may update Human sprint lines, or ask an agent to.

Format: `| ID | status | owner | title | note |`. Keep the note short, for example `review → Codex` or `blocked: waiting for S1.8`.

| Status | Meaning |
|---|---|
| `todo` | Not started. It can start when every depends-on sprint is `done`. |
| `in-progress` | The owner is working on it. Only one agent works on a sprint at a time. |
| `review` | The owner finished, tests are green, and the handoff notes are filled in. It is waiting for the reviewer. |
| `done` | The reviewer checked the acceptance criteria and tests. A conditional sprint that wasn't needed is closed as `done` with the note "not needed". |
| `blocked` | It can't proceed, and the reason is in its handoff notes or STATUS line. Conditional sprints start as `blocked`. |

| ID | Status | Owner | Title | Note |
|---|---|---|---|---|
| S0.1 | todo | Human | Record real nights (held-out raw) |  |
| S0.2 | todo | Human | Provider accounts, API keys & spend alerts |  |
| S0.3 | todo | Claude Code | Golden case format, loader & 20 dev cases |  |
| S0.4 | todo | Human | Label & freeze held-out cases |  |
| S0.5 | todo | Human | Record 20 clips with reference transcripts |  |
| S0.7 | todo | Claude Code | Domain: dates, normalization & templates |  |
| S0.8 | todo | Claude Code | Domain: op union, set/clear, references & lifecycle |  |
| S0.9 | todo | Claude Code | Domain: evidence locator, dedupe decision & policy |  |
| S0.10 | todo | Claude Code | Trigram scorer & §7.5.1 fixture |  |
| S0.6 | todo | Claude Code | STT bake-off spike (throwaway Python) |  |
| S0.11 | todo | Claude Code | Turn analyzer adapter, context builder & prompt v1 |  |
| S0.12 | todo | Claude Code | Eval runner (cmd/eval) with split reports |  |
| S0.13 | todo | Claude Code | Gate run & prompt iteration |  |
| S0.14 | todo | Human | Go/no-go and the D-02 & D-03 decisions |  |
| S0.15 | blocked | Claude Code | Second LLM adapter (conditional) | conditional on S0.14 |
| S1.1 | todo | Codex | Repo skeleton & toolchain |  |
| S1.2 | todo | Codex | Schema migrations & sqlc setup |  |
| S1.3 | todo | Codex | HTTP foundation |  |
| S1.4 | todo | Codex | Seeded accounts, sessions & settings |  |
| S1.5 | todo | Codex | Task CRUD, lifecycle & events |  |
| S1.6 | todo | Codex | Day API: days, ranges, workload & move-unfinished |  |
| S1.7 | todo | Codex | OpenAPI root, shell paths & generated types |  |
| S1.8 | todo | Human | Choose host & provision the dev managed Postgres |  |
| S1.9 | todo | Codex | Target-DB verification & P1 exit check |  |
| S2.1 | todo | Claude Code | AI endpoints in OpenAPI |  |
| S2.2 | todo | Claude Code | AI store queries |  |
| S2.3 | todo | Claude Code | AnalyzeTurn use case |  |
| S2.4 | todo | Claude Code | Conversation & message endpoints |  |
| S2.5 | todo | Claude Code | Tray edits & all-or-nothing apply |  |
| S2.6 | todo | Claude Code | Message edit, supersede & race safety |  |
| S2.7 | todo | Claude Code | Conversation delete & privacy cascade |  |
| S2.8 | todo | Claude Code | Scenario API suite, replay mode & P2 exit |  |
| S3.1 | todo | Claude Code | Transcription endpoint & STT adapter |  |
| S3.2 | todo | Codex | Web app skeleton |  |
| S3.3 | todo | Codex | Recorder |  |
| S3.4 | todo | Human | Device tests on our phones |  |
| S4.1 | todo | Codex | App shell, login & RTL foundations |  |
| S4.2 | todo | Codex | Day view |  |
| S4.3 | todo | Codex | Talk sheet: conversation & input |  |
| S4.4 | todo | Codex | Plan Tray & apply |  |
| S4.5 | todo | Codex | Settings, privacy & PWA |  |
| S4.6 | todo | Codex | E2E suite & accessibility basics |  |
| S4.7 | todo | Human | Journey walkthrough on our phones |  |
| S5.1 | todo | Claude Code | Retention purge, deletion verification & cost report |  |
| S5.2 | todo | Codex | Error-state polish |  |
| S5.3 | todo | Human | Production host, secrets & alerts |  |
| S5.4 | todo | Codex | Deploy & backup restore drill |  |
| S5.5 | todo | Claude Code | Deployed scenario pass & live eval |  |
| S5.6 | todo | Human | Two weeks of nightly use |  |
| S5.7 | todo | Claude Code | Fold failures & daily-use metrics |  |
| S5.8 | todo | Human | MVP exit decision |  |
