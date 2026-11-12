# Guided trial: see axial stop four agent failures

About 15 minutes, one machine, no account. You will make a tiny work graph,
then watch axial refuse the four mistakes that make agent fleets lose work:

1. two agents take the same task (double claim);
2. an agent dies and its task is stuck forever (stale lease);
3. a new agent starts a task with no idea what the last one did (lost handoff);
4. a task is called "shipped" with nothing to show for it (unsupported closure).

Every output below was captured from a real run. Your timestamps will differ.

## 1. Install and point at a scratch database

Install the `axial` binary (see the README; on a
Mac see [Add a teammate](add-a-teammate.md)). Then use a throwaway database so
nothing you do here touches real work:

```bash
export AXIAL_DB_URL="file:$(mktemp -d)/trial.db"
export AXIAL_ACTOR=alice          # "alice" is the first agent
axial doctor                      # config, database and schema should all pass
```

## 2. Make a small work graph

Three issues, and one dependency: the docs wait for the feature.

```bash
axial project create TRIAL "Trial project"
axial issue create TRIAL "Add rate limiting" -p high
axial issue create TRIAL "Write the docs" -p medium
axial issue create TRIAL "Fix typos" -p low
axial issue block TRIAL-2 --by TRIAL-1
```

```
created project TRIAL: Trial project
created TRIAL-1 [todo] p:high: Add rate limiting
created TRIAL-2 [todo] p:medium: Write the docs
created TRIAL-3 [todo] p:low: Fix typos
linked TRIAL-2 blocked-by TRIAL-1
```

TRIAL-2 is now blocked. `axial next` will skip it until TRIAL-1 is done (you
will see this in step 4 and step 7).

Already have issues in Linear or GitHub? `axial import` brings them in from a
JSONL export, for example
`axial import --from github --project TRIAL issues.jsonl`. See
Provider interop. You do not need it for this trial.

## 3. Connect two agents

Each agent is its own session with its own name (`AXIAL_ACTOR`) and the same
scratch database. Use a separate terminal per agent and export the same
`AXIAL_DB_URL` in each.

An agent that runs shell commands (any CLI coding agent) needs only the two
variables. Start it from a terminal where they are set:

```bash
export AXIAL_DB_URL="<the same file: URL as step 1>"
export AXIAL_ACTOR=alice
claude        # or codex, or any agent that can run shell commands
```

An agent that speaks MCP gets `axial mcp` as its server, with the actor and
database fixed for that server. For Claude Code:

```bash
claude mcp add axial-alice -e AXIAL_DB_URL="$AXIAL_DB_URL" -e AXIAL_ACTOR=alice -- axial mcp
claude mcp add axial-bob   -e AXIAL_DB_URL="$AXIAL_DB_URL" -e AXIAL_ACTOR=bob   -- axial mcp
```

See Quickstart for the MCP
details. Then give each agent a task, for example to alice: "Use axial to claim
TRIAL-1 with a 2h lease and work on it." and to bob: "Use axial next in project
TRIAL with a 2h lease and work on what you get."

To run bob as a real second agent, open another terminal and start him as his
own session, the same way as alice but with his own name:

```bash
export AXIAL_DB_URL="<the same file: URL as step 1>"
export AXIAL_ACTOR=bob
claude        # or codex, or any agent that can run shell commands
```

