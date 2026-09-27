# How to add a teammate to an axial organization

This page is for two people: the **admin** who already runs an organization on
the hosted service, and the **teammate** joining it, who may be starting from
nothing on a Mac. Follow it in order. At the end the teammate can sign in, see
the project, close tickets as themselves, and run agents that write as agents.

Words in *italics* on first use are in the [glossary](GLOSSARY.md).

> Requires an axial that stores a person's and an agent's credential apart
> (ADR-138). Check with `axial config`: it prints a `credential_kind:` line.

---

## The shape you are building

A teammate is two identities, and they must stay apart:

| Who | Credential | Comes from | Used by |
|---|---|---|---|
| the teammate | a personal *session* | `axial login --github` | their own shell, the web page |
| their agents | an agent *service token* | the account page | the agent's own process only |

Keeping them apart is what makes closing tickets painless. The teammate's
approvals and closes are recorded as a person, which is what review gates ask
for; the agents' work is recorded as agents, and an agent can never approve its
own work as the person.

Give each person's agents their own name: `claude-dana`, `codex-dana`. Each
agent name issued from the account page is its own account (ADR-139), so a
token can only ever write as the name it was issued for. But the name is the
agent: two people's agents both called `claude` are one identity to the
tracker, and a claim by one reads to the other as `already claimed by claude`.

A token the page marks **shared agent account** was issued before each agent
had its own account. It still works; to separate it, issue a new token for the
same name and revoke the old one.

---

## Admin: invite them

1. Open **https://axial.airforge.io/account**.
2. Under **Invites**, enter their **GitHub username**, choose a role, and
   choose the **Workspace** they will work in (it lists the workspaces you
   are in; the one you have selected comes first). Accepting the invite
   gives them that workspace, and the message below names it.
   Choose **Admin** if they should issue their own agent tokens; a **Member**
   who opens the token page gets `managing tokens requires an administrator`.
3. Send them the message the page shows. Nothing is emailed, so the page
   hands you the text instead: how to install axial, sign in with
   `axial login --github`, and check with `axial status`, with your
   organization and service address filled in. **Copy message** copies it; it
   stays under **Message** on the invite's row for as long as the invite is
   pending.
4. Invite them **before** they sign in. Someone who signs in before an
   invite exists is asked whether they are joining a team or starting their
   own organization; if they choose to wait, your invite joins them at their
   next sign-in. Someone who already has an account joins when they next sign
   in with GitHub or approve an `axial login --github` code. The username is
   matched regardless of capitals. (An account last signed in before v0.70.1
   needs one fresh GitHub sign-in first: sign out at `/auth/logout`, then sign
   in again.)
5. The invite's **Status** says `waiting: @<username> hasn't signed in yet`
   until they sign in, then `joined <time>`. Only a waiting invite can be
   cancelled.

Invite **before** they first sign in. If they sign in first, axial asks whether
they are joining a team or starting their own organization; once you have sent
the invite they choose **Check again** (or simply sign in again) and join yours.
If they chose to start their own by mistake, they can delete that empty
organization from their account page (**Delete this organization**).

Do not create them with `axial admin principal create --kind human`: a
principal made that way has no GitHub identity, so it can never sign in. The
web invite is the only path that produces a person who can.

---

## Teammate: install axial (macOS)

No GitHub access is needed; the release binaries are public:

```bash
curl -fsSL https://github.com/mjanveaux/axial-releases/releases/latest/download/install.sh | sh
axial version
```

It installs to `~/.local/bin/axial`; if that folder is not on your `PATH`, the
installer prints the line to add. Run the same command to upgrade. Builds exist
for Apple silicon only; on an Intel Mac, build from source (README §Install;
requires Go and read access to the private repository).

---

## Teammate: sign in as yourself

1. Open **https://axial.airforge.io/** and choose **Sign in with GitHub**. The
   invite puts you in the admin's organization.
2. In a terminal:

   ```bash
   axial login --github
   ```

   It prints a URL and a code. Open the URL in the browser you just signed in
   with, type the code, and approve. Your session is saved to
   `~/.config/axial/config.toml`.

3. Do **not** set `AXIAL_ACTOR` in your shell profile. Your session already
   knows who you are, and an `AXIAL_ACTOR` meant for an agent would make your
   own commands claim to be the agent.

---

## Teammate: give your agents their own credential

1. On **https://axial.airforge.io/account**, issue an agent token named for
   your agent, e.g. `claude-dana`. It is shown once.
2. Save it beside your own login:

   ```bash
   axial login --token <the agent token>
   ```

   It is stored as `db_token_<workspace>.agent`, next to your session, and
   replaces nothing.

3. Tell each agent's session to use it, in that session's environment only.
   For Claude Code, add to `~/.claude/settings.json`:

   ```json
   { "env": { "AXIAL_CREDENTIAL": "agent", "AXIAL_ACTOR": "claude-dana" } }
   ```

   Neither value is a secret; the token stays in the config file. `axial mcp`
   and `axial dev` pick the agent credential on their own.

---

## Teammate: check it

In your own shell:

```bash
axial config    # credential_kind: human (default)
axial whoami    # kind: human, and your own name
axial doctor    # every line ok; actor: ok
```

In an agent session (or `AXIAL_CREDENTIAL=agent axial whoami`):

```bash
axial whoami    # kind: service, actors: claude-dana
```

If `doctor` reports `actor: refused`, the credential cannot act as the name
you set: set `AXIAL_ACTOR` to one of the names it lists, or unset it.

---

## Working a ticket, and closing it

- Your agents claim and implement: `axial next --project APP --lease 2h`.
- You review: `axial context APP-42 --full`.
- You approve and close as yourself:

  ```bash
  axial issue approve APP-42
  AXIAL_ATTEMPT=$(axial attempt start --role review) \
    axial issue close APP-42 --expect-status in_progress --from-merge
  ```

  The `attempt start` wrapper marks the close as its own review run, which a
  `fresh-attempt` gate requires. Until AXIAL-907 lands, a person's close does
  not get one automatically, and the web page's close button cannot satisfy
  such a gate — close gated issues from the terminal.

For gated issues, ask the admin to use `--separation fresh-attempt
--human-approval`. Avoid `different-principal` while agent tokens share one
agent account (AXIAL-909): it cannot tell two agents apart.

---

## When something is off

| You see | It means | Do |
|---|---|---|
| `managing tokens requires an administrator` | you were invited as Member | ask the admin to change your role, or to issue the token |
| `not permitted to act as "…"` | your credential and your `AXIAL_ACTOR` disagree | `axial whoami` lists who you may act as; pass `--actor <one of them>` on the command, or unset `AXIAL_ACTOR` |
| `requires review by a different attempt` on close | the close carried no review run | wrap the close in `AXIAL_ATTEMPT=$(axial attempt start --role review)` as above |
| `credential_kind: agent (…; none stored …)` | an agent session found no agent credential | run step 2 of *give your agents their own credential* |
| `approve` refused as a service account | you ran it with an agent's credential | run it in your own shell, without `AXIAL_CREDENTIAL=agent` |
