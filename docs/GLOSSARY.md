# Glossary

One page, one definition per word. Read this before SPEC.md, and come back to it
whenever two documents seem to disagree: most of the time they are using two
names for one thing, and the "also called" line here says so.

Each entry gives the plain meaning first, then where the binding definition
lives (a SPEC section or an ADR). SPEC.md is normative; this page is not. If
they differ, SPEC.md wins and this page has a bug.

Words are grouped by the question they answer.

---

## What is being tracked

**Workspace.** The unit of isolation. Every record belongs to exactly one
workspace, and a credential selects it. In the hosted service a workspace
belongs to an organisation (org); "tenant" means the same thing as workspace in
most sentences. Alcalytics is a workspace; so is `axial` itself.
*SPEC §24.*

**Project.** A short key inside a workspace that numbers its issues: project
`ALC` gives issues `ALC-1`, `ALC-2`, … A workspace can hold several projects.
*SPEC §6.*

**Issue.** The unit of work. It has a status (`todo`, `in_progress`,
`blocked`, `done`, `canceled`), a priority, labels, a description, comments,
references, acceptance criteria, and at most one assignee with a lease. One
issue maps to one pull request; a bigger effort is a parent issue with one
child per pull request. *SPEC §7.*

**Description.** The issue's canonical statement of scope. When scope changes
the description changes (`issue update --description-file`), and the old text
stays in its history. Comments are for discussion and events, never for the
new truth. *DOGFOOD.md §Canonical scope.*

**Milestone.** A dated grouping of issues inside one project, for scheduled
work. **Collection.** A named grouping across projects, for a program that
spans them. Neither replaces the parent/child tree that maps issues to pull
requests. *SPEC §6a, §6b.*

**Dependency.** An edge saying "this issue cannot start until that one is
done" (`issue block ALC-5 --by ALC-4`). Dependencies are what dispatch reads;
setting a status to `blocked` by hand does not create one. *SPEC §13.*

**Frontier.** The set of issues whose dependencies are all satisfied and that
nobody holds: the work that could start right now. `axial next` picks from it;
`axial dev run` keeps it drained. Also called *the dependency-ready frontier*.
*SPEC §14, §25.*

**Claim and lease.** Claiming an issue makes you its assignee for a bounded
time, the lease. A lease that lapses can be reclaimed by anyone; `heartbeat`
extends it. Re-claiming an issue you already hold is a conflict on purpose.
*SPEC §9.*

**Handoff.** The structured note left when a claim is released: what is done,
what remains, and the gotchas. `issue release` records it; `axial context`
shows the latest one first. *SPEC §9, §16.*

**Dispatch.** Choosing the next issue for an actor. `axial next` does it in one
step: pick the best issue on the frontier and claim it. *SPEC §14.*

**Focus.** The one issue an actor is treating as current in a given checkout.
Harness hooks read it to know which issue a forwarded decision belongs to.
*ADR-090.*

**Decision.** A question that needs a person, filed on an issue. While it is
open the issue is excluded from dispatch; when it is answered the issue is
dispatchable again. The asker cannot answer its own decision. *SPEC §23.*

**Reference.** A typed link from an issue to an artifact: a file, a pull
request, a commit, another issue. The relation says why (`produces`,
`references`, `supersedes`). A reference can go *stale* when its target
changes; `refs --stale` lists them and `ref touch` acknowledges one after you
have re-read it. *SPEC §8a.*

**Acceptance criterion (AC).** A promise that axial can check before an issue
closes. Stored-state criteria (`reference-present`, `issue-closed`) are
evaluated directly. Checkout-backed criteria (`file-exists`) need an
attestation from `axial verify`. A criterion is either close-gating or
advisory. *SPEC §8b.*

**Attestation.** The recorded result of `axial verify` running a checkout-backed
criterion, anchored to the commit it ran against. *SPEC §7a, §8b.*

**Evidence.** What closes an issue honestly: a pull request and the commit that
was verified. `issue close --evidence pr:NN --evidence commit:SHA
--verified-commit SHA`. Closing without proof is allowed but marked
`unverified`, and the oversight view surfaces it. *SPEC §7, §7a.*

**Outcome.** How an issue ended: `shipped`, `wontfix`, or `duplicate-of <ID>`.
Recorded at close alongside the evidence. *SPEC §7.*

**Close guard.** The checks that run when an issue closes: gating acceptance
criteria, an open review gate, the expected-status guard. A failed guard is
exit 4. **Close policy** is the workspace-wide setting for how strict the
grounded-close rules are (`axial close-policy`). *SPEC §8b.*