With MCP, start a Claude Code session that uses the `axial-bob` server (and
alice's session the `axial-alice` one). In steps 4 to 7, bob's lines are what
his agent does: ask it to run `axial next` and `axial context`, as in the
prompts above.

Optional shortcut: if you only want to see the behaviour without running agents,
one terminal is enough. Run a command as bob by putting `AXIAL_ACTOR=bob` in
front of it, as the commands below do.

## 4. Failure prevented: the double claim

Alice takes TRIAL-1:

```bash
axial issue claim TRIAL-1 --lease 2h
```

```
claimed TRIAL-1 by alice (lease expires 2026-11-11T15:01:20Z)
```

Bob tries the same task:

```bash
AXIAL_ACTOR=bob axial issue claim TRIAL-1 --lease 2h
echo $?
```

```
error[CONFLICT]: TRIAL-1 already claimed by alice (lease expires 2026-11-11T15:01:20Z)
4
```

The claim is a single atomic step, so two agents racing for one task cannot
both win. Exit code 4 means "conflict", and an agent can branch on it.

Better: bob does not have to pick a task at all. `axial next` hands him the best
task nobody holds. TRIAL-1 is held and TRIAL-2 is blocked by it, so bob gets
TRIAL-3. This claim uses a 2 second lease on purpose, for the next step:

```bash
AXIAL_ACTOR=bob axial next --project TRIAL --lease 2s
```

```
claimed TRIAL-3 [todo] p:low @bob: Fix typos (lease expires 2026-11-11T13:01:22Z)
```

## 5. Failure prevented: the stale lease

A claim carries a lease. Pretend bob's agent just died. Wait for its 2 second
lease to run out, and alice can take TRIAL-3:

```bash
sleep 3
axial issue claim TRIAL-3 --lease 2h
```

```
claimed TRIAL-3 by alice (lease expires 2026-11-11T15:01:23Z)
```

Without a lease, a crashed agent would hold the task until a person noticed.
With one, the task is available again on its own. Always claim with `--lease`.

## 6. Failure prevented: the lost handoff

Alice still holds TRIAL-1. When she stops early, she says what she did and what
is left:

```bash
axial issue release TRIAL-1 \
  --done "wrote the limiter" \
  --remaining "add tests" \
  --gotchas "config.toml is read before env"
```

The next agent, bob, starts from one read:

```bash
AXIAL_ACTOR=bob axial next --project TRIAL --lease 2h
AXIAL_ACTOR=bob axial context TRIAL-1 --full
```

```
claimed TRIAL-1 [todo] p:high @bob: Add rate limiting (lease expires 2026-11-11T15:01:23Z)
```

The context output includes:

```
handoff_by: alice
handoff_at: 2026-11-11T13:01:23Z
handoff_done: wrote the limiter
handoff_remaining: add tests
handoff_gotchas: config.toml is read before env
```

Bob did not need alice's chat history, only the tracker.

## 7. Failure prevented: the unsupported closure

By default, axial lets an agent close a task as shipped with no proof, but marks
it `unverified` for a person to see. Switch the workspace to strict and the
same close is refused:

```bash
axial close-policy strict
AXIAL_ACTOR=bob axial issue close TRIAL-1 --outcome shipped
```

```
error[CONFLICT]: shipped requires grounding — attach --evidence (pr:/commit:/url:), satisfy a close criterion, or run: axial verify TRIAL-1
```

Now make a real artifact. Bob writes a small change in a scratch git repository,
commits it, and closes with that commit. The author identity is set for this
repository only, so it works on a machine where Git is not configured yet:

```bash
cd "$(mktemp -d)" && git init -q
git config user.name bob
git config user.email bob@example.com
printf 'limit = 100  # requests per minute\n' > rate_limit.conf
git add rate_limit.conf
git commit -q -m "Add rate limiting"
AXIAL_ACTOR=bob axial issue close TRIAL-1 --outcome shipped --evidence "commit:$(git rev-parse --short HEAD)"
```

```
updated TRIAL-1 status: todo -> done
updated TRIAL-1 outcome: (none) -> shipped
added evidence to TRIAL-1: commit:5dcd091
```

Be clear about what strict mode checks: that a reference is attached, not that
the commit or pull request exists. It stops the lazy "done"; it does not
catch a made-up reference. `axial verify` and `--from-merge` check more.

Closing TRIAL-1 also unblocks TRIAL-2, so the dependency from step 2 now
clears:

```bash
AXIAL_ACTOR=bob axial next --project TRIAL --lease 2h
```

```
claimed TRIAL-2 [todo] p:medium @bob: Write the docs (lease expires 2026-11-11T15:01:23Z)
```

## 8. Look at what happened

```bash
axial status --project TRIAL   # counts, work in flight, what needs a person
axial log                 # who did what, newest first
```

Every claim, release and close above is in the log with the actor that did it.

## When you are done

Delete the scratch directory. To keep going for real, unset `AXIAL_DB_URL` and
follow the README quickstart. When a second
machine or a teammate needs the same tracker, move to the hosted service with
the Quickstart.

## Running this as a facilitator

Watch for three things and write them down: where the person got stuck, which
of the four failures they said they had hit before, and whether they came back
the next day. Those answer the trial's one question: does seeing these
refusals make someone want axial on their own work?
