---
name: opencode-fanout
description: "Fan work out to the OpenCode CLI as background workers, up to 6 at once, to keep bulk mechanical work off the main agent's quota. Use when a task splits into independent chunks that need a capable model but not your judgement - transcribing or reformatting a dump, converting a batch of files, drafting boilerplate, per-file summaries. Free OpenCode Zen models (`opencode/*-free`) make it zero-cost. Requires checking `opencode models` and asking the user which model to use before dispatching."
---

# OpenCode Fan-Out

Dispatch OpenCode CLI runs as background workers so the main agent spends tokens on judgement rather than on bulk. Each
worker is a full agent with file access in one directory; you write the brief, it writes the files, you review.

**Six concurrent workers is the ceiling.** Past that they contend for the same files and the review cost exceeds the
saving.

---

## When This Pays, And When It Does Not

| Delegate                                                          | Keep                                                   |
| ----------------------------------------------------------------- | ------------------------------------------------------ |
| Converting a dump into structured Markdown                        | Deciding what the structure should be                  |
| The same transformation across many files                         | Anything where being wrong is expensive and quiet      |
| Drafting from a spec you have already written                     | Writing the spec                                       |
| Per-file summaries, inventories, mechanical extraction            | Architecture, naming, API shape, security              |
| Boilerplate with a worked example to copy                         | Work needing conversation context the worker cannot see |

**A worker has none of your context.** Everything it needs goes in the prompt. If the brief takes longer to write than
the task takes to do, do the task.

---

## Preflight, In Order

**1. Confirm the CLI exists.** If it does not, say so with the install command and stop; do not silently do the work
yourself at full price.

```bash
command -v opencode || echo "MISSING: install with 'bun add -g opencode-ai'"
```

**2. List the models.** Never assume an id - the catalogue changes and a wrong id fails late.

```bash
opencode models opencode | grep -E -- '-free$|big-pickle'   # the free Zen models, live
opencode models | grep -iE "glm|deepseek|gemini"            # everything else, 400+ entries
```

The free list rotates: stealth and preview models arrive and leave without notice. Read it every session.

**3. Ask the user which model to use, unless they already named one.** Use `AskUserQuestion`, one question, with the
default first and marked. Model choice is cost, and it is the user's cost. A model the user named earlier in the same
session is a standing answer: state it back and dispatch rather than asking again.

- **A free Zen model from step 2** - the default. `opencode/space-bunny-free` and `opencode/step-5-preview-free` ran
  clean worker briefs when this was written, and both are zero-retention. Several other free models keep prompts or
  train on them; check https://opencode.ai/docs/zen/ before sending private code to one
- `openrouter/z-ai/glm-5.3-flash` - the paid fallback when no free model fits or the work is private
- `opencode-go/<model>` - only when the user has an OpenCode Go subscription; it draws from that plan's monthly
  per-model allowance, shown on the Go page of the OpenCode Console

State the chosen id back before dispatching.

**Free Zen models only answer the OpenCode CLI.** A direct API call returns "OpenCode's free tier can only be used from
within OpenCode". Run them through `opencode run`; never route them through a proxy or another harness.

**4. Point the title model at a free model.** OpenCode names each session with a separate small model, paid by default.
On an account with no credit it logs `Insufficient account funds` on every run. Export this once per shell:

```bash
export OPENCODE_CONFIG_CONTENT='{"small_model":"opencode/space-bunny-free"}'
```

---

## Dispatch

### The Command

```bash
opencode run \
  --model <provider/model> \
  --variant <low|medium|high> \
  --dir <absolute path to the working directory> \
  "<the whole brief, one argument>" \
  < /dev/null > "<log path>" 2>&1
```

**Always close stdin with `< /dev/null`.** From a non-interactive shell an open stdin makes `opencode run` wait on it
forever: the log stays empty and no file is touched until `timeout` fires. Verified with two free models.

`--variant` is the reasoning effort. Omit it for the model's default.

**Always redirect to a log file.** Piping to `tail` buffers, so if the worker is killed you get an empty file and no
idea why. The log is the only evidence you will have.

**Always wrap the run in `timeout`.** A worker can hang with the process alive, the log at 0 bytes and no file touched,
and stay that way indefinitely. Observed: fifteen minutes on a four-file brief with nothing written. `timeout 900`
turns that into an exit code you will be notified about instead of a wait you have to notice.

```bash
timeout 900 opencode run --model "$MODEL" --dir "$PWD" "$BRIEF" < /dev/null > "$LOG" 2>&1; echo "exit=$?"
```

Exit 124 means the timeout fired. Check `git status --porcelain` in `--dir` before deciding whether anything landed.

**`--dir` is a hard boundary.** OpenCode auto-rejects reads outside it - `permission requested: external_directory ...
auto-rejecting` - and then **exits 0 anyway** having done nothing. Copy every input inside the directory first, and
delete it afterwards.

### Parallel, Up To Six

Launch each with `run_in_background: true`, one per call in a single message so they start together. Give each its own
log file and its own output paths. **Never let two workers write the same file.**