**Changes cursor.** The opaque token `axial changes --since <cursor>` returns
so a session can catch up after a restart. Keep the last one; never construct
or compare it. *SPEC §15.*

**Document number.** ADRs, feature rows and milestones take their number from
`axial alloc next`, never from "highest in the tree plus one", because several
actors allocate at once. *SPEC §29.*

---

## Who did it

Axial records three layers of identity on every mutation. They answer
different questions, and confusing them is the single most common source of
trouble.

**Credential (token).** The secret a process presents to the service. Several
tokens can name the same account. A token lives in `~/.config/axial/config.toml`
or `AXIAL_DB_TOKEN`, never in a checkout. *SPEC §4, ADR-135.*

**Principal.** The account behind a credential. Answers *which account is
accountable*. Has a kind: `service` (an agent token) or `human` (a person who
ran `axial login`). Several tokens, one principal. `axial whoami` shows yours.
*ADR-123, ADR-124.*

**Actor.** The label recorded on a mutation. Answers *who to attribute this
to*: `claude`, `codex`, `mark`. An actor is bound to a principal and is only
usable by that principal, but changing the actor never changes the principal.
Since M78 no review gate compares actors; an actor is attribution, not
independence. *ADR-124 §1.*

**Attempt.** One execution. Answers *which run did this*. The supervised dev
loop mints one per worker at launch. An unsupervised session mints its own with
`axial attempt start` and must not mint again until the session ends;
`axial attempt resume` continues one and never mints. Do not generate one in a
repository `.envrc`: entering a directory is not starting an execution, and a
reviewer launched from an implementer's shell would inherit the implementer's
id and fail its own gate. Also called *run*, *execution*, *session id*.
*SPEC §35, ADR-054, ADR-131.*

**Session.** An attempt as seen from the outside: one agent process from
launch to exit. The launcher (a shell, a supervisor, `axial attempt start --
<cmd>`) is the session boundary. *ADR-131.*

**Execution receipt.** A record of what one attempt ran: role, runtime, model,
authenticating principal, a closed set of parameters. The supervised loop writes
one per attempt; an unsupervised session writes its own with
`axial attempt register` or the `execution_register` tool, and neither surface
can write an **observed** model.

Mostly provenance for audit. One field is a predicate input and only one: a
`different-model` gate reads the canonical observed model, and nothing else on
the receipt is ever compared. *SPEC §31 and §35, ADR-124 §5 as amended by
ADR-136.*

**Human login.** `axial login --github` gives a person a 30-day credential that
is theirs, not an agent's. It runs only on an interactive terminal, and the dev
loop refuses to launch workers under it. *SPEC §32, ADR-133.*

**Checkout declaration.** The tracked file `.axial/config.toml` in a
repository. It may say *where the tracker is* (`db_url`, `workspace`) and
nothing else: no actor, no attempt, no credential. `axial config` shows what
resolved and from where; `axial doctor` checks it. *ADR-091.*

---

## Review

**Review gate.** A requirement, recorded on an issue by `axial issue review`,
that an independent review happens before the issue can close. The gate
carries a separation mode chosen when it was requested; it is never
re-evaluated under a later, different rule. Requesting a gate does not perform
a review. *SPEC §7, ADR-124.*

**Separation mode.** What the gate asserts about the review's independence.
Exactly five exist:

| Mode | Asserts | Selectable |
|---|---|---|
| `fresh-attempt` | the review ran in a different attempt than the implementation | yes, the default |
| `different-principal` | that, plus a different authenticated account is accountable | yes |
| `different-model` | that, plus a different **observed** model answered | yes |
| `different-runtime` | that, plus a different runtime answered | yes |
| `legacy-actor` | a pre-M78 gate: the actor strings differed | no, historical only |

`different-model` and `different-runtime` are independent: one runtime runs many
models and one model is reachable from several, so a different runtime never
satisfies a model requirement.

Only `different-principal` carries accountability. A different model is a second
opinion, not a second party — the same harness, the same accountable account —
and no mode claims otherwise. None of these says a human reviewed; that is
**human approval**, a separate requirement. *ADR-124 §2 to §4, ADR-136.*

**Observed model, asserted model.** What a receipt says ran, split by who said
it. The **asserted** model is what a session claimed or asked for; it is
provenance and routing input, shown and exported, and it never satisfies a gate,
because a value a session chooses about itself cannot constrain that session. The
**observed** model is what adapter evidence showed actually answered, and it is
the only one `different-model` compares. A session cannot write an observed model
on any surface. Absent or ambiguous is **unknown**, and unknown is not diverse —
two unknowns are not two models. *ADR-136 §3 to §5.*

