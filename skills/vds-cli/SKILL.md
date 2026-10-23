---
name: vds-cli
description: >
  Operate or diagnose a VDS organization through vds-cli: inspect resources,
  manage authorized changes (including connecting a custom MCP server as a
  connector), run agent chats, retrieve deliverables, or resolve authentication
  and organization selection. Use for actual terminal/CLI work, not general VDS
  feature or UI questions (use vd-product-guide for those).
license: MIT
---

# vds-cli

`vds-cli` builds itself from the server's capability catalog: five built-ins —
`auth`, `orgs`, `config`, `introspect`, `call` — plus one command per operation the
server offers. A new endpoint becomes a new command with no CLI release.

**So do not memorise commands, and do not expect this document to list them.** The
CLI is its own reference, and it is a good one:

| Question | Ask the CLI |
|---|---|
| What can this server do? | `vds introspect`, narrowed by `--tag <family>` or `--q <text>` |
| How do I call one operation? | `vds introspect <operationId>`, or `--json` for the full schema |
| What flags does a command take? | `vds <tag> <action> --help` |
| Am I allowed to? | `vds introspect --org <orgId>` |
| Output formats, exit codes, headless login | `vds --help` documents all three |

`vds --help` already carries the exit-code table, the machine-readable output flags,
and the re-entrant login loop, so read it rather than guessing. If a command seems
missing, run `vds introspect --refresh` and work from the live list — never hand-roll
HTTP against `/openapi/*` to get around it.

Everything below is only what the CLI **cannot** tell you.

## Step 0 — Get the right CLI

There is usually no `vds` binary installed. Invoke it through `npx`, keeping the
explicit `@latest` — without it npx silently reuses a cached or globally installed
older copy:

```bash
npx --yes @vibedecision/vds-cli@latest --version
```

For subsequent examples, replace `vds` with the full `npx --yes @vibedecision/vds-cli@latest` invocation when no binary is installed. Do not store the command and its arguments in a scalar shell variable: argument splitting differs between bash and zsh. If `vds` is already on `PATH`, use it and skip `npx`.