```bash
# One worker. Repeat per chunk, each backgrounded, each with a distinct LOG and target.
LOG=<scratch>/oc-<chunk>.log
opencode run --model "$MODEL" --dir "$PWD" "$(cat <<'BRIEF'
...
BRIEF
)" < /dev/null > "$LOG" 2>&1
```

Wait for the task notifications. Do not poll, and do not start reviewing chunk 1 while chunk 2 is still writing to a
file you are reading.

### Stalls

A healthy run writes to its log within the first minute or two. If the log is still 0 bytes **and** `git status
--porcelain` in `--dir` is empty after three minutes, the worker is stalled, not slow. Kill it and either re-dispatch
once with a shorter brief or do the work yourself; a brief that already contains the exact code is faster to apply
than to wait on.

Kill by executable name, never by pattern: `pkill -x opencode`. `pkill -f "opencode run"` also matches the shell that
is running your own command, because the brief is in its argument list, and kills your session's command (exit 144).

Do not put a repo you care about under a worker without a branch or a separate `git worktree`: a stalled or half-done
run then costs one `git checkout -- .`, not a reconstruction.

### Sequential

Chain in one background call when later chunks depend on earlier output, or when the workers touch overlapping files.
One log per stage, `&&` between them, so a failure stops the chain instead of feeding garbage forward.

**Prefer parallel.** Sequential is for real dependencies, not for tidiness.

---

## Writing The Brief

A worker prompt is a work order, not a chat message. Six things, and the first two are what actually prevent damage:

1. **Name every file to create, and say "and nothing else".** Without it you get stray scratch files in the repo
2. **State the inputs as paths inside `--dir`**, and say "read them, never mention their paths in your output" if the
   output is committed - otherwise machine paths leak into the deliverable
3. **Give the output format concretely.** Heading levels, table columns, casing. "Well structured" produces whatever the
   model likes today
4. **Carry the house rules in.** Line width, casing, tone, no emoji. The worker has never read your `AGENTS.md`
5. **Say what must be preserved verbatim** when the task is a transformation. Models summarise by reflex, and a lossy
   pass over a source record is worse than no pass
6. **Ask for a short report** - what it wrote, what it could not do, what it guessed at

Use a quoted heredoc so `$`, backticks and quotes survive the shell.

---

## Verifying, Which Is Not Optional

**OpenCode exits 0 when it has done nothing at all.** Verified: a run that was auto-denied its input file still returned
`EXIT=0`. Never report success from an exit code.

Check, in this order:

```bash
ls -la <expected outputs>                       # do they exist, are they non-trivial
grep -c "<structural marker>" <output>          # right shape, right count
grep -rn "/home/\|C:\\\\Users\|/tmp/" <output>  # no machine paths leaked
tail -20 "$LOG"                                 # what it said it did
```

Then **read the parts that carry risk.** In practice a good worker still fabricates a plausible column, invents a
heading level, or quietly drops the one row you cared about. Spot-check the data a decision will be made on: the
numbers, the names, the totals.

Fix small defects yourself. Re-dispatch only if a chunk is broadly wrong, with the defect named in the new brief.

---

## Failure Modes Seen In The Wild

| Symptom                                        | Cause And Fix                                                                      |
| ---------------------------------------------- | ------------------------------------------------------------------------------------ |
| Empty log, exit 0, no output files             | Input was outside `--dir` and auto-rejected. Copy it inside and rerun                |
| Empty log, exit 124, nothing written           | stdin was left open. Add `< /dev/null` and rerun                                     |
| `free tier can only be used from within OpenCode` | A free model was called outside `opencode run`. Dispatch it through the CLI       |
| `Insufficient account funds`                   | Paid model on a $0 Zen balance, or the title model. Pick a free model; see step 4  |
| `An active OpenCode Go subscription is required` | An `opencode-go/*` model without Go. Use a free Zen model instead                |
| Empty log, process alive, no files, for minutes | Stalled. `pkill -x opencode`, then re-dispatch shorter or apply the brief yourself   |
| Your own shell died with exit 144              | You ran `pkill -f "opencode run"`, which matched your command line. Use `pkill -x`  |
| Exit 124                                       | `timeout` fired. Check `git status --porcelain` before assuming nothing landed        |
| Empty log, process gone                        | Killed. Never diagnose from a `\| tail` pipe - redirect to a file from the start     |
| Output has an invented column or section       | The brief left the format open. Specify columns and headings literally               |
| Source text summarised instead of transcribed  | Say "transcribe, do not summarise" and name what must survive verbatim               |
| Stray files in the repo                        | The brief did not say "and nothing else". `git status` after every run               |
| Model id rejected                              | Catalogue drift. Re-run `opencode models` rather than retrying the same id           |

---

## Reporting Back

Say which model ran, how many workers, what each produced, **and what you corrected**. The corrections are the useful
part - they tell the user whether the next fan-out should use a stronger model or a tighter brief.

Never present a worker's output as verified when you only checked that the file exists.
