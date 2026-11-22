# Working with Codex and Claude Code

Each sprint has one **owner agent** and one **reviewer agent**, identified in its sprint document. The owner implements and fixes the sprint; the reviewer checks the result without changing its files. The owner edits only the files listed in that sprint document. Include `docs/work/STATUS.md` in the allowed file list when the owner is expected to update its status line; that permission covers only the owner's own line.

Before starting, read the sprint document, its **Read first** references, and `docs/work/STATUS.md`. Do not start a sprint until every sprint in its **depends-on** list is marked done. If a dependency is missing or unclear, stop and ask the humans.

## Cycle

1. **Kickoff:** The owner checks dependencies and ownership, reads the required documents, and briefly states the plan.
2. **Implement:** The owner works within the sprint's file list and acceptance criteria.
3. **Self-check:** The owner runs the specified checks, records results and limitations, and checks the changed files against the allowed list.
4. **Handoff:** The owner fills the sprint's Handoff notes and marks its own status line ready for review.
5. **Review:** The reviewer reads the sprint, its references, handoff, and changes; checks the acceptance criteria; and reports findings by priority without making code changes.
6. **Fix:** The owner addresses the findings, repeats relevant checks, and hands the sprint back for review. Repeat review and fix as needed.
7. **Done:** After the reviewer reports that acceptance criteria are met and no blocking findings remain, the owner marks its own status line done. The humans decide whether and when to merge.

Use the status labels and format defined in `docs/work/STATUS.md`. Only the owner edits its own sprint's line; the reviewer communicates review results to the owner instead of editing status. If the status file does not define the needed transition, ask the humans rather than inventing one. A sprint document should explicitly include its own handoff notes and status line in its permitted files if the owner must edit them.

## Blocks, decisions, and Git

If blocked, stop dependent work, record the blocker and next action in the sprint's handoff notes and, if available, its status line. Tell the humans what is needed. If a requirement is ambiguous or needs a product or architecture decision, present the concrete question and wait for a human decision; do not guess or start a dependent sprint. Continue independent work only if it is authorized and within the sprint's file list.

Use one branch per sprint, named `sprint/<ID>`. The humans merge branches. Do not merge on an agent's own initiative. Follow any more restrictive instructions from the humans about Git operations.

The [templates](templates/) are ready to paste after replacing their placeholders. Sprint-specific prompts, when prepared, are in `sprints/`.
