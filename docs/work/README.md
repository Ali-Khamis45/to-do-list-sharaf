# docs/work: how this folder works

This folder breaks the MVP (P0–P5) into sprints for three kinds of owner:
- **Claude Code**, which owns the AI track: domain core, prompt, eval, and the analysis pipeline.
- **Codex**, which owns the shell track: DB, auth, task API, recorder, and frontend.
- **Human**: the two of us, for things only we can do. That means recording clips, writing real-night cases, deciding, and testing on our phones.

The two humans work only when free, so everything is planned **by scope, not dates**.

## Reading order
1. This README: the rules.
2. [STATUS.md](STATUS.md): what's happening now.
3. [roadmap.md](roadmap.md): every sprint in one table, plus the dependency graph.
4. The phase file ([phases/](phases/)), then the milestone ([milestones/](milestones/)), then your sprint ([sprints/](sprints/)).
5. The doc sections listed under **Read first** in your sprint. Section numbers match [../plan.md](../plan.md), which is the source for the four topic docs.

## Layout
```
docs/work/
  README.md        rules (this file)
  STATUS.md        live status, one line per sprint (the only shared file)
  roadmap.md       all phases → milestones → sprints, dependency graph
  phases/P0..P5.md goal, scope, out of scope, milestones, exit gate
  milestones/M<phase>.<n>-<slug>.md   goal, sprints, acceptance, demo
  sprints/S<phase>.<n>-<slug>.md      the unit of work
```

## Status legend
| Status | Meaning |
|---|---|
| `todo` | Not started. It can start when every depends-on sprint is `done`. |
| `in-progress` | The owner is working on it. Only one agent works on a sprint at a time. |
| `review` | The owner finished, tests are green, and the handoff notes are filled in. It is waiting for the reviewer. |
| `done` | The reviewer checked the acceptance criteria and tests. A conditional sprint that wasn't needed is closed as `done` with the note "not needed". |
| `blocked` | It can't proceed, and the reason is in its handoff notes or STATUS line. Conditional sprints start as `blocked`. |

