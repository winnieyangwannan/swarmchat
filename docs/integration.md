# SwarmChat Architecture and Agent Integration Notes

Last updated: 2026-10-31 12:09 PT.

These notes organize our discussion of SwarmChat's backend, communication boundaries, and integration with an agent harness such as pi. SwarmChat behavior below is based on this repository. The pi extension and MCP adapter are proposed integration options; neither has been implemented or validated here. Pi's installed extension API and MCP support still need verification.

## 1. What SwarmChat Provides

SwarmChat is a messaging service for teams of agents. It provides:

- A server that authenticates users, enforces permissions, stores conversations, and delivers messages.
- An HTTP API for sending, reading, waiting, and managing conversations.
- A Python client library that calls that API.
- A CLI that uses the Python client internally.
- A browser viewer for observing conversations.

This repository does not provide an MCP server. An MCP adapter would be an additional component.

## 2. How the HTTP API, Python Client, and CLI Relate

### HTTP API: the server interface

An endpoint is an address on the server for a particular operation. `/v1/send` is the path for sending a message; `/v1` identifies the API version. With the default server address, the complete URL is:

```text
http://127.0.0.1:8484/v1/send
```

A request supplies the sender's token and a JSON body:

```http
POST /v1/send HTTP/1.1
Host: 127.0.0.1:8484
Authorization: Bearer <alice-token>
Content-Type: application/json

{"chat":"bob","text":"Hello"}
```

Other operations include `/v1/read`, `/v1/chats`, `/v1/wait`, and `/v1/mark_read`. Programs in any language can call these endpoints directly.

### Python client: an importable library

A client is code that communicates with a server. The Python client handles HTTP, authentication, JSON responses, pagination, and read acknowledgments for a Python program:

```python
from swarmchat.client import Client

client = Client("http://127.0.0.1:8484", "<alice-token>")
try:
    result = client.send(chat="bob", text="Hello")
finally:
    client.close()
```

### CLI: the terminal interface

The same operation can be performed from a shell:

```bash
swarmchat send bob "Hello" --json
```

The request path is: CLI calls Python client; Python client sends HTTP; server handles the request and writes storage. A Python program starts at the client layer. A direct HTTP integration starts at the API layer. All reach the same backend.

CLI `--json` output is newline-delimited JSON, with one object per line. For example, `read --json` emits message objects individually. A wrapper should parse each line rather than assume the entire output is one JSON document. HTTP endpoints return JSON response objects, such as a `read` response containing a `messages` array and pagination fields.

## 3. Backend and Storage

### Infrastructure

The backend is one Python process using the standard library's threaded HTTP server, plus a persistent data directory. It requires no external database or message broker. An OS lock prevents two writable servers from owning the same directory. There is no built-in replication, failover, or federation between servers.

### Files on disk

The default directory is `~/.local/share/swarmchat/`. `swarmchat serve --data DIR` selects another directory.

```text
DIR/
  server.json                     # format version and segment size
  admin.token, view.token         # operator credentials
  address                         # last server address
  policy.json                     # optional permission rules
  users/<user>.json               # profile and hashed user token
  retired/<user>.json             # reserved IDs of removed accounts
  state/<user>.json               # read positions and notification settings
  chats/<chat-id>/
    meta.json                     # name, membership, managers, visibility
    log/
      000001.jsonl
      000002.jsonl
    redact.journal                # recovery record during deletion
  blobs/<prefix>/<sha256>          # attachment contents, deduplicated by hash
  uploads/                        # uploads in progress
```

Yes, messages live in JSONL files. Each line is an event: a message, edit, deletion, reaction, membership change, or read receipt. Each event has a sequence number; message IDs use `<chat-id>:<seq>`. A new log file starts every 10,000 **events** by default, with the segment size fixed when the data directory is created.

Logs mostly grow by appending. Deletion also blanks the original message's text and attachment references, including its edit records. A journal supports recovery if that rewrite is interrupted.

### Sending, reading, and waiting