- **npx missing, or Node < 22** → ask the user to install Node.js >= 22
  (<https://nodejs.org>) and stop. Nothing else works until that is fixed.
- **A command fails with `client_too_old` (exit 8)** → this CLI predates what the
  server accepts, and the error already names the version to move to. Run
  `vds upgrade` (or re-invoke with an explicit `@latest`) and retry. Do not improvise
  a fallback — not the older hand-written `vds skills` verbs, not `curl`. There is no
  version number to memorize here: the server states its own minimum, so `vds upgrade`
  and the error text are always current and this file cannot go stale.

## Step 1 — Which server

Decide from the user's own words, before running any login:

| Signal | Target |
|---|---|
| An explicit `http://` / `https://` URL | that URL |
| They say it is local — "local", "localhost", "on my machine", "port 3000" | `http://localhost:3000` |
| They name a remote environment — "staging", "test", "prod" — but give no URL | ask for the URL; never guess a hostname |
| No signal, and credentials already exist | keep the current `baseUrl`; do not switch |
| No signal, no credentials | ask: local · a custom URL · `https://vibedecision.com` |

Match on meaning rather than those exact words. Users write in whatever language they
think in, so treat any equivalent phrasing in any language as the same signal.

## Step 2 — Authenticate

Start with `vds auth status --json`. Exit **3** means "no usable credential" — that is
the command working correctly, not a tool failure.

If a login is needed, run it yourself. Never ask the user to run the command, and
never ask them for a token, key or password: the device flow exists so that no secret
passes through the conversation. `vds --help` gives the exact `--no-wait` polling
loop; what it does not tell you is what goes wrong:

- **Re-run the identical command to poll. Never start a second `auth login`.** A fresh
  authorization *invalidates the code the user is at that moment looking at*, and the
  server allows only 10 starts per 5 minutes. A correct retry reports `"reused": true`
  with the same code. This is the most expensive mistake available here, and it fails
  invisibly — the user just sees a code that stops working.
- **Relay both the code and the device label, and let the user approve.** Comparing
  the two against the page is what stops an approval being phished, so surface both
  every time and never approve on their behalf. Pass `--device-label "…"`: the
  hostname default means nothing to someone looking at an approval page for a
  container.
- **`source: agent-run`** means the platform wrote the CLI's config for this one
  conversation round, and marked it as its own with a platform-minted token. The server
  and organization are fixed — skip login and org selection entirely and go straight to
  the task.

## Step 3 — Organization, and its silent failures

`vds orgs list` gives ids, names and your role; `vds config` shows what resolved.
Two failures the CLI will not warn you about:

- **A wrong-but-valid org is silent.** The CLI refuses only when the org is
  *ambiguous*, never when you picked the wrong one — the command simply succeeds
  against the wrong organization, and nothing in the output reveals it. Confirm the
  org **and** the server before any write, and say both back to the user.
- **`orgs use` does not always stick.** Under `agent-run` it takes effect, but the
  platform rewrites that file every round, so it silently reverts on the next turn; with
  a credential from `VDS_CLI_TOKEN` and no config file it fails with a misleading "Not
  authenticated" even though you are authenticated. Prefer `--org <orgId>` per
  command, or `VDS_ORG_ID`.

## Read-only mode

`vds config` and `vds auth status` report `readOnly`. When it is `true` — inside a VDS
chat that is the default, set by the user's vds-cli read-only switch — the CLI offers
only the operations the server marks read-only:

- `vds introspect` and `--help` list only those. The write commands still exist on the
  server; they are withheld from this credential.
- Every write — a withheld command, `vds call <id>` of a write, raw `vds call
  --method/--path` — is refused with `read_only_mode` (exit 4) before anything is sent,
  and the server refuses the same credential on its own.
- Use it for what it is for: looking things up to finish the task. If the task needs a
  change, say exactly what you would change and let the user make it, or ask them to turn
  read-only mode off for their next message. Never edit or replace the CLI's config,
  switch credentials, or reach the API another way to get around it.

## Before you write anything

- Reuse explicit authorization already given for the operation and target. For an authorized routine create or update, resolve the exact server/org and proceed. Ask before a destructive action, broader sharing, or permission replacement whose resulting scope the user has not approved; first prepare the concrete before/after change. Do not request a second approval solely because the operation is a write.
- **Read the tier-2 schema first.** `vds introspect <operationId>` states semantics
  the flag list omits — for instance that a permissions update *replaces* the whole
  grant list rather than merging, so any grant you leave out is revoked.
- **Never edit an agent to make one chat turn work.** Everything a single run can need is a
  per-turn option on `threads runs-create` itself (`vds introspect threads.runs.create` lists
  them); none of it is ever saved to the agent (connectors you name do stay on that chat),
  and nothing has to be mounted on the agent first. `agents update` is a permanent change to
  a shared agent that every other chat and user inherits, so use it only when the user asked
  to change the agent. If a run rejects an id as *not available to you*, that is an access
  problem to report, not something to route around through the agent.
- `--scope GLOBAL` makes a skill platform-wide. Never pass it unless explicitly asked.
- Report the id and version after a successful create.
- **Never read, open or print a credential.** `vds config` masks the token and
  `vds auth status` reports the whole active identity, so every legitimate question is
  already answered — there is no reason to go hunting for credentials on disk, and
  doing so is not acceptable even to "check" something. If the user pastes a key or
  token into the conversation, tell them to revoke it and use `vds auth login` instead.

## Two traps

- **`base64` wraps at 76 columns** on both macOS and Linux, so a `--content-base64`
  payload needs `tr -d '\n'` or it arrives with embedded newlines. For anything but a
  tiny file, write the body to a file and pass `--data @payload.json` — a long blob
  can exceed the shell's argument limit.
- **A non-JSON response is written to a file, defaulting to the working directory.**
  Pass `--output <path>` whenever the user named a destination, or a download lands in
  whatever directory you happened to be in — which is how they end up committed into
  someone's repository.

## Reference

`references/auth-and-orgs.md` — credential discovery order, the device-login state
machine, and the environment variables. Read it when a login or an org resolution
behaves in a way `vds auth status` does not explain.

`references/custom-mcp.md` — adding an MCP server that is not in the connector
catalog: choosing its transport, keeping its secrets out of the conversation, and the
create → test → fix loop. Read it before `connectors createCustomMcp`, and whenever a
custom MCP connector fails its test or its config needs changing.
