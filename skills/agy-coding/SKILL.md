---
name: agy-coding
description: Drive agy CLI to change a repo: write the task file, check the envelope, run tests yourself, upgrade Flash to Pro to Claude, and loop until green.
---

# agy-coding

agy changes the working tree. You are the caller: pick the model, write the task, check the result, run verification, read the diff, commit. Do not treat agy as one-click done.

Official docs: https://www.antigravity.google/docs/cli/

## When to Use

- The user wants code, tests, or a refactor in a git repo
- You need headless, resume, or a model upgrade
- Do not use this to decide product right/wrong, to ship to prod, or to let agy commit

## Procedure

1. `cd` to the repo root. Confirm no other agy is hitting the same tree.
2. Run `agy models` and use a slug from that list. Do not invent names.
3. Write the task to a file (goal, scope, verify command, bans). Feed it with `-p "$(cat /tmp/agy-task.md)"`.
4. Put `--model` before `-p`. Set `--print-timeout` to 15-20m. Use `--output-format json`.
5. Add `--dangerously-skip-permissions` only on a trusted isolated tree. Otherwise shell stays Ask; headless becomes a soft-deny: exit 0, envelope SUCCESS, zero file changes.
6. Trust only: `status == "SUCCESS"`, save `conversation_id` immediately, treat any `error` as failure.
7. You run typecheck, tests, and needed real clicks. The stop condition is those exit codes, not the envelope.
8. If it failed: resume the same model with `--conversation` and an incremental prompt (what went red, which fake-fix is banned).
9. Changing models means a new session plus the previous failure evidence. Upgrade chain: newest Flash (max thinking) -> miss twice then newest Pro -> miss again then Claude. Do not retry the same tier forever.
10. On timeout, run `git status` / `git diff` first. Timeout does not mean nothing was written.
11. When green: read the diff, drop `.agy-*`, temp scripts, and agy-written acceptance docs. You commit. agy does not commit.

Resume:

```bash
agy --model <same-model> \
  --conversation "$CONVERSATION_ID" \
  --effort high \
  --dangerously-skip-permissions \
  --print-timeout 20m \
  --output-format json \
  -p "$(cat /tmp/agy-task-resume.md)"
```

## Pitfalls

- `--model` after `-p` falls back to the default model
- Default 5-minute timeout cuts real edits short
- SUCCESS + unchanged tree: check permissions and auth first
- Two agy processes on one tree
- Hand-editing product code then resuming
- Letting agy run `git commit` or `git push`
- Unit tests green, real click dead: fake navigation or server runtime in the client bundle; upgrade to Pro
- Treating a local-only rewrite as the production design

## Verification

Before start: correct repo, slug from `agy models`, model flag before `-p`, timeout >= 15m, no second agy, prompt has result + verify + bans.

After: envelope clean or timeout diff already inspected; `conversation_id` saved; you ran tests; diff has no junk and no fake-fix; commit author is this digital employee name and its fixed email.
If any box is unchecked, stop. Do not burn another Flash run on the same miss.
