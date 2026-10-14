# OpenCode Fan-Out

![Claude Code skill](https://img.shields.io/badge/Claude_Code_skill-D97757?style=for-the-badge&logo=claude&logoColor=white)
![OpenCode](https://img.shields.io/badge/OpenCode-000000?style=for-the-badge)
![MIT licence](https://img.shields.io/badge/MIT_licence-blue?style=for-the-badge)

**Fan work out to headless OpenCode CLI workers, up to six at once, on free OpenCode Zen models.**

> The worker does the bulk, you keep the judgement.

_Fan-out_ is dispatching many independent workers at once and reviewing what comes back.

## Table of Contents

<details>
  <summary>Expand</summary>
  <ol>
    <li><a href="#what-it-does">What It Does</a></li>
    <li><a href="#quick-start">Quick Start</a></li>
    <li><a href="#which-model-to-use">Which Model to Use</a></li>
    <li><a href="#how-it-stays-safe">How It Stays Safe</a></li>
    <li><a href="#what-it-cannot-do">What It Cannot Do</a></li>
    <li><a href="#licence">Licence</a></li>
  </ol>
</details>

## What It Does

- **Headless OpenCode Workers.** Each worker is one `opencode run` in one directory. You write the brief, it writes the
  files, you review.
- **Zero Cost on Free Zen Models.** OpenCode serves a rotating set of free models, including stealth and preview models
  that other providers do not offer. They only answer the OpenCode CLI, so this skill drives that CLI.
- **Up to Six at Once.** Launch them in parallel, each with its own brief, log and output files, and never let two
  workers write the same file.

If a task needs your judgement or your conversation context, you do not need this skill.

## Quick Start

1. **Install OpenCode and Sign In.** Choose OpenCode as the provider.

   ```bash
   bun add -g opencode-ai
   opencode auth login
   ```

1. **Clone Into the Global Skills Directory**, then restart Claude Code. The skill is available as `/opencode-fanout`.

   ```bash
   git clone https://github.com/TolongLabs/opencode-fanout ~/.claude/skills/opencode-fanout
   ```

1. **List the Free Models.** The list changes often.

   ```bash
   opencode models opencode | grep -E -- '-free$|big-pickle'
   ```

1. **Smoke-Test One Worker.** Close stdin, or the run waits forever.

   ```bash
   export OPENCODE_CONFIG_CONTENT='{"small_model":"opencode/space-bunny-free"}'
   mkdir -p /tmp/oc-smoke && cd /tmp/oc-smoke && printf 'alpha\nbeta\n' > in.txt
   timeout 300 opencode run --model opencode/space-bunny-free --dir "$PWD" \
     "Read in.txt and write out.txt with its lines uppercased. Create no other file." < /dev/null > run.log 2>&1
   cat out.txt   # ALPHA BETA
   ```

## Which Model to Use

| Model                          | Cost                      | Notes                                                    |
| ------------------------------ | ------------------------- | -------------------------------------------------------- |
| `opencode/space-bunny-free`    | Free                      | Default. Zero-retention. Verified on worker briefs       |
| `opencode/step-5-preview-free` | Free                      | Zero-retention. Verified on worker briefs                |
| Other `opencode/*-free`        | Free                      | Some keep prompts or train on them. Read the Zen docs    |
| `openrouter/z-ai/glm-5.3-flash` | Paid, cheap              | Fallback for private code or when no free model fits     |
| `opencode-go/<model>`          | OpenCode Go subscription  | Draws on the Go plan's monthly per-model allowance       |

Retention terms per free model: [OpenCode Zen docs](https://opencode.ai/docs/zen/). Go allowances are on the Go page
of the [OpenCode Console](https://opencode.ai/console).

## How It Stays Safe

- **The Agent Asks Before Dispatching.** Model choice is cost, so the skill asks which model to use.
- **`--dir` Is a Hard Boundary.** OpenCode rejects reads outside it.
- **Every Run Has a Timeout and a Log.** A stalled worker becomes exit 124, not an endless wait.
- **Exit 0 Proves Nothing.** The skill checks the output files, not the exit code.

## What It Cannot Do

- **Free Models Outside OpenCode.** A direct API call or a proxy is refused with "OpenCode's free tier can only be used
  from within OpenCode". This skill never works around that.
- **Guarantee a Model Stays Free.** Free and preview models are limited-time and leave without notice.

## Licence

MIT. See [LICENSE](LICENSE).
