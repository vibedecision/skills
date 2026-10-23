# Creating a custom MCP connector

Use this when the user wants VDS agents to use an MCP server that is not in the
connector catalog. A custom MCP connector holds one MCP server, given as the standard
`mcpServers` config most MCP server READMEs show.

The live contract is the reference: `vds introspect connectors.custom-mcp.create`, plus
`--json` for every field of `CustomMcpServer`. When it disagrees with this file, the
contract wins. This file covers what the contract cannot: choosing the transport,
handling the server's secrets, and the create → test → fix loop.

## Before you start

- **Read-only mode must be off** (see "Read-only mode" in SKILL.md). Creating, testing
  and updating are all writes — `connectors test` too, because it records its result on
  the connector.
- **Confirm the server and the org** as SKILL.md describes, and say both back to the
  user before creating anything.
- **Check that custom MCP is switched on:** `vds connectors manifests-list --search
  custommcp`. `available: false` means a platform administrator has turned it off (the
  `custom_mcp_connector_enabled` system setting). Stop and tell the user; there is no
  workaround.

## 1. Pick the transport

| The server's docs show | Server entry |
|---|---|
| A URL — Streamable HTTP, usually ending in `/mcp` | `{"url": "https://…/mcp", "headers": {…}}` |
| An `npx …` command | `{"command": "npx", "args": ["-y", "<package>", …], "env": {…}}` |
| A `uvx …` command | `{"command": "uvx", "args": ["<package>", …], "env": {…}}` |

Do not create a custom MCP connector — tell the user why instead — when the server:

- **is SSE-only** (`"type": "sse"`, a URL ending in `/sse`). Look for its Streamable HTTP
  endpoint first; many servers offer both.
- **needs an OAuth sign-in.** That is the separate Generic MCP (OAuth 2.1) connector type
  (`generic-mcp-oauth`), which needs the browser authorization flow, so the user creates
  it in the web app.
- **starts with anything other than `npx` or `uvx`** — `node`, `python`, `docker`, a
  local binary.

One connector holds exactly one server; for several servers, create several connectors.
Keys VDS does not use (`disabled`, `cwd`, `timeout`, `autoApprove`, …) are rejected, so
remove them rather than passing a client config through unchanged.

## 2. Secrets

- **Header and env values are stored as given, unencrypted, and anyone who can read the
  connector sees them** — in `connectors get` and `connectors list`, and in the response
  to every create and update. A new connector follows the organization's Sharing Mode,
  which by default makes every member a manager of it: able to read the secret, and to
  edit, delete and share the connector. Tell the user this before storing a key.
- **To keep a secret private, add it last.** Create the connector without it (an HTTP
  server without its `Authorization` header, a stdio server without that env variable),
  narrow access with `connectors permissions-update`, then add the secret with
  `connectors update` (§5). Get approval for the resulting scope first, as SKILL.md
  requires for any permission replacement.
- **Never put a placeholder where a value belongs** — `${API_KEY}`, `{{token}}`,
  `<your key>`, `YOUR_API_KEY`. Nothing fills them in: the API rejects the `${…}`,
  `{{…}}` and `{NAME}` forms, and the others you have to catch yourself.
- **Watch for `$` in stdio values.** MCPHub, which runs stdio servers, expands `$NAME`
  in `args` and `env` values — from the server's own `env`, else its own environment,
  else empty — so `Pa$Word1` reaches the server as `Paord1`. A secret containing a `$`
  followed by a capital letter or `_` cannot be passed to a stdio server as it is; tell
  the user.
- **Keep the secret out of the conversation.** Have the user put it in an environment
  variable or a file they control, build the payload from that (§3), and print nothing
  that contains it: use `--quiet` on create and update, which prints only the id, and
  never print an unfiltered `connectors get`. Inside a VDS chat the user has no shell to
  set a variable, and anything they type stays in the chat history — say so, and
  suggest they enter the secret themselves on the connector's Edit page in the web app.

## 3. Create

Write the body to a file and pass it with `--data` — the nested map is awkward as a
flag, and a secret on the command line ends up in shell history.

