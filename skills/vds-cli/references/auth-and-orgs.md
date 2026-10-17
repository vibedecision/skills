# Credentials, environments and organizations

The parts of authentication and org resolution that `vds --help` and `vds auth status`
do not explain. For anything about *driving* operations, ask the CLI — `vds introspect`
and `vds <tag> <action> --help` are the reference.

## Where credentials come from

Resolved **per field**, so a partial source is topped up by a lower one rather than
discarded. Highest priority first:

1. **`VDS_CLI_TOKEN`**, then **`VDS_API_KEY`** (an alias kept because it is the name
   most likely already baked into CI).
2. **The login the CLI stores locally**, written by `vds auth login` — and, inside a
   managed sandbox, written fresh each conversation round by the VDS platform instead.
   Treat it as opaque either way: `vds auth status` and `vds config` report everything
   about it that you need, and its contents are none of your business.

There is no platform-only tier above these. What distinguishes a platform credential
from a human one is the **token**, not where it was found — see "Inside an agent
sandbox" below.

Base URL and organization resolve down the same chain, which is why a source that names
only a token still lets a lower one name the org.

**Discovery is not authentication.** A token found anywhere counts as logged in only
once `GET /v2/auth/whoami` accepts it — which is what `vds auth status` does, and why a
token in an environment variable can still report "not authenticated": the server
rejected it. `vds auth status --local` answers "what is configured?" without the
round-trip, reporting `authenticated: null` rather than guessing.

## The device-login state machine

`--no-wait` makes login re-entrant, which is what makes it usable from inside an agent
turn: the URL and code become the command's result on stdout instead of being printed
to stderr behind a ten-minute block.

- **Not yet approved** → exit **3**, with `{"data":{"status":"pending","userCode":…}}`.
- **Approved** → exit **0**, and the identity is persisted.

A pending authorization exits 3, including the first call. Retry the identical login command only while its response reports pending. Stop on success, denial, or another error; do not use an unconditional `until` loop that retries every failure.

Outcomes that are **not** "keep waiting":

- **Denied** → `precondition_failed`, and the pending state is cleared. A refusal is a
  decision, so it is not silently re-requested.
- **Expired** → checked against the server first, because a stored `expiresAt` is only
  a local clock reading and a suspended laptop must not be able to declare a live
  authorization dead. A token that still works is kept.
- **Abandoning one deliberately** (wrong server, user walked away) → `vds auth logout`
  clears it.

A server outage never destroys a credential: only a definitive 4xx marks a token
unusable.

### Why there is no `--scope`

An authorized device may call every endpoint. What it may *do* is decided per request
from the user's own role and the resource ACLs, so there is no narrower grant to
request. A 403 therefore means *the user* lacks permission, not that the credential is
too narrow — and `vds introspect --org <orgId>` tells you which it is before you try,
keeping the two apart because the remedies differ: a missing role capability needs an
org admin, while a missing key scope needs a different key.

## Inside an agent sandbox

When `source` is `agent-run`, the platform wrote the config file for that one
conversation round and **the server and organization are fixed**. The marker is the
token: only the VDS server can mint an `art_` token, while `vds auth login` always
generates a `vdt_` one locally. So `agent-run` means "these credentials are not yours to
change", regardless of which file or variable they arrived in.

- `vds auth login` and `vds auth logout` **refuse** with `unsupported_operation`. Not a
  bug, and not a no-op either: they write the very file the platform wrote, so a login
  would swap this round's identity for an unrelated one and act as the wrong user until
  the next round overwrote it back, while a logout would revoke the round's credential
  outright.
- `vds orgs use` **does** take effect here — it edits that same file, preserving the
  token — but only until the next round rewrites it. Prefer `--org <orgId>` per command,
  or `VDS_ORG_ID`: a per-command override cannot be reverted underneath you.
- `vds auth status` still works, and is how you see the active identity.
- `readOnly: true` in that file is the platform's too: it is written each round the
  user's vds-cli read-only switch is on, and the token itself is minted read-only to
  match, so the server refuses writes whatever the file says. No CLI command clears it —
  `orgs use` keeps it, and login and logout refuse here anyway.

## Switching environments

Pass `--server` explicitly every time, including for the official cloud — it is the
only thing that makes the target unambiguous in a transcript. The CLI stores one login
at a time, so switching **replaces** the previous one and going back means logging in to
that server again.

No `logout` first: a pending authorization for a different host is discarded
automatically, and trailing slashes and host casing are normalised so they do not read
as a different server. Each server keeps its own catalog cache, so switching does not
force a `--refresh`.

## Organization precedence

Highest first: `--org` → `VDS_ORG_ID` → the active org stored in the config file (by
`vds orgs use`, or by the platform inside a sandbox) → the server's `active_org` → a
sole membership.

Belong to several with none selected and the CLI **refuses rather than guessing**,
listing the candidates in the error. That is the right outcome: picking one would
produce a command that succeeds against the wrong organization, which nothing in the
output would reveal.

`vds orgs use` also refuses an org you are not an active member of
(`org_membership_required`, exit 4) and names the ones you can use — so it is safe to
attempt, and its error doubles as a listing.

## Environment variables

| Variable | Effect |
|---|---|
| `VDS_CLI_TOKEN` | Bearer token; outranks the stored login |
| `VDS_API_KEY` | Alias for the above, kept for CI |
| `VDS_BASE_URL` | Server to talk to, above the stored login |
| `VDS_API_URL` | Default server when nothing else names one |
| `VDS_ORG_ID` | Active organization, above the one set by `vds orgs use` |
| `VDS_OUTPUT_FORMAT` | `json` for every command in the session |
| `VDS_CONFIG_DIR` | Relocate the CLI's config directory — use it to keep a test run out of a real login |
