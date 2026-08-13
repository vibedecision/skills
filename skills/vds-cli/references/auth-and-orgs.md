# Credentials, environments and organizations

The parts of authentication and org resolution that `vds --help` and `vds auth status`
do not explain. For anything about *driving* operations, ask the CLI — `vds introspect`
and `vds <tag> <action> --help` are the reference.

## Where credentials come from

Resolved **per field**, so a partial source is topped up by a lower one rather than
discarded. Highest priority first:

1. **The agent-run credentials file** — written fresh each conversation round by the
   VDS platform when the CLI runs inside a managed sandbox. Short-lived (~30 minutes),
   never user-managed, and *not* the file a human's `vds auth login` writes.
2. **`VDS_CLI_TOKEN`**, then **`VDS_API_KEY`** (an alias kept because it is the name
   most likely already baked into CI).
3. **The login the CLI stores locally**, written by `vds auth login`. Treat it as
   opaque: `vds auth status` and `vds config` report everything about it that you need,
   and its contents are none of your business.

Base URL and organization resolve down the same chain, which is why an agent-run file
with a blank `orgId` still contributes its token while letting a lower source name the
org.

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

Every not-yet-authorized outcome exits 3, including the very first call. Exit 0 means
logged in and nothing else — which is what makes `until vds auth login --no-wait; do
sleep 5; done` correct.

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

When `source` is `agent-run`, the platform supplies credentials for that one
conversation round and **the server and organization are fixed**.

- `vds auth login` and `vds auth logout` **refuse** with `unsupported_operation`. Not a
  bug: the agent-run credentials outrank anything login would write, so the login would
  appear to succeed and change nothing — while clobbering the stored personal login of
  whoever later uses the CLI in that environment.
- `vds orgs use` writes that same outranked file, so it silently has no effect. The org
  looks unchanged on the next `vds config`, which is confusing if you do not know why.
- `vds auth status` still works, and is how you see the active identity.

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

Highest first: `--org` → the run's `orgId` (agent-run) → `VDS_ORG_ID` → the active org
stored by `vds orgs use` → the server's `active_org` → a sole membership.

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
| `VDS_BASE_URL` | Server to talk to, below agent-run and above the stored login |
| `VDS_API_URL` | Default server when nothing else names one |
| `VDS_ORG_ID` | Active organization, above the one set by `vds orgs use` |
| `VDS_OUTPUT_FORMAT` | `json` for every command in the session |
| `VDS_CONFIG_DIR` | Relocate the CLI's config directory — use it to keep a test run out of a real login |
