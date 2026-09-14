---
title: "Which Protocol Is Your MCP Server Speaking?"
date: 2026-09-14T00:00:00Z
summary: "MCP's July revision replaced the handshake every client and server used to open a conversation. I pointed a probe at twelve servers to see which had moved, and the answer turned out to be one line in each project's dependency file."
tags: ["mcp", "python", "typescript", "sdk", "agents", "interoperability"]
categories: ["AI Engineering"]
---

<p class="lede">MCP's July revision replaced the handshake every client and server
used to open a conversation. I pointed a probe at twelve servers to see which had
moved, and the answer turned out to be one line in each project's dependency
file.</p>

Until the [2026-07-28 revision](https://modelcontextprotocol.io/specification/2026-07-28/changelog),
an MCP client opened every connection with an `initialize` handshake. The revision
removed it. A current client asks `server/discover` what the server supports, and
carries the protocol version on each request after that. Older clients and
servers are still in use, so both SDKs kept the old path alongside the new one, and a
given server might speak either, or both.

I'd just built [a server of my own](/posts/let-the-database-say-no/) on the new
revision, and wanted to know which protocol everyone else's servers speak.

## You can't find out by connecting

The obvious move is to connect with the SDK and look at what you got. The trouble is
that the Python SDK's client, left on its default, does exactly the negotiation you're
trying to observe. It tries `server/discover`, falls back to `initialize` if that
fails, and hands you a working session. Whether the server spoke the new protocol or
the old one isn't recorded anywhere you can easily get at. The docstring for that
fallback logic describes it as a denylist — anything that isn't positive evidence of
a modern server falls back.

That's a reasonable default for a client, and it's also why the probe can't use it.
`ClientSession` exposes `discover()` and `initialize()`
as separate calls, and the probe makes each one on its own connection. The
connections have to be separate, because one that has answered `discover` is locked
into the new protocol, so asking it for a handshake afterwards tells you about the
lock and nothing about the server.

Each path is recorded as having worked, been refused with an error code, or produced
nothing usable. Those last two have to stay apart, because a server that never
answered hasn't told you anything about which protocol it speaks.

## Twelve servers

I probed the six official reference servers from the MCP project, five servers
published by companies whose tools people actually run, and my own as a control. Each
was at its latest version and started locally with `npx` or `uvx` with no
credentials, and the probe never calls a tool. It negotiates, lists what the server says it offers,
and checks that the listings work.

Microsoft's Playwright server first:

```console
$ discover-probe stdio -- npx -y @playwright/mcp
target   npx -y @playwright/mcp
server   Playwright
era      legacy-only

  FAIL  discover             rejected: error -32601: Method not found
  pass  handshake            negotiated 2025-11-25
  skip  claims-modern        this path did not connect
  pass  claims-legacy        tools 24
  skip  capabilities-by-era  needs both paths to connect
  pass  deprecated-logging   does not advertise logging
```

It doesn't know `server/discover` exists, and says so correctly. It speaks the
previous revision and offers 24 tools. Upstash's Context7, run the same way:

```console
$ discover-probe stdio -- npx -y @upstash/context7-mcp
target   npx -y @upstash/context7-mcp
server   Context7
era      both

  pass  discover             supports 2026-07-28
  pass  handshake            negotiated 2025-11-25
  pass  claims-modern        tools 2, resources 0, prompts 0
  pass  claims-legacy        tools 2, resources 0, prompts 0
  pass  capabilities-by-era  identical in both eras
  pass  deprecated-logging   does not advertise logging
```

That version of Context7 was published the day I probed it. Here are all twelve:

| Server | Publisher | SDK underneath | Speaks July? |
| --- | --- | --- | --- |
| `server-everything` | MCP project | TypeScript v1 | No |
| `server-filesystem` | MCP project | TypeScript v1 | No |
| `server-memory` | MCP project | TypeScript v1 | No |
| `mcp-server-fetch` | MCP project | Python v1 | No |
| `mcp-server-time` | MCP project | Python v1 | No |
| `mcp-server-git` | MCP project | Python v1 | No |
| Playwright | Microsoft | TypeScript v1, bundled | No |
| `dbt-mcp` | dbt Labs | Python v1 | No |
| Context7 | Upstash | TypeScript v2 | Yes |
| MotherDuck | MotherDuck | Python v2 | Yes |
| AWS Documentation | AWS Labs | Python v2 | Yes |
| `sqlite-mcp` | me | Python v2 | Yes |

The SDK column predicts the last one on every row. Every server on a version-one SDK
spoke only the old handshake, and every server on version two spoke both. I didn't
find one that had implemented discovery by hand, or one on a v2 SDK that had turned it
off. No server advertised a capability it couldn't list. And `server-everything`, which
the MCP project uses to demonstrate every protocol feature, was published on 31 August,
five weeks after the revision it doesn't speak.

## The upgrade that bumping won't get you

Before probing any vendor servers, I'd checked the TypeScript side and written down
that its SDK hadn't shipped the July revision, because the newest
`@modelcontextprotocol/sdk` on npm is 1.30.0 and there's no mention of `2026-07-28`
anywhere in its build.

Then Context7, an npm package, answered `server/discover`, so I went and read its
dependencies. It depends on `@modelcontextprotocol/server` and
`@modelcontextprotocol/node`, both at 2.0.0, and doesn't use
`@modelcontextprotocol/sdk` at all. TypeScript v2 went out on 27 July under those new
package names, and `@modelcontextprotocol/sdk` itself stops at 1.x.

If you maintain a TypeScript server, Dependabot will keep bumping
`@modelcontextprotocol/sdk` for you and you'll stay on the old protocol, because the
version that speaks the new one has a different name. On the Python side it's the same
package, `mcp`, so moving is an ordinary major-version upgrade that someone has to
choose to take.

## Two ways to say you don't know a method

The servers that don't speak July turn it down in two different ways, and the split
follows the SDK again. The TypeScript ones answer `-32601 Method not found`. The
Python ones don't:

```console
$ discover-probe stdio -- uvx dbt-mcp
target   uvx dbt-mcp
server   dbt
era      legacy-only

  FAIL  discover             rejected: error -32602: Invalid request parameters
  pass  handshake            negotiated 2025-11-25
  skip  claims-modern        this path did not connect
  pass  claims-legacy        tools 39, resources 1, prompts 0
  skip  capabilities-by-era  needs both paths to connect
  pass  deprecated-logging   does not advertise logging
```

`-32602` is the JSON-RPC code for calling a method that exists with the wrong
arguments. For a method the server has never heard of, the spec says `-32601`. The
Python v1 SDK models incoming requests as a closed set of known types, which fits the
unknown method failing validation before anything routes it — though I haven't traced
that path through the v1 source, so I'd call it the likely mechanism rather than a
proven one.

It matters if you write a client that negotiates for itself. Fall back to the old
handshake only when you see `-32601`, which is the obvious reading of the spec, and
every Python v1 server in this table — dbt Labs' included — looks like a server that
won't talk to you at all. The Python v2 SDK's own negotiation falls back on nearly any
error, which handles both kinds of refusal in the table.

## What running it on real servers taught me about the probe

MotherDuck's first result was `neither`, meaning it had answered and refused both
protocols, and the probe exited successfully. In fact it had died on startup before
answering anything, because I'd left off a flag it needs:

```text
Error: In-memory databases require the --read-write flag.
```

The SDK reports a dead server process as an error with the code `-32000` and the
message "Connection closed", and my probe treated every SDK error as the server saying
no. It had also been sending the server's stderr to `/dev/null`, which is
where that line went. A crashed server now reads as unreachable, exits
with a failure code, and shows the tail of its stderr. With the flag added, MotherDuck
speaks both.

`dbt-mcp` timed out on discovery the first time I ran it and answered normally every
time after. Discovery runs first, so on a first `uvx` run it also absorbs downloading
the package, while the handshake that follows finds a warm cache. That would have made
any server I probed for the first time look more old-fashioned than it is. Now, if
discovery gets no answer but the handshake then succeeds, the probe tries discovery
once more and says it did. A server that never answers at all times out twice.

I also made a mistake outside the probe. I'd concluded that the Python servers' error
code and message contradicted each other, because the message read "Invalid request".
My own summary script had cut every message at 40 characters, and the full text was
"Invalid request parameters", which matches the code.

## Run it yourself

```bash
uvx --from "git+https://github.com/pavanrao/data-tools#subdirectory=tools/discover-probe" \
    discover-probe stdio -- npx -y @modelcontextprotocol/server-everything

# anything after -- is the server command, passed through untouched
discover-probe --json stdio -- uvx mcp-server-time
```

It exits `0` when every listing a server advertised worked, `1` when one didn't, and
`2` when the server never answered. A server that only speaks the old protocol exits
`0`, because the failure codes are kept for servers that advertise something they
can't do.

## Loose ends

It only speaks stdio so far, which every server here offers, so anything available
only over HTTP is out of reach. I also haven't probed anything running on someone
else's infrastructure, because that means sending requests to their endpoints and I
haven't decided to do that yet.

Capabilities varied a lot more than protocols did. The AWS server and mine, both on
Python v2, advertise change notifications only under the new protocol, which is where
the mechanism for them lives. MotherDuck, also Python v2, does the reverse for its tool
list, and advertises the MCP Apps interface extension only under the new protocol.
Context7 reports identical capabilities in both. Four servers isn't enough to tell
whether those differences come from the SDKs or from how each server was written.

The table will change as soon as the reference servers move to v2 SDKs. I've dated it
for that reason, and would re-run it before quoting it anywhere.

<div class="colophon">
  <p><strong>The code.</strong> <code>discover-probe</code> lives in
  <a href="https://github.com/pavanrao/data-tools">pavanrao/data-tools</a> under
  <code>tools/discover-probe/</code>. Its design record,
  <code>docs/011_discover-probe.md</code>, has the full table with versions and the
  corrections to what I first wrote down, and the raw measurement is in
  <code>evidence/discover-probe.jsonl</code>.</p>
  <p><strong>How this was made.</strong> <code>discover-probe</code> and this write-up
  were both built with Claude Code. Every piece of terminal output here was copied from
  a run on 14 September 2026, and all twelve servers were probed on the same build of
  the tool.</p>
  <p>Written against MCP protocol revision 2026-07-28.</p>
</div>