**Human approval.** An explicit, recorded approval of an issue by a human
principal (`axial issue approve`), required when the gate was requested with
`--human-approval`. Login alone, or possessing a human token, is not approval.
*ADR-134.*

**Reviewer.** Whoever satisfies the gate: a second attempt, a second
principal, or the person approving. In the dev loop the reviewer is a
supervised attempt on the runtime that did not implement the change.
*ADR-111 §1.*

**Verdict.** The reviewer's recorded result: approve, or request changes with
findings (file and line, why it is wrong, the minimal fix). Stored as attempt
evidence and as an issue comment. *ADR-111 §2.*

**Remediation.** The bounded loop of fixing findings and re-reviewing. When it
runs out, the work is handed off, not force-closed. *ADR-111 §5.*

---

## The dev loop

Every name in this section refers to one feature, `axial dev`, or to a part of
it. Tenants have coined their own names for it; the aliases are listed so a
reader can map them.

**Dev loop.** The whole autonomous delivery feature: declare how a repository
runs agents, supervise attempts that implement issues, deliver pull requests,
route review, observe merges, close with evidence. It is the `axial dev`
command group. Also called *the supervised dev loop*, *the autonomous dev
loop*, and, in the Alcalytics repository, *the dev controller* or *the axial
dev controller*. *SPEC §25, ADR-088, ADR-092, ADR-093, ADR-109.*

**Dev-loop declaration.** The tracked file `.axial/dev.toml`: which project,
which runtime, how many workers and reviewers, where worktrees go, plus six
tables: `[routing]`, `[coordination]`, `[verify]`, `[review]`, `[merge]` and
`[validation]`. It
declares how the loop runs here and never who runs it. `axial dev init` writes
it; `axial dev doctor` validates it. *SPEC §25, ADR-092.*

**Supervisor.** The process that `axial dev start` or `axial dev run` becomes.
It holds the per-checkout lock, mints the attempt, allocates a worktree,
launches the worker, watches it, commits and pushes on the attempt branch,
opens the pull request, and releases the issue truthfully when the attempt
ends. There is one live supervisor per checkout. *ADR-093, ADR-109.*

**Worker.** The agent runtime subprocess the supervisor launches to implement
one issue, in its own worktree, under a fixed actor and attempt. A worker has
no delivery credential: it cannot push, open a pull request, merge, or close
its own issue. Also called *the agent*, *the implementer*. *ADR-093, ADR-109.*

**Runtime and adapter.** The runtime is the agent product a worker runs on
(`claude`, `codex`, `pi`). The adapter is axial's code that speaks to that
runtime. `[routing] runtimes` in `dev.toml` lists the ones this repository
allows. *ADR-095, ADR-111, ADR-122.*

**`axial dev start`.** Supervise exactly one attempt in the foreground, deliver
it, and exit. Its stdout and exit codes are per-attempt public API.
*ADR-093 §1.*

**`axial dev run`.** The outer delivery controller: everything `dev start`
does, repeated over the frontier, plus review dispatch, merge observation and
closure, until told to stop. Also called *the controller*, *the outer
controller*, *the delivery controller*, and, in Alcalytics, *the dev
controller*. `dev run` and `dev start` share one lock, so running both on one
checkout is a conflict. *ADR-109.*

**`axial dev review`.** Review delivered work on the runtime that did not
write it, and drive remediation. **`axial dev close`.** Prove what merged and
close the issue against that commit. `dev run` calls both. *ADR-110, ADR-111.*

**Delivery.** The supervisor's act of turning a finished attempt into a pull
request: commit on the attempt branch, push, open one non-draft pull request,
comment the URL. Delivery is the supervisor's, never the worker's.
*ADR-109 context.*

**Worktree.** The private git checkout each attempt works in, under the path
`dev.toml` names. One checkout has one writer at a time; a worktree is how two
attempts avoid staging each other's changes. *AGENTS.md §Dogfooding.*

**Park.** Leaving an issue on the frontier without starting it, because
something about it needs a person or a reset first. A parked issue is not
failed and not lost; it waits. **Quarantine.** The state of an attempt whose
failure the loop will not retry on its own; a quarantined attempt is parked for
a human decision. *SPEC §25.*

**Merge automation.** The optional `[merge]` table that lets the loop merge
when the provider's checks pass for the exact head. Absent by default; a
repository without observable check contexts leaves merge to a person or CI.
*SPEC §25.*