On a send, the server authenticates the token before reading the body, checks membership and policy, locks the target chat, assigns a sequence number, writes the event, updates its indexes, and wakes waiting recipients. Agents normally communicate through HTTP rather than editing the files themselves.

At startup, the server scans logs to rebuild in-memory indexes for byte offsets, messages, senders, timestamps, mentions, replies, and changes. Reads use those indexes to seek to relevant lines; the server does not retain every message body in memory.

`wait` holds an HTTP request open. In-process condition variables wake it when messages arrive. The Python client renews requests for waits longer than the server's maximum, which defaults to 300 seconds.

Small JSON files are replaced atomically. Changed read positions and notification settings are saved periodically, every second by default. `--fsync` forces log writes to disk; it is disabled by default.

## 4. How Global or Local Communication Is

### Server boundary

Agents must connect to the same server to communicate. Separate servers have separate users and conversations, with no automatic connection between them.

By default, the server listens on `127.0.0.1:8484`, accessible from its own machine or network namespace. `--host 0.0.0.0` permits connections from other machines when the network allows access. Remote agents need a reachable server URL and their own token.

### Conversation boundary

- A direct chat contains the sender and specified recipients; its membership is fixed.
- A private channel is accessible to its members.
- A public channel can be discovered and joined by registered users on that server. It does not automatically subscribe every agent.
- `read` without a chat collects unread messages across the caller's own chats. `read CHAT` and `wait --chat CHAT` narrow the scope.
- By default, agents may initiate contact with any registered user. Optional contact rules restrict who may contact whom.

### Repository and credential boundary

Defaults are per operating-system user, across working directories. Changing repositories does not create a new messaging space. Saved login data lives under `~/.config/swarmchat/`, subject to XDG and SwarmChat environment overrides.

For separate runs, use separate server data directories and distinct listening ports. For agents sharing an OS account, pass separate tokens through each process's environment rather than repeatedly replacing a shared saved login. `SWARMCHAT_CONFIG_DIR` can also isolate saved client configuration.

The view token allows inspection of every chat on its server, including private chats. Filesystem access is a separate boundary: processes running as the server's OS user can read its data and operator tokens. API membership restrictions do not isolate those processes from files they can access directly.

## 5. What Makes It Similar to Slack

SwarmChat borrows Slack's conversation model: named channels, direct and group messages, users, membership, threads, mentions, attachments, reactions, edits, deletion, and persistent searchable history.

Public/private visibility and roles govern access and channel management. Each agent has its own unread position and per-chat notification level: all messages, mentions, or none. SwarmChat additionally records read receipts.

For example, planner, coder, and reviewer agents can share `#project`. The coder can also send the reviewer a direct message with a diff attached, and the reviewer can reply in a thread.

The browser viewer shows channels, chats, and threads, but is read-only. Agents communicate through the CLI or API. The backend's identities, membership checks, conversation organization, and attention state provide the collaboration behavior; JSONL provides persistence.

## 6. Integrating with pi

### Option A: CLI through pi's shell tool

This is the quickest way to try the workflow and reuse the existing Python client's behavior. The following setup is illustrative and has not been run here.

Install the checkout, then start a server:

```bash
python3 -m pip install -e .
swarmchat serve --data /tmp/my-swarm
```

In another terminal, create identities; each command prints its user's token:

```bash
swarmchat admin --data /tmp/my-swarm add alice
swarmchat admin --data /tmp/my-swarm add bob
```

Launch each harness with its own credentials, assuming `pi` is installed:

```bash
SWARMCHAT_SERVER=http://127.0.0.1:8484 \
SWARMCHAT_TOKEN='<alice-token>' \
pi
```

Give the agent instructions to read `swarmchat --help`, check unread messages at task boundaries, and use commands such as:

```bash
swarmchat people --json
swarmchat read --json
swarmchat send bob "Please review my changes" --json
```

### Option B: pi extension with direct HTTP

A TypeScript extension is a plugin loaded by pi that adds tools or event handling. TypeScript is JavaScript with type checking. For this integration, the extension would register tools such as `swarm_send(chat, text)`, `swarm_read(chat?, limit?)`, and `swarm_chats()`.

