# Run the dev loop on your repository

The *dev loop* (`axial dev`) takes ready issues from your tracker and has agents
work them: it claims an issue, gives an agent a separate git worktree, checks
the result from what actually happened (a commit, a passing check) rather than
the agent's own report, opens a pull request, and, if you allow it, has another
agent review it and merges it. This page gets it running on a repository that
already uses axial. The full contract is SPEC §25.

Before you start, have the basics working: a tracker target in
`.axial/config.toml`, and your own login plus an agent credential
([add a teammate](add-a-teammate.md) walks through both).

---

## 1. What you need installed

| For | Install | Sign in |
|---|---|---|
| agents on Claude | the `claude` CLI | `claude` once, interactively |
| agents on Codex | the `codex` CLI | `codex login` |
| pull requests and merges | the `gh` CLI | `gh auth login` |
| the tracker | `axial` (`curl -fsSL https://github.com/mjanveaux/axial-releases/releases/latest/download/install.sh | sh`) | `axial login --token <agent token>` |

You need at least one agent runtime. `pi` runs on Linux only (its shell sandbox
is Linux-only); on macOS use `claude` or `codex`, which have not yet been
verified by a supervised run on a Mac — watch your first run.

The dev loop always uses the **agent** credential, never your own login: it
refuses a person's session, and `axial dev` picks the agent credential by
itself.

---

## 1a. Set up each runtime on this machine

Runtime and model settings are **per machine**. They go in
`~/.config/axial/config.toml`, never in the repository's `.axial/dev.toml`:
tracked files must not choose which program a supervisor runs or which model
it pays for. `axial dev doctor` checks every one of them before anything is
claimed, and names the key to set when one is wrong.

**Claude** (`claude`)

```toml
claude_path = "/usr/local/bin/claude"      # optional; otherwise found on PATH
claude_config_dir = "/home/you/.claude"    # optional; which Claude account agents use
claude_permission_mode = "acceptEdits"     # the default; a headless agent cannot answer prompts
```

**Codex** (`codex`)

```toml
codex_path = "/usr/local/bin/codex"        # optional; otherwise found on PATH
codex_home = "/home/you/.codex"            # optional; which Codex account agents use
codex_sandbox_mode = "workspace-write"     # the default; the other allowed value is read-only
```

**pi** (`pi`, Linux only)

pi needs more, because axial builds its sandbox and its tracker connection
itself:

```toml
pi_path = "/opt/pi-0.84.3/pi"              # the pi binary (there is no AXIAL_PI_BINARY)
pi_agent_dir = "/home/you/.pi/agent"       # optional; pi's account directory
pi_mcp_adapter_path = "/opt/pi-mcp-adapter" # a pi-mcp-adapter 2.21.1 checkout (contains index.ts)
```

- Only tested pi versions run: **0.84.2 and 0.84.3**. A newer pi is refused
  until it has been re-tested against axial, and there is deliberately no
  override. Keep a supported version and point `pi_path` at it.
- pi has no built-in MCP client, so `pi_mcp_adapter_path` is required: it is
  how a pi worker reaches the tracker.
- pi cannot start without a model, so **every** role profile must be mapped
  for it, with provider-qualified names such as `anthropic/claude-sonnet-4-5`
  (see Models below).

**Models**

A model mapping is a key per role **profile**, optionally per **risk tier**
and per **runtime**:

```text
model_<profile>                    every runtime
model_<profile>_<tier>             every runtime, one tier
model_<runtime>_<profile>          one runtime
model_<runtime>_<profile>_<tier>   one runtime, one tier
```

- Profiles: `planning`, `implementation_standard`,
  `implementation_high_risk`, `review`.
- Tiers: `routine`, `standard`, `high`, `critical`.
- Runtimes: `claude`, `codex`, `pi`.
- Keys spell profiles with underscores (`model_implementation_standard`);
  commands spell them with hyphens (`axial dev resolve implementation-standard`).
- A key without a runtime applies to **every** runtime. Model names are the
  vendor's, so on a machine that runs more than one runtime, qualify them.
- `effort_*` keys (reasoning effort) follow exactly the same scheme.

For example, a machine running Claude and Codex:

```toml
model_claude_implementation_standard = "<claude model id>"
model_claude_review                  = "<claude model id>"
model_codex_implementation_standard  = "<codex model id>"
model_codex_review                   = "<codex model id>"
effort_codex_review                  = "high"
```

A profile with no mapping lets the runtime choose its own default, except on
pi, which refuses. To see which key a dispatch would use, and why:

```bash
axial dev resolve implementation-standard --tier standard
axial dev resolve review
```

