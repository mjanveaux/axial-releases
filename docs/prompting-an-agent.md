# How to prompt an agent with axial

This page is for the person who runs an agent (Claude Code, Codex, or anything
that can call a CLI) and wants it to use axial well. It covers the one-time
setup, the loop the agent lives in, a short AGENTS.md block you can paste into
a repository, how to write an issue an agent can act on, and the prompts that
work. Words in *italics* on first use are defined in the [glossary](GLOSSARY.md).

For running agents unattended, see the *dev loop* (`axial dev`, SPEC §25).
This page is about the ordinary case: a session you launch by hand.

---

## 1. One-time setup

Three things have to be true before the first prompt.

**The tracker is selected by the checkout, the credential by the machine.**

```toml
# <repo>/.axial/config.toml   (tracked, committed)
db_url = "https://axial.airforge.io"
workspace = "myteam"
```

Put both credentials on the machine with `axial login`; each lands on its
own key, so neither replaces the other:

```bash
axial login --github                       # you: your own session → db_token
axial login --token <agent token>          # your agents → db_token_myteam.agent
```

```toml
# ~/.config/axial/config.toml   (per machine, never committed)
db_url = "https://axial.airforge.io"
workspace = "myteam"
db_token = "<your session>"
db_token_myteam.agent = "<agent token from your account page>"
```

Then tell the agent's session which one is its own, in the agent's
environment only — a harness settings file (`env` in Claude Code's
`settings.json`), or the launcher:

```bash
AXIAL_CREDENTIAL=agent AXIAL_ACTOR=claude claude
```

`AXIAL_CREDENTIAL` is not a secret, so it is safe in a settings file; the
token stays in the machine file. `axial mcp` and `axial dev` choose the agent
credential on their own. Everything else you run in your own shell uses your
session, so `issue approve` and `issue close` record you, not the agent. Check
with `axial config`: its `credential_kind:` line says which kind was asked
for, and `db_token:` names the key that answered.

The checkout file may name a target and nothing else. A token in a
repository file is a configuration error, and an `.envrc` that exports one is
the same mistake with more steps.

**The agent has a name.** Set `AXIAL_ACTOR` in the agent's environment to a
stable label for this agent: `claude`, `codex`, `pi` — not `actor` in the
machine file, which your own commands read too. It is attribution,
not identity: the *principal* is the account behind the token, and the label
only records who to credit.

**The preflight passes.** Have the agent run these before its first write:

```bash
axial config     # every resolved value and where it came from, no network
axial doctor     # connectivity, schema, and the workspace's project keys
axial whoami     # principal, bound actors, effective actor
```

If any of the three shows a target you did not expect, stop. Never let an
agent "fix" it by pointing at a different database or creating a workspace.

---

## 2. Every session: start an attempt

An *attempt* is one execution. Review gates compare attempt ids, so a
session that has none, or that inherited another session's, cannot satisfy a
gate and cannot be told apart in the audit trail.

Mint one at the launcher, once, and let everything the agent runs inherit it:

```bash
# Interactive: the shell exports it, the agent inherits it
export AXIAL_ATTEMPT="$(axial attempt start)"
claude

# Or: replace the launcher with the agent, under a fresh id
axial attempt start -- claude
axial attempt start --role review -- codex     # a reviewer, launched separately
```

Rules that follow from this:

- `start` always mints. `axial attempt resume` never does; use it to
  continue a session after a crash or a context reset.
- Never mint in `.envrc`. Entering a directory is not starting an execution,
  and a reviewer opened from the implementer's shell would inherit the
  implementer's id and fail its own gate.
- Never rotate the id inside a session to get past a gate. A gate that
  refuses is telling you the review is not independent.

---

## 3. The loop the agent lives in

Three commands do most of the work:

```bash
axial next --project APP --lease 2h      # pick the best ready issue and claim it
axial context APP-42 --full               # one read: ownership, handoff, deps, history
axial issue close APP-42 --from-merge     # after merge: outcome and evidence from the PR
```

The full lifecycle, for an agent that is given a specific issue rather than
"the next one":

```bash
axial context APP-42 --full
axial issue claim APP-42 --lease 2h
axial issue update APP-42 --status in_progress --expect-status todo

git checkout -b claude/APP-42-short-description
# ... work; commits carry the trailer  Axial: APP-42
# ... open the PR with title  [APP-42] Short description  and body line  Axial: APP-42

axial comment add APP-42 "PR: https://github.com/org/repo/pull/118"
axial ref add APP-42 --relation produces --kind pr --target https://github.com/org/repo/pull/118
axial verify APP-42                      # attests checkout-backed acceptance criteria

# after the PR merges
axial issue close APP-42 --expect-status in_progress --from-merge
```

`--from-merge` reads the issue's `produces` PR reference, confirms it merged,
and records the outcome and the merge commit as evidence. Without a PR
reference, spell it out:

```bash
axial issue close APP-42 --expect-status in_progress \
  --outcome shipped --evidence pr:118 --evidence commit:<sha> --verified-commit <sha>
```

When the agent must stop before it is done:

```bash
axial issue release APP-42 \
  --done "what is complete" \
  --remaining "what the next actor should do" \
  --gotchas "risks or non-obvious context"
```

The *handoff* is what the next session reads first in `axial context`. It is
worth more than any amount of chat history.

Things the loop never does:

- Guess an id. `issue create` echoes the id; concurrent actors allocate
  numbers between observations, so `APP-43` is a guess even if it looks
  obvious.
- Close on PR-open. An open PR or locally complete code is `in_progress`.
- Reactivate a `done`, `blocked` or `canceled` issue quietly. Say so and ask.
- Poll with `sleep`. Wait with `axial issue await APP-42 --until done` or
  `axial watch`; a timeout is exit 5.
