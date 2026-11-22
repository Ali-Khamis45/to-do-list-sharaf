# Sprint prompts and checklists

Dependency order is one valid topological order; unrelated sprints may proceed independently. Check live status and every listed dependency before starting. Conditional S0.15 runs only if required by S0.14. Human items are checklists. A text-first milestone is fixed: resolve any conflicting STT prerequisites with the humans before starting.

| Sprint | Owner | Reviewer | Depends on |
|---|---|---|---|
| [S0.1 ? Record real nights (held-out raw)](S0.1.md) | Human | Codex | ? |
| [S0.2 ? Provider accounts, API keys & spend alerts](S0.2.md) | Human | Claude Code | ? |
| [S1.1 ? Repo skeleton & toolchain](S1.1.md) | Codex | Claude Code | ? |
| [S1.8 ? Choose host & provision the dev managed Postgres](S1.8.md) | Human | Codex | ? |
| [S0.3 ? Golden case format, loader & 20 dev cases](S0.3.md) | Claude Code | Codex | S1.1 |
| [S0.7 ? Domain: dates, normalization & templates](S0.7.md) | Claude Code | Codex | S1.1 |
| [S1.2 ? Schema migrations & sqlc setup](S1.2.md) | Codex | Claude Code | S1.1 |
| [S1.3 ? HTTP foundation](S1.3.md) | Codex | Claude Code | S1.1 |
| [S5.3 ? Production host, secrets & alerts](S5.3.md) | Human | Codex | S1.8 |
| [S0.4 ? Label & freeze held-out cases](S0.4.md) | Human | Codex | S0.1, S0.3 |
| [S0.5 ? Record 20 clips with reference transcripts](S0.5.md) | Human | Claude Code | S0.1, S0.3 |
| [S0.8 ? Domain: op union, set/clear, references & lifecycle](S0.8.md) | Claude Code | Codex | S0.7 |
| [S1.4 ? Seeded accounts, sessions & settings](S1.4.md) | Codex | Claude Code | S1.2, S1.3 |
| [S0.6 ? STT bake-off spike (throwaway Python)](S0.6.md) | Claude Code | Codex | S0.2, S0.5 |
| [S0.9 ? Domain: evidence locator, dedupe decision & policy](S0.9.md) | Claude Code | Codex | S0.8 |
| [S1.5 ? Task CRUD, lifecycle & events](S1.5.md) | Codex | Claude Code | S1.4, S0.7, S0.8 |
| [S0.10 ? Trigram scorer & §7.5.1 fixture](S0.10.md) | Claude Code | Codex | S1.1, S0.9 |
| [S0.11 ? Turn analyzer adapter, context builder & prompt v1](S0.11.md) | Claude Code | Codex | S0.9, S0.4, S0.2 |
| [S1.6 ? Day API: days, ranges, workload & move-unfinished](S1.6.md) | Codex | Claude Code | S1.5 |
| [S0.12 ? Eval runner (cmd/eval) with split reports](S0.12.md) | Claude Code | Codex | S0.11, S0.10, S0.6 |
| [S1.7 ? OpenAPI root, shell paths & generated types](S1.7.md) | Codex | Claude Code | S1.6 |
| [S2.2 ? AI store queries](S2.2.md) | Claude Code | Codex | S1.2, S0.10 |
| [S0.13 ? Gate run & prompt iteration](S0.13.md) | Claude Code | Codex | S0.12 |
| [S1.9 ? Target-DB verification & P1 exit check](S1.9.md) | Codex | Claude Code | S1.7, S1.8, S0.10 |
| [S2.1 ? AI endpoints in OpenAPI](S2.1.md) | Claude Code | Codex | S1.7 |
| [S3.2 ? Web app skeleton](S3.2.md) | Codex | Claude Code | S1.7 |
| [S0.14 ? Go/no-go and the D-02 & D-03 decisions](S0.14.md) | Human | Claude Code | S0.13 |
| [S4.1 ? App shell, login & RTL foundations](S4.1.md) | Codex | Claude Code | S3.2, S1.4 |
| [S0.15 ? Second LLM adapter (conditional)](S0.15.md) | Claude Code | Codex | S0.14 |
| [S2.3 ? AnalyzeTurn use case](S2.3.md) | Claude Code | Codex | S2.2, S0.14 |
| [S3.1 ? Transcription endpoint & STT adapter](S3.1.md) | Claude Code | Codex | S0.14, S2.1, S2.2 |
| [S4.2 ? Day view](S4.2.md) | Codex | Claude Code | S4.1, S1.6 |
| [S2.4 ? Conversation & message endpoints](S2.4.md) | Claude Code | Codex | S2.3, S2.1, S1.4 |
| [S3.3 ? Recorder](S3.3.md) | Codex | Claude Code | S3.2, S3.1 |
| [S2.5 ? Tray edits & all-or-nothing apply](S2.5.md) | Claude Code | Codex | S2.4, S1.5 |
| [S3.4 ? Device tests on our phones](S3.4.md) | Human | Codex | S3.3 |
| [S4.3 ? Talk sheet: conversation & input](S4.3.md) | Codex | Claude Code | S4.1, S3.3, S2.4 |
| [S2.6 ? Message edit, supersede & race safety](S2.6.md) | Claude Code | Codex | S2.5 |
| [S2.7 ? Conversation delete & privacy cascade](S2.7.md) | Claude Code | Codex | S2.6 |
| [S4.4 ? Plan Tray & apply](S4.4.md) | Codex | Claude Code | S4.3, S2.5, S2.6 |
| [S2.8 ? Scenario API suite, replay mode & P2 exit](S2.8.md) | Claude Code | Codex | S2.7 |
| [S4.5 ? Settings, privacy & PWA](S4.5.md) | Codex | Claude Code | S4.1, S2.7 |
| [S4.6 ? E2E suite & accessibility basics](S4.6.md) | Codex | Claude Code | S4.4, S4.5, S2.8 |
| [S5.1 ? Retention purge, deletion verification & cost report](S5.1.md) | Claude Code | Codex | S2.8 |
| [S4.7 ? Journey walkthrough on our phones](S4.7.md) | Human | Codex | S4.6 |
| [S5.2 ? Error-state polish](S5.2.md) | Codex | Claude Code | S4.6 |
| [S5.4 ? Deploy & backup restore drill](S5.4.md) | Codex | Claude Code | S5.3, S5.2, S5.1 |
| [S5.5 ? Deployed scenario pass & live eval](S5.5.md) | Claude Code | Codex | S5.4 |
| [S5.6 ? Two weeks of nightly use](S5.6.md) | Human | Claude Code | S5.5 |
| [S5.7 ? Fold failures & daily-use metrics](S5.7.md) | Claude Code | Codex | S5.6 |
| [S5.8 ? MVP exit decision](S5.8.md) | Human | Claude Code | S5.7 |