## Workflow
1. **Pick** a `todo` sprint you own whose depends-on sprints are all `done`. Set its line in STATUS.md to `in-progress`.
2. **Read** the "Read first" sections before you touch anything.
3. **Work** only in the files and directories listed for the sprint. If you need anything else, stop and write the request in the handoff notes.
4. **Finish** with every test green (`make test`, plus the sprint's own targets) and the acceptance criteria checked. Fill in **Handoff notes**, then set STATUS to `review → <reviewer>`.
5. **Review.** The reviewer checks each acceptance criterion and runs the tests, then adds a "Review" sub-section to the handoff notes.
   - If everything passes, the reviewer sets the sprint to `done`.
   - If not, the reviewer sets it back to `in-progress` with a short note, and the owner fixes it.
6. **Commits are made by humans only.** Agents never commit. A human reads the diff after review and commits.

**Reviewers:** Every sprint is reviewed by the other agent. Human sprints are reviewed by the agent that consumes their output. The reviewer checks format and completeness only, never held-out content.

## File ownership

| Area | Owner | Notes |
|---|---|---|
| `docs/product-requirements.md`, `docs/architecture-and-ai-pipeline.md`, `docs/data-and-api.md`, `docs/roadmap-and-testing.md`, `docs/decisions.md`, `docs/plan.md`, `docs/work/**` | Claude Code | STATUS.md is shared (see below) |
| `docs/prompts/**` | Codex | Claude Code never creates or edits anything here |
| `internal/domain/**`, `internal/ai/**`, `internal/eval/**`, `cmd/eval/**`, `test/scenarios/**` | Claude Code | Codex may *import* `domain/normalize`, `domain/lifecycle`, `domain/civil` |
| `prompts/**` (repo root, the turn-analyzer prompt and schema) | Claude Code | Not the same as `docs/prompts/` |
| `eval/golden/README.md`, `eval/golden/schema/**`, `eval/golden/dev/**`, `eval/stt/**`, `eval/reports/**`, `spikes/stt-bakeoff/**` | Claude Code | |
| `eval/golden/heldout/**`, `eval/clips/**`, `eval/feedback/**`, `.env` | Human | See held-out confidentiality |
| `internal/app/{analyze,proposals,conversations,transcribe,purge}/**` | Claude Code | |
| `internal/app/{tasks,auth,settings}/**` | Codex | Claude Code calls the `tasks` service from apply (read-only use) |
| `internal/httpapi/{routes_ai,conversations,messages,proposals,transcriptions,ai_spec_test}*.go`, `cmd/api/jobs_ai.go` | Claude Code | Codex creates `routes_ai.go` and `jobs_ai.go` as empty hooks in S1.3 |
| All other `internal/httpapi/**`, `internal/config/**`, `cmd/api/**` | Codex | |
| `internal/store/{conversations,messages,transcriptions,analysis,proposals,dedupe,purge}*.go` and the matching `db/queries/*.sql` (plus `cost.sql`, `metrics.sql`) | Claude Code | |
| `internal/store/{db,users,sessions,tasks}*.go`, `db/queries/{users,tasks,task_events}.sql`, `sqlc.yaml` | Codex | |
| `db/migrations/**` | Codex | Claude Code asks for schema changes through the handoff notes |
| `api/openapi.yaml`, `api/paths/{auth,me,tasks,days}.yaml` | Codex | |
| `api/paths/{conversations,messages,proposals,transcriptions}.yaml` | Claude Code | |
| `web/**`, `deploy/**`, `Dockerfile*`, `docker-compose*.yml`, `.github/**`, `Makefile`, `mk/shell.mk`, `.env.example`, `.gitignore` | Codex | |
| `mk/ai.mk` | Claude Code | Created empty by Codex in S1.1 |

### Shared-file rules
- **`go.mod` / `go.sum`:**
  - Codex creates them in S1.1.
  - Either agent may change them, but only with `go get` or `go mod tidy` inside its own sprint, and must list the new dependency in its handoff notes.
- **Generated code** (`internal/store/sqlc/**`, `internal/httpapi/gen/**`, `api/gen/**`):
  - Never hand-edit it.
  - Regenerate it with `make sqlc` or `make api-gen` inside your sprint. Whoever finishes last regenerates.
- **STATUS.md:** Change one line at a time, and only the line for your own sprint.
- **Your sprint file:** Edit only its Handoff notes section. Never edit another owner's sprint file, except to add the "Review" sub-section when you are its reviewer.

## Held-out confidentiality
- **The held-out split must never be read by the prompt author.** It lives in `eval/golden/heldout/**` and the held-out clips in `eval/clips/heldout/**`.
- **Claude Code** never opens, prints, or quotes those files. Its tools read them only programmatically, and reports show counts, metrics, and pass/fail only.
- **Codex** may validate their format when it is the reviewer. It never pastes their content into a shared doc.
- **Contamination:** If anyone needs to look at a held-out case to debug it, that case moves to dev and the humans record a replacement real night.
- **Privacy:** Confirm the repo is private before committing held-out cases or any audio.

## Human sprints
- **Who:** The two of us work them. Mark progress in STATUS.md yourself, or ask an agent to do it.
- **Order:** They come first wherever they block other work. Recording the real nights (S0.1) and the clips (S0.5), labeling the held-out split (S0.4), and the provider keys (S0.2) all gate the P0 eval.
- **Decisions:** Decision sprints (S0.14, S5.8) end with Claude Code recording the outcome in [decisions.md](../decisions.md).

## Conditional sprints
S0.15 (a second LLM adapter) starts as `blocked`. It becomes `todo` only if S0.14 says the gate was missed. Otherwise it is closed as `done` with the note "not needed".

## Failure log (P5)
During daily use (S5.6), write one entry per failure in `eval/feedback/log.md`:

```
## 2026-12-03 · speaker
- said: <what you said, or "private">
- expected: <what you wanted>
- got: <what the app did>
```

## Where to start
- [S0.1](sprints/S0.1-real-nights.md) (Human): record real nights. The prompt can't be written until these are labeled and frozen.
- [S1.1](sprints/S1.1-repo-skeleton.md) (Codex): repo skeleton. Every Go sprint depends on it.
- [S0.2](sprints/S0.2-provider-keys.md) (Human): provider accounts and keys.

After those, Claude Code starts [S0.3](sprints/S0.3-golden-format.md) (golden format) and [S0.7](sprints/S0.7-domain-dates-normalize.md) (domain dates), and Codex starts [S1.2](sprints/S1.2-schema-migrations.md) and [S1.3](sprints/S1.3-http-foundation.md).

COMPLETE