- Swap `AXIAL_ACTOR` to make a review look independent. The gate does not
  compare actors, and the audit trail records the swap.

---

## 4. An AGENTS.md block you can paste

This is the whole discipline, in the form an agent can auto-load. Replace
`APP` and the branch prefix. Keep the repository's own rules below it.

```markdown
## Issue tracking (axial)

Work in this repository is tracked in axial, project `APP`. Any change that
will become a pull request is tracked by exactly one issue; read-only
investigation needs none.

Before the first write in a session: `axial config`, `axial doctor`,
`axial whoami`. If the target is not project `APP` in workspace `<name>`,
stop and report. Never point at another database.

Every session runs under one attempt. The launcher mints it
(`axial attempt start`); do not mint again, and do not rotate it.

Lifecycle:
1. Identify the issue before changing files. If none exists and the change
   is headed for a PR, `axial issue create APP "<deliverable title>"` and use
   the id it echoes.
2. `axial context APP-N --full`, then `axial issue claim APP-N --lease 2h`.
3. `axial issue update APP-N --status in_progress --expect-status todo`.
4. Branch `claude/APP-N-short-description`; commit trailer `Axial: APP-N`;
   PR title `[APP-N] Short description`; PR body line `Axial: APP-N`.
5. One PR per issue. A multi-PR effort is a parent with one child per PR.
6. Record the PR: `axial comment add APP-N "PR: <url>"` and
   `axial ref add APP-N --relation produces --kind pr --target <url>`.
7. Close only after merge: `axial issue close APP-N --expect-status in_progress --from-merge`.
   If you cannot merge, leave it `in_progress` and say so.
8. Stopping early: `axial issue release APP-N --done ... --remaining ... --gotchas ...`.

Prerequisites are dependency edges (`axial issue block APP-N --by APP-M`),
not a hand-set `blocked` status. Take the next queued task with
`axial next --project APP --lease 2h`, which selects and claims in one step.

axial's exit codes are part of its contract: 0 ok, 1 runtime, 2 usage,
3 not found, 4 conflict or failed guard, 5 timeout. Consume ids echoed by
mutations; never predict them.
```

That is about forty lines. If yours has grown past a hundred, it is
restating SPEC.md; link to SPEC.md and DOGFOOD.md
instead. DOGFOOD.md is the long form of this block, as used by axial itself.

---

## 5. Writing an issue an agent can act on

An agent reads the issue description as its brief, so the description is
where the prompting actually happens. The description is the *canonical
scope*: when scope changes, edit it (`issue update --description-file`) and
the old text stays in its history. Do not append "ADDENDUM" comments.

**Title:** the deliverable, not the activity. "The export includes closed
issues" beats "Look into export".

**Description, in this order:**

1. **Objective.** One paragraph: what exists when this is done.
2. **Scope.** What may change. Name files or packages if you know them.
3. **Exclusions.** What must not change, stated as concretely as the scope.
   Agents expand scope when nothing tells them where it ends.
4. **Pointers.** The ADR, spec section, runbook or prior issue that owns the
   rules this work must honour. Links, not summaries.
5. **Done when.** The checks that decide it. Then make them acceptance
   criteria, so the close guard enforces them:

   ```bash
   axial ac add APP-42 --assert file-exists --path docs/export.md --gate close
   axial ac add APP-42 --assert reference-present --kind pr --gate close
   ```

   For behaviour, arm the criterion on the test file, and name the passing
   run in the close evidence. Reserve `file-exists` on the deliverable itself
   for documents.

**Structure with edges, not prose.** "Do B after A" is
`axial issue block APP-B --by APP-A`. A program is a parent with children,
`--parent APP-P`. Dispatch reads edges; it cannot read "after".

**Leave out** the conversation that led here, alternatives you rejected
unless the agent would otherwise re-propose them, and anything the code
already says.

For a slice inside a larger authorized program, the description is the
*packet*: prompts/implementation-packet.md
lists the fields, and `axial context APP-42 --packet` projects and checks one.

---

## 6. Prompts that work

Prompts are short when the discipline lives in AGENTS.md and the brief lives
in the issue. These are enough:

> Take the next queued task in APP.

> Work APP-42.

> Review APP-42. You are a separate session; start your own attempt.

> APP-42's PR merged. Close it.

> Scope APP-40 into children, one per PR, with dependency edges. Do not
> implement anything.

> Before you change anything: `axial config`, `axial doctor`, `axial whoami`,
> and tell me what they say.

And the ones that cause trouble:

> Close APP-42, the PR is open. *(closing on PR-open; the evidence does not exist yet)*

> Use `--actor mark` for the review. *(a label swap is not an independent review, and it misattributes)*

> Just create APP-43 for that. *(a predicted id)*

> Check every 20 minutes until the PR merges. *(the model as a timer; use `issue await` or `watch`)*

---

## 7. Under MCP instead of the CLI

An agent that speaks the Model Context Protocol gets the same tracker
operations as tools:

```bash
axial attempt start --role implement -- axial mcp
```

One process, one fixed actor, attempt and workspace; never swap identities per
call. `config`, `doctor`, `verify`, `sync-git`, `watch`, `export` and `import`
stay CLI-only, so a session that needs them runs the CLI too.

---

## 8. Where to go next

- DOGFOOD.md: the long form, as axial uses it on itself.
- prompts/: session bootstrap, feature session,
  contract review, and the program/promotion/packet/review templates.
- Role profiles: what `--role implement|review|control`
  and a risk tier mean.
- SPEC §25 and ADR-109: running
  the loop unattended with `axial dev run`.
