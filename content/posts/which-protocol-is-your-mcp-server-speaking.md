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
carries the protocol version on each request after that. Plenty of older clients and
servers are still around, so both SDKs kept the old path alongside the new one, and a
given server might speak either, or both.

I'd just built [a server of my own](/posts/let-the-database-say-no/) on the new
revision and wanted to know how it would get on with the rest of the ecosystem. That
meant finding out what other servers actually speak, which turned out to be harder to
ask than I expected.

## You can't find out by connecting

The obvious move is to connect with the SDK and look at what you got. The trouble is
that the Python SDK's client, left on its default, does exactly the negotiation you're
trying to observe. It tries `server/discover`, falls back to `initialize` if that
fails, and hands you a working session. Whether the server spoke the new protocol or
the old one isn't recorded anywhere you can easily get at. The docstring for that
fallback logic describes it as a denylist — anything that isn't positive evidence of
a modern server falls back.

For a client that just needs a session, that's the right design. For working out what
a server is, it throws the answer away.

So the probe doesn't use it. `ClientSession` exposes `discover()` and `initialize()`
as separate calls, and the probe makes each one on its own connection. Separate
connections matter more than they look: one that has answered `discover` is locked
into the new protocol, so asking it for a handshake afterwards tells you about the
lock and nothing about the server.

Each path ends one of three ways. It worked. The server answered and said no, in which
case the error code is kept. Or nothing usable came back at all. The difference
between those last two is the whole point, because a server that never answered
hasn't told you anything about which protocol it speaks.

## Twelve servers

I probed the six official reference servers from the MCP project, five servers
published by companies whose tools people actually run, and my own as a control. All
at their latest versions, all started locally with `npx` or `uvx`, no credentials,
and the probe never calls a tool. It negotiates, lists what the server says it offers,
and checks that the listings actually work.

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

Both npm packages, both reasonably recent, opposite answers. Here's all twelve:

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

The last column is entirely predicted by the third. Every server on a version-one SDK
spoke only the old handshake, and every server on version two spoke both. None of them
implemented discovery by hand, and none failed to despite a v2 SDK. Which protocol a
server speaks, on this sample, is a dependency decision nobody on the project
necessarily made on purpose.

Every server that listed a capability could back it up. Nothing advertised tools and
then failed to list them.

The reference servers stand out. `server-everything` is the MCP project's showcase of
every protocol feature, and the version I probed was published on 31 August, five
weeks after the revision it doesn't speak.

## The upgrade that bumping won't get you

This part I got wrong first.

Before I'd probed any vendor servers, I'd checked the TypeScript side and written down
that the SDK hadn't shipped the July revision. The evidence looked solid. The newest
`@modelcontextprotocol/sdk` on npm is 1.30.0, and its build has no mention of
`2026-07-28` anywhere.

Then Context7, an npm package, answered `server/discover`. So I went and read its
dependencies. It doesn't use `@modelcontextprotocol/sdk` at all. It depends on
`@modelcontextprotocol/server` and `@modelcontextprotocol/node`, both at 2.0.0. The
TypeScript SDK's second version wasn't released as a new major of the package everyone
already had. It went out on 27 July as separate packages, and the old one simply stops
at 1.x.

That's a trap for anyone maintaining a TypeScript server. Dependabot will happily keep
bumping `@modelcontextprotocol/sdk` for you, and you'll stay on the old protocol
indefinitely, because the version that speaks the new one is a different package name.
The Python side doesn't have this problem, since `mcp` 2.x is the same package with a
higher number. It just requires someone to take the major bump.

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
error, which looks a lot more reasonable with this table in front of you.

## What running it on real servers taught me about the probe

It had bugs I'd never have found against test servers.

MotherDuck's first result was `neither`, meaning it had answered and refused both
protocols, and the probe exited successfully. It hadn't answered anything. It died on
startup because I'd started it without a flag it needs:

```text
Error: In-memory databases require the --read-write flag.
```

The SDK reports a dead server process as an error with the code `-32000` and the
message "Connection closed", and my probe treated every SDK error as the server saying
no. It had also been sending the server's stderr to `/dev/null`, which is precisely
where that one explanatory line went. A crashed server now reads as unreachable, exits
with a failure code, and shows the tail of its stderr. With the flag added, MotherDuck
speaks both.

`dbt-mcp` timed out on discovery the first time I ran it and answered normally every
time after. Discovery runs first, so on a first `uvx` run it also absorbs downloading
the package, while the handshake that follows finds a warm cache. That would have made
any server I probed for the first time look more old-fashioned than it is. Now, if
discovery gets no answer but the handshake then succeeds, the probe tries discovery
once more and says it did. A server that genuinely never answers times out twice.

And one mistake that wasn't in the probe at all. I'd concluded that the Python servers'
error code and error message contradicted each other, because the message read "Invalid
request". My own summary script had cut every message at 40 characters. The full text
was "Invalid request parameters", which matches the code exactly.

## Run it yourself

```bash
uvx --from "git+https://github.com/pavanrao/data-tools#subdirectory=tools/discover-probe" \
    discover-probe stdio -- npx -y @modelcontextprotocol/server-everything

# anything after -- is the server command, passed through untouched
discover-probe --json stdio -- uvx mcp-server-time
```

It exits `0` when every listing a server advertised worked, `1` when one didn't, and
`2` when the server never answered. A server that only speaks the old protocol exits
`0`, since that's a fact about the server rather than a lie it told.

## Loose ends

It only speaks stdio so far. Every server here offers it, but plenty of hosted servers
only offer HTTP, and I haven't probed anything running on someone else's
infrastructure. That's a decision about sending requests to other people's endpoints,
not a technical gap, and I'd rather make it deliberately.

Capabilities varied a lot more than protocols did. The AWS server and mine, both on
Python v2, advertise change notifications only under the new protocol, which is where
the mechanism for them lives. MotherDuck, also Python v2, does the reverse for its tool
list, and advertises the MCP Apps interface extension only under the new protocol.
Context7 reports identical capabilities in both. So the SDK settled which protocols a
server speaks, and what each server claims within them came down to how it was written.
Four servers isn't enough to say more.

These numbers have a short shelf life. The day the reference servers move to v2 SDKs,
the table flips. I've dated it for that reason, and would re-run it before quoting it
anywhere.

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
