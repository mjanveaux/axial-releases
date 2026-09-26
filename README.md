# axial

**A task tracker for people and the AI agents that work with them.**

This repository publishes axial's release binaries. The source code is not
published here; every release built from it is copied here, with a checksum for
each file and an installer.

## What axial is

axial keeps a team's work — projects, issues, milestones, decisions — in one
place that both people and coding agents (Claude Code, Codex, pi) use directly.
It is built so that several agents can work through a backlog at the same time
without stepping on each other, and so that a person can see what happened and
decide what matters.

- **One command line for people and agents.** Output is short, stable text with
  fixed exit codes, so an agent can read it and a script can branch on it.
  `axial ui` opens an interactive terminal view for a person.
- **Safe for many agents at once.** Claiming an issue is atomic: two agents
  asking for work never get the same issue. Claims carry a lease and a handoff
  note, so work left half-done is picked up rather than lost.
- **Every change is attributed.** Each write records who made it and in which
  session. A person's approval can only be recorded by that person.
- **Agents can ask a person.** An agent that needs a decision files one; the
  person answers it in the terminal or the web view, and the agent continues.
- **A supervised loop for repositories.** `axial dev run` hands ready issues to
  agent runtimes in isolated git worktrees, has the delivered pull request
  reviewed, and, when you allow it, merges it — within a token budget you set.
- **Hosted or local.** Use a team's hosted service, or keep everything in a
  single local file with no account at all. Your data can always be exported.

## Install

On macOS (Apple silicon) or Linux (x86-64 or arm64):

```bash
curl -fsSL https://github.com/mjanveaux/axial-releases/releases/latest/download/install.sh | sh
```

The installer picks the build for your machine, checks it against the
published sha256, and installs it to `~/.local/bin/axial`. It prints the line
to add to your `PATH` if that folder is not on it yet. It never asks for a
password.

- Pin a version: `curl … | AXIAL_VERSION=v0.70.0 sh`
- Install somewhere else: `curl … | AXIAL_INSTALL_DIR=/usr/local/bin sh` (the
  folder must be writable by you)

There is no build for Intel Macs or Windows.

## Upgrade

```bash
axial upgrade --check   # says whether a newer release exists; changes nothing
axial upgrade           # installs the newest release over this binary
```

`axial upgrade` downloads from this repository, verifies the checksum, runs the
new binary once to confirm its version, and only then replaces the old one — if
anything fails, the installed binary is untouched. It refuses to downgrade. It
never runs by itself: there is no background check and no update reminder, so
a binary an agent is using never changes underneath it.

It needs no GitHub account. If you hit GitHub's rate limit for anonymous
requests, set `GITHUB_TOKEN` (any token) and run it again.

Releases before v0.70.0 do not have `axial upgrade`; run the install command
above once instead.

## Getting started

### Joining a team

Ask the team's admin to invite your **GitHub username**. The admin's invite
page gives them a message to send you with these same steps, filled in.

```bash
axial login --github
```

This prints a short code and an address. Open the address in a browser signed
in to GitHub, enter the code, choose the workspace, and approve. Then:

```bash
axial status     # where each project stands
axial ui         # browse and work in a terminal view
```

If you sign in before you have been invited, axial asks whether you are joining
a team or starting your own organization — choose to wait, and you join the
team automatically the next time you sign in after the invite.

### Trying it locally, with no account

```bash
export AXIAL_DB_URL="file:$HOME/.local/share/axial/axial.db"
export AXIAL_ACTOR="$USER"

axial project create DEMO "Demo project"
axial issue create DEMO "Write the first issue" -p high
axial status
axial next                 # claim the most important ready issue
axial issue close DEMO-1
```

Everything lives in that one file.

## Using axial with agents

Agents act under their own name, with their own token — never a person's.

1. On the hosted service's account page, an admin issues an **agent token**
   for an agent name (for example `henry`) and a workspace.
2. On the machine the agent runs on:

   ```bash
   axial login --token '<agent token>'
   ```

   A person's own `axial login --github` and an agent's token are kept apart
   on the same machine. A process run for an agent should set
   `AXIAL_CREDENTIAL=agent`; `axial mcp` and every `axial dev` command do that
   by default.
3. Give the agent the tracker:
   - as an MCP server — `axial mcp` (stdio), which exposes axial's operations
     as tools; or
   - as a command line — `axial conventions` prints the whole output contract,
     exit codes and command list in one screen, written for an agent to read.

A typical agent turn: `axial next` to claim work, `axial context <ID>` to read
everything about it, `axial issue update`/`comment add` as it goes, and
`axial issue release` with a handoff if it stops early.

### The supervised dev loop

In a git repository whose issues live in axial:

```bash
axial dev init      # writes .axial/dev.toml; detects the agent runtimes installed here
axial dev doctor    # checks everything a run needs, before anything is claimed
axial dev run       # keeps the ready issues moving
```

`dev init` picks the first of `claude`, `codex` and `pi` it finds. Review,
merging and a token budget are off until you turn them on in `.axial/dev.toml`;
`axial dev doctor` shows what is configured, including which runtime reviews
whose work.

## Everyday commands

| Command | What it does |
|---|---|
| `axial status` | Where each project stands: counts, milestones, what needs you |
| `axial issue list --open` | The issues still open |
| `axial issue view <ID>` | One issue in full |
| `axial next` | Claim the best ready issue for you |
| `axial issue close <ID>` | Close an issue |
| `axial issue lint <ID>` | Is this issue specified well enough to hand to an agent? |
| `axial decision list` | Questions agents are waiting on you to answer |
| `axial workspace list` | Every workspace you can use, and which one this terminal uses |
| `axial doctor` | Check configuration, sign-in and connection |
| `axial <command> --help` | Help for any command |

## Checking a download by hand

Each release has one tarball per platform, a `.sha256` for each, and a combined
`checksums.txt`:

```bash
shasum -a 256 -c axial_<version>_darwin_arm64.tar.gz.sha256   # must print: OK
tar -xzf axial_<version>_darwin_arm64.tar.gz                  # one file: axial
```

A file downloaded with a web browser on macOS is quarantined; clear it with
`xattr -d com.apple.quarantine axial`. The installer and `axial upgrade` do not
need this. Binaries are ad-hoc signed, not notarized.

## Versions

Releases follow `vMAJOR.MINOR.PATCH`. Before 1.0, a minor version may change
command output that scripts read; each release's notes list those changes
under **Breaking**. `axial version` prints the version you are running.

## Help

This repository has no issue tracker. If you use axial through a team's hosted
service, ask its admin. `axial doctor` is the first thing to run when something
does not work: it names what is wrong and how to fix it.