**Verify grant.** `[verify] commands`: the exact local command prefixes a
worker may run to check its own work. Never includes `git push`, `gh`, deploy
or credentialed commands; those are refused at parse time. *SPEC §25.*

**Role profile.** The kind of session a slice needs, named without a vendor
model: `planning`, `implementation-standard`, `implementation-high-risk`,
`review`. `axial dev resolve` shows which model a profile resolves to here.
*ROLE-PROFILES.md.*

**Risk tier.** How much an error would cost, set by whoever authorizes the
slice: `low`, `standard`, `high`, `critical`. It sets the review expectation
and the evidence a profile must show. *ROLE-PROFILES.md.*

**Production lead.** Not a command. The role, human or model, that sits
outside the loop and watches it: reads delivered pull requests, answers
decisions, records the canonical PR comment and reference, and does the
high-risk closes a repository reserves for a separate actor. A lead that polls
with `sleep` is the defect ADR-126 fixes. Also called *the lead*, *the outer
lead*. *ADR-126.*

---

## How axial talks

**Agent mode and human mode.** Every command prints a compact text contract
for agents on stdout; `axial ui` is the one interactive, human-facing surface.
There is no separate "machine" flag. *SPEC §1.*

**axi/1.** The name and version of that stdout/stderr/exit-code contract.
`axial conventions` prints it. **TOON** is the name of its line format
(`PROJ-14 [in_progress] p:high @codex #backend: Title`). *SPEC §1 to §3.*

**Exit codes.** `0` ok (including empty), `1` runtime, `2` usage, `3` not
found, `4` conflict (a claim taken or a guard failed), `5` timeout. Errors are
one line on stderr: `error[CODE]: message`. *SPEC §2, §3.*

**Guard.** An `--expect-…` flag that makes a mutation conditional on the state
you last saw (`--expect-status todo`, `--expect-assignee claude`). A failed
guard is exit 4, and nothing was written. *SPEC §7.*

**Idempotency key.** A caller-chosen string on a mutation so a lost response
can be retried safely: the same key replays the same result instead of
creating twice. Use it once per distinct operation. *SPEC §7.*

**MCP.** The tracker operations exposed as tools to a runtime that speaks the
Model Context Protocol, via `axial mcp`. One process, one fixed actor, attempt
and workspace. `config`, `doctor`, `verify`, `sync-git`, `watch`, `export` and
`import` stay CLI-only. *ADR-022.*

---

## Words from axial's own development

These appear throughout docs/ because axial tracks itself. They describe how
axial is built, not how you use it.

**Milestone (Mnn) and slice (Sn).** axial's own roadmap is cut into numbered
milestones (M58, M78, …), each into slices (S1, S2, …). `M78/S3` is the third
slice of milestone 78. Unrelated to the `milestone` command a tenant uses.
*ROADMAP.md.*

**Promotion.** The act of an authority accepting a slice for implementation,
with a stated risk tier and evidence bar. A **promotion record**
(`docs/Mnn-PROMOTION-RECORD.md`) is the written result. Distinct from
`axial promote`, which moves a local workspace into the cloud backend; the two
share a word and nothing else. *AUTONOMOUS-RUNTIME-PROMOTION-GATE.md.*

**Packet.** The bounded brief a worker or reviewer receives for one slice:
scope, the ADR and SPEC references, what is out of bounds. Never the
implementer's conversation history. *docs/prompts/implementation-packet.md.*

**ADR.** An architecture decision record in `docs/adr/`, numbered by
`axial alloc next adr`. Accepted ADRs are binding on the code; proposed ones
are not.

**Dogfood.** axial tracking its own development in the `axial` workspace.
`DOGFOOD.md` is the operating manual for that. A *dogfood shortcoming* is
friction with axial itself, filed as its own issue, never as a child of the
issue that hit it.

---

## Aliases at a glance

| You may read | It means |
|---|---|
| dev controller, axial dev controller | `axial dev run`, or the dev loop as a whole |
| supervised dev loop, autonomous dev loop | the dev loop |
| controller, outer controller, delivery controller | `axial dev run` |
| the agent, the implementer | the worker |
| run, execution, session id | the attempt |
| tenant | the workspace |
| account | the principal |
| receipt | the execution receipt |
| the lead, the outer lead | the production lead |
| the frontier | the dependency-ready frontier |
| AC | acceptance criterion |
| `axial promote` | moving a local workspace to the cloud, not a promotion record |