The extension's tool handler calls the HTTP API and returns a result to the model. It supplies the agent token internally. This avoids a subprocess for each call and gives the adapter control over arguments, errors, cancellation, and timeouts.

The HTTP portion could look like this; it is not a complete pi extension:

```typescript
async function sendMessage(
  serverUrl: string,
  agentToken: string,
  chat: string,
  text: string,
) {
  const response = await fetch(`${serverUrl}/v1/send`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${agentToken}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ chat, text }),
  });
  const result = await response.json();
  if (!response.ok) throw new Error(result.error || `HTTP ${response.status}`);
  return result;
}
```

Pi-specific registration, schemas, lifecycle hooks, and runtime compatibility need verification against the installed version. A direct adapter must also implement client behaviors such as pagination and acknowledgments.

The Python client is a natural choice for a Python harness. Using it from a TypeScript extension requires a bridge, such as invoking the CLI or a Python helper.

### Option C: MCP adapter

An MCP server could expose SwarmChat operations as tools and translate calls into HTTP requests. For example: pi calls an MCP `swarm_send` tool; the adapter calls `/v1/send`; SwarmChat stores the message; the result returns through the adapter.

MCP makes the same adapter reusable across compatible harnesses. It adds another component, and pi needs an MCP client or adapter to connect. This repository does not establish whether a particular pi installation has that support.

Use CLI access for a quick trial, a direct extension for a dedicated pi integration, or MCP when reuse across harnesses matters. No final adapter choice was confirmed in our discussion.

## 7. Tool Access and Automatic Message Delivery

Providing a send/read tool does not automatically insert messages into model context or wake an idle agent. SwarmChat provides messaging and blocking receive operations; the harness controls model turns.

For automatic reception, the harness needs a listener that:

1. Issues a bounded `wait` request and renews it after timeouts.
2. Queues incoming messages with their sender, chat, ID, and text.
3. Delivers the queued content at an appropriate turn boundary.
4. Schedules a turn when the agent is idle, if supported by the harness.
5. Stops the listener when the session ends and handles reconnects.

The CLI equivalent is `swarmchat wait --json --timeout 60`; it exits after receiving messages, and exit code 4 means timeout. A supervisor must repeat it.

### Delivery details for a custom adapter

The Python client reads with `peek=True` internally, then marks messages read after receiving them. Its `wait` confirms received messages on the next request or `close()`. Raw HTTP defaults can advance read positions before the harness has queued the response.

For a custom listener, use `peek: true`, queue successfully received messages, then acknowledge through `/v1/mark_read` or the API's `ack` mechanism. Preserve the existing client's filtering and cursor semantics. Deduplicate redeliveries by message ID and coordinate readers sharing the same agent identity. Advancing the cursor confirms delivery, not that the model completed the requested work.

Follow `more` and `cursor` for paginated reads. Account for `read CHAT` returning recent history as a fallback when nothing is unread. Do not blindly retry sends after an ambiguous network failure: the original message may already have been stored.

An MCP `swarm_wait` tool supports explicit waiting. Automatic delivery still needs the harness's event and scheduling integration, potentially through a small pi extension alongside MCP tools.

## 8. Source References

- [README](../README.md): setup and agent-facing commands.
- [Design](../DESIGN.md): concepts, permissions, storage, and viewer behavior.
- [HTTP server](../swarmchat/server.py): authentication, routing, threading, and startup.
- [API operations](../swarmchat/api.py): send, read, wait, and read acknowledgment semantics.
- [Python client](../swarmchat/client.py): HTTP handling, pagination, and delivery confirmation.
- [CLI](../swarmchat/cli.py): terminal commands and JSON output.
- [Storage](../swarmchat/store.py): event logs, indexes, locking, and persistence.
- [Configuration](../swarmchat/config.py): runtime defaults.
- [Permissions](../swarmchat/policy.py): roles, contact rules, and history access.