A Streamable HTTP server, with its token read from an environment variable the user set:

```bash
payload=$(mktemp)
jq -n --arg token "$DOCS_MCP_TOKEN" '{
  name: "Docs Search",
  description: "Searches the internal docs",
  mcpServers: { docs: {
    url: "https://mcp.example.com/mcp",
    headers: { Authorization: ("Bearer " + $token) }
  } }
}' > "$payload"
vds connectors custom-mcp-create --data @"$payload" --quiet
rm -f "$payload"
```

`--quiet` prints only the new connector's id. `--json`, or the default table, would print
the whole `mcpConfig`, token included.

A stdio server needs no secret here:

```json
{
  "name": "Fetch",
  "description": "Fetches web pages as markdown",
  "mcpServers": { "fetch": { "command": "uvx", "args": ["mcp-server-fetch"] } }
}
```

- `name` must start with a letter, digit or Chinese character and may contain only
  those plus spaces, `_` and `-`, and it must be unique in the organization. A 409 means
  it is taken — possibly by an archived or deleted connector, or by one you cannot read,
  none of which `connectors list` shows. Check `vds connectors list --search "<name>"`;
  if it is not there, choose another name.
- `icon` is optional: a `data:image/png;base64,…` (or `image/jpeg`, `image/svg+xml`)
  data URL of at most 5 MB. Build it into the payload file; it is far too long for a
  flag.
- Report the new connector's `id`.

## 4. Test

```bash
vds connectors test <connectorId> --json
```

- **Read `data.success`. A failed test still exits 0** — the test ran; the connection
  did not work. The result carries no secrets, so `--json` is fine here.
- **A pass proves less than it sounds.** MCPHub registers the server as a temporary
  copy, reads it back after about three seconds and removes it, so `success: true`
  means MCPHub accepted the config — not that the server connected or that its tools
  work. A wrong endpoint path or a rejected token can still pass. The real check is a
  chat turn that uses the connector (§6): its tools only show up if the server works.
- The result is recorded on the connector (`testStatus`, `lastTestedAt`), where everyone
  who can read it sees it.

| `message` / `error` | Meaning |
|---|---|
| `Command not in (NPX/UVX)` | The launcher is not `npx` or `uvx` |
| `Args is empty` | `args` does not name the package to run |
| `Url is required` | The HTTP entry has a blank `url` |
| `Failed to start MCP server` | MCPHub refused to create the server from this config; `error` carries its reason |
| `Server test failed` | MCPHub created the server but could not report it back |
| `Connection test failed` | The test itself could not run — MCPHub unreachable or not configured, for instance; `error` says why |

The first three only appear on connectors made in the web app: the API rejects those
configs up front.

## 5. Fix the config

`mcpServers` replaces the whole server config, so build the patch from the current one.
Piping `connectors get` straight into the file keeps the secret off the screen:

```bash
patch=$(mktemp)
vds connectors get <connectorId> --json \
  | jq '{mcpServers: .data.mcpConfig.mcpServers} | .mcpServers.docs.url = "https://mcp.example.com/v2/mcp"' \
  > "$patch"
vds connectors update <connectorId> --data @"$patch" --quiet
rm -f "$patch"
```

- To look at the current config without its secrets:
  `vds connectors get <connectorId> --json | jq '.data.mcpConfig.mcpServers | map_values(del(.headers, .env))'`.
- A connector made in the web app with several servers must be cut down to one.
- Never send `mcpConfig` or `connectionParams` for a custom MCP connector; both are
  refused.
- An update does not reset the stored test status, so run `connectors test` again.
- A connector just created by mistake can be removed with `vds connectors delete
  <connectorId>`. That is a hard delete with no trash: say so and get confirmation first.

## 6. Use it

For a single chat turn, pass the id to the run:
`vds threads runs-create <threadId> --connector-ids '["<connectorId>"]'` (the flags are
in `vds introspect threads.runs.create`). As SKILL.md says, do not edit an agent to
make one turn work. Attach the connector to an agent only when the user asks for that.