**The worktrees folder**

`.axial/dev.toml` is shared through git, so keep its `worktrees` path
**relative**, like the `../<repo>-worktrees` that `dev init` writes. An
absolute path such as `/home/you/...` only works on one machine.

## 2. Declare the loop for this repository

```bash
axial dev init --project APP --workers 1
```

It finds the agent runtimes installed on this machine and uses the first of
claude, codex and pi it can launch, printing a `runtimes detected:` line. Pass
`--runtime claude` to choose instead. This writes `.axial/dev.toml`, which you
commit. It declares a target, never a
credential. Then add what `init` does not write, for your toolchain. For a
Node repository:

```toml
# .axial/dev.toml
project = "APP"
runtime = "claude"
workers = 1
reviewers = 0
worktrees = "../app-worktrees"

[verify]
# What an agent may run to check its own work. Without this, agents get the Go
# toolchain only — they cannot run your tests in any other language.
commands = ["npm ci:*", "npm run build:*", "npm test:*"]
# What the supervisor may OBSERVE as passing verification for a commit: your
# CI's check names. Empty means nothing counts as verified.
checks = ["ci/test"]
```

Leave `[review]` and `[merge]` out to begin with: the loop then opens pull
requests and never merges, and you review and merge them yourself. Add them
later (SPEC §25):

- `reviewers = 1` with `[review] policy = "required"` has a runtime review each
  pull request. A different runtime is asked first (claude's work goes to
  codex, then pi), and the implementer reviews its own work in a fresh session
  only when none of those can run. With only an Anthropic account, say so
  directly and skip the wait on runtimes you don't have:

  ```toml
  [review]
  policy = "required"
  claude = ["claude"]
  ```

- `[merge] mode = "strong"` lets axial arrange the merge through GitHub
  auto-merge, which needs branch protection with a required check.
- `[budget]` caps the tokens agents may use — `issue_tokens` per issue,
  `run_tokens` per `dev run`. It is off by default; an issue that reaches its
  cap is parked on a decision for you (SPEC §25).

---

## 3. Check before you run

```bash
axial dev doctor
```

Every line should be `ok`. It names the runtime and account that would run,
the tracker target, the actor, and — if you declared `[merge]` — whether the
repository can do the merge you asked for. Fix anything it reports before
running; a failure here costs nothing, a failure mid-run costs an attempt.

---

## 4. Give it work it can do

The loop takes the **highest-priority ready issue**: open, unclaimed, not
blocked, with no open decision. It never takes a parent issue that still has
open children, or an issue labelled `human-only` — use that label for steps an
agent cannot perform (enable a cron in production, fetch a credential).

An issue an agent can finish in one attempt says what to do, what not to touch,
and how you will know it is done. See
[prompting an agent §5](prompting-an-agent.md) for the shape, and add at least
one close-gating acceptance criterion (`axial ac add`).

To start small, name the issues:

```bash
axial dev run --issues APP-12,APP-13
```

---

## 5. Run it, and watch

```bash
axial dev run                   # in the foreground; Ctrl-C drains cleanly
axial dev controller start      # or detached, surviving the terminal
axial dev status                # supervisor, workers, attempts, leases
axial dev attach                # follow a detached controller's output
axial dev stop                  # drain and wait for it to exit
```

The web page at your service's address shows what needs you: decisions agents
filed, stalled work, and pull requests waiting for review.

When the loop hits something it cannot decide, it files a *decision* on the
issue and moves on. Answer it on the web page, by replying to its email if the
channel is set up, or with `axial decision answer`.

---

## 6. When something goes wrong

| You see | Do |
|---|---|
| `dev status` says `supervisor: stale` | the process was killed (a laptop sleeping counts). Start it again; it adopts what it can |
| `already claimed` when resuming an issue | the killed run still holds the claim: `axial issue release APP-12 --done "…" --remaining "…"`, then `axial dev review` |
| the whole run stopped after one failure | by design: the first failed attempt drains the run so a broken setup cannot burn through the queue. Fix the cause, then run again |
| a runtime's quota window closed | work is parked, not failed, and resumes when the window reopens (or on another declared runtime) |
| an agent could not run your tests | your `[verify] commands` does not grant them — add the prefixes |

Never kill a dev run with `timeout` or `kill -9`: use `axial dev stop` or
Ctrl-C, which drain and release cleanly.

---

## 7. Paste this into your repository's AGENTS.md

The short block in [prompting an agent §4](prompting-an-agent.md) tells every
agent — dev-loop workers and sessions you launch by hand — how to use the
tracker in your repository. It is the same for both.
