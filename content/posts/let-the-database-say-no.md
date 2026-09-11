---
title: "Let the Database Say No"
date: 2026-09-09T00:00:00Z
summary: "Most SQL-over-MCP servers decide whether a query is safe by reading the query. That's a denylist, and denylists have to guess what's coming. SQLite has had a better answer sitting in it for twenty years."
tags: ["mcp", "sqlite", "sql", "security", "agents", "python"]
categories: ["Data Engineering"]
---

<p class="lede">Most SQL-over-MCP servers decide whether a query is safe by reading
the query. That's a denylist, and denylists have to guess what's coming. SQLite has
had a better answer sitting in it for twenty years.</p>

The plan was a small [MCP](https://modelcontextprotocol.io/) server: read-only SQL
over a SQLite file, three tools, a row cap. I assumed the time would go on the
protocol. It didn't. Nearly all of it went on one question that turned out to be
harder than it looks — when an agent hands you a SQL statement, how do you decide
whether to run it?

## First attempt: read the SQL

Look at the statement. If it starts with `SELECT`, run it. If it contains `DROP` or
`DELETE` or `UPDATE`, refuse. Most published examples do a version of this, and it's
what I wrote first.

It's a denylist, so it has to anticipate. Here's what mine already needed to get
right, before anything unusual turned up:

<div class="verdicts">
  <div class="verdict">
    <span class="tag-verdict no">misses</span>
    <p><code>WITH x AS (DELETE FROM t RETURNING *) SELECT * FROM x</code> — starts with <code>WITH</code>, and in several engines that's a write.</p>
  </div>
  <div class="verdict">
    <span class="tag-verdict no">misses</span>
    <p><code>PRAGMA journal_mode = WAL</code> — no forbidden keyword anywhere, and it changes the database.</p>
  </div>
  <div class="verdict">
    <span class="tag-verdict no">misses</span>
    <p><code>SELECT 1; DROP TABLE orders</code> — starts with <code>SELECT</code>, if you only check the front.</p>
  </div>
  <div class="verdict">
    <span class="tag-verdict no">refuses wrongly</span>
    <p><code>SELECT * FROM shipments WHERE note LIKE '%DELETE%'</code> — a keyword scan says no to an ordinary query.</p>
  </div>
</div>

Every one of these has a fix, and that's sort of the problem. They're patches against
shapes somebody happened to think of, and the list of shapes belongs to whoever
maintains the SQL dialect rather than to me. The part that bothered me more is that
the failures are quiet. If a new syntax slips past the check next year, nothing tells
you. The server keeps returning results and you keep believing it's read-only.

## Letting SQLite decide

SQLite will answer the question for you, using two things that have been in there for
years and don't come up much.

First, open the file read-only. Not a flag your code consults later — the handle
can't write, in the same way a file opened for reading can't be written to.

```python
con = sqlite3.connect(f"file:{path}?mode=ro", uri=True)
```

Second, install an **authorizer**. SQLite calls it before every action it's about to
take and tells you what the action is, which table, and which column. You return OK
or DENY.

```python
_READ_ACTIONS = frozenset({
    sqlite3.SQLITE_SELECT,
    sqlite3.SQLITE_READ,
    sqlite3.SQLITE_FUNCTION,
})

def authorize(action, arg1, *rest):
    if action not in _READ_ACTIONS:
        return sqlite3.SQLITE_DENY
    return sqlite3.SQLITE_OK

con.set_authorizer(authorize)
```

That's the guard. It doesn't care how the statement is spelled, or whether it uses a
CTE, or what SQL grows next year, because the engine that's about to run the thing is
the same one deciding whether it may. Those four cases from earlier:

<div class="verdicts">
  <div class="verdict">
    <span class="tag-verdict yes">denies</span>
    <p>The writing CTE — though on SQLite the parser gets there first, because its
    CTEs are SELECT-only. On an engine where that statement is legal, the authorizer
    is what stops it.</p>
  </div>
  <div class="verdict">
    <span class="tag-verdict yes">denies</span>
    <p>The pragma. <code>SQLITE_PRAGMA</code> isn't in the allowed set.</p>
  </div>
  <div class="verdict">
    <span class="tag-verdict yes">denies</span>
    <p>The chained <code>DROP</code>, twice over — one call carries one statement anyway.</p>
  </div>
  <div class="verdict">
    <span class="tag-verdict yes">allows</span>
    <p>The <code>LIKE '%DELETE%'</code> query. It's a read, and the text inside it was never the question.</p>
  </div>
</div>

All four, and I didn't reason about any of them. That's the trade an allowlist buys
you: it can be wrong, but only by refusing something it should have allowed, which
you find out about immediately because someone complains.

<div class="callout">
  <span class="label">The general form</span>
  <p>Put the boundary where the system already enforces one. On a warehouse that's a
  read-only role granted on a narrow set of views, not a string check in your server.
  Same idea, different engine.</p>
</div>

## The guard's first victim was me

I wrote the authorizer, then wrote `schema()`, which reads column information with
`PRAGMA table_info`. It got refused. My own tool couldn't read its own schema, and
the message SQLite gives you is three words long:

```text
sqlite3.DatabaseError: not authorized
```

No mention of which action, which table, or which pragma. I spent a while assuming
I'd broken the connection string.

What's actually going on is that `PRAGMA` is one authorizer action covering
everything from `table_info`, which reads metadata, to `journal_mode`, which changes
the database. It can't be allowed wholesale, so the fix names the one pragma the tool
needs:

```python
_READ_PRAGMAS = frozenset({"table_info"})
```

I'd have preferred to discover this some other way, but it's the strongest argument
for the approach that I have. A denylist would have let every pragma through,
including the ones that write, and nothing would have failed. I'd have shipped it and
never known. Instead the allowlist broke my own code in development, which is the
cheapest place for it to break.

## How do you say no?

Having refused something, you have to report it. The obvious move is to raise an
error — the protocol has a perfectly good error channel and it's right there.

I don't think it's the right channel, though. An error tells a model that something
broke, and the sensible response to something breaking is to try again. But nothing
broke. The server did exactly what it was built to do. The model doesn't need to know
that the call failed; it needs to know what to do differently.

So a refusal comes back as an ordinary, successful result that happens to say no:

```json
{
  "refused": true,
  "guard": "not-read-only",
  "reason": "this database is served read-only; the statement asked to change it"
}
```

`guard` is a stable token, so code can branch on it and a model can recognise it
without parsing English. `reason` is for whoever reads the transcript afterwards.
Since the result isn't an error, nothing downstream treats it as a blip worth
retrying.

| Guard | Fires when |
| --- | --- |
| `not-read-only` | The statement asked to change something. |
| `multiple-statements` | A second statement tried to ride along after a semicolon. |
| `table-not-allowed` | The table is outside what this server exposes. |
| `unreadable-database` | The file couldn't be opened read-only at all. |

## If you truncate, say so

Every server like this caps how many rows it returns, or one careless query eats the
whole context window. The cap isn't the interesting bit. What matters is that a
truncated result which doesn't mention it was truncated is worse than no result at
all, because the first page of an answer looks exactly like the entire answer.

```json
{"rows": [["ada"], ["bob"]], "truncated": true, "row_limit": 2, "elapsed_ms": 0.18}
```

`elapsed_ms` is standing in for the field that'll matter on a real warehouse, where
it would be bytes scanned and money. A model can't trade accuracy against cost if it
has no idea what anything costs. On SQLite that's a rounding error and the field is
nearly pointless. On Snowflake it's the whole bill — the compute dwarfs the token
spend by more than people expect, and the agent's loop is what drives it.

## Run it against something real

[Chinook](https://github.com/lerocha/chinook-database) is a sample database most
people have run into: a fictional music store, with a catalogue of artists, albums
and tracks sitting next to a `Customer` table full of names, addresses, phone
numbers and email addresses. That split is useful here, because it is the same split
you have in production — the thing an agent should see, beside the thing it should
not.

```bash
curl -L -o chinook.db \
  https://github.com/lerocha/chinook-database/releases/download/v1.4.5/Chinook_Sqlite.sqlite
```

The server is three tools. This is one of them, in full:

```python
@server.tool()
def query(sql: str) -> dict[str, object]:
    """Run one read-only SQL statement. Returns rows, plus whether the row cap
    truncated them."""
    return query_impl(db, sql)
```

Start it scoped to the catalogue, and the customers simply are not there:

```console
$ sqlite-mcp --table Album --table Artist --table Track schema chinook.db
Album
    AlbumId                  INTEGER NOT NULL PK
    Title                    NVARCHAR(160) NOT NULL
    ArtistId                 INTEGER NOT NULL
Artist
    ArtistId                 INTEGER NOT NULL PK
    Name                     NVARCHAR(120)
Track
    TrackId                  INTEGER NOT NULL PK
    Name                     NVARCHAR(200) NOT NULL
    ...
```

A normal query, capped, saying so:

```console
$ sqlite-mcp --max-rows 3 query chinook.db "SELECT Name FROM Artist ORDER BY ArtistId"
Name
---------
AC/DC
Accept
Aerosmith

-- truncated at 3 rows; this answer is partial.
```

And four things it will not do. Each names the guard that stopped it, and exits
non-zero, so a script can branch without parsing the sentence:

```console
$ sqlite-mcp --table Album --table Artist --table Track \
    query chinook.db "SELECT Email, Phone FROM Customer"
refused [table-not-allowed]: that table is outside the tables this server exposes

$ sqlite-mcp query chinook.db "DELETE FROM Customer WHERE CustomerId = 1"
refused [not-read-only]: this database is served read-only; the statement asked to change it

$ sqlite-mcp query chinook.db "SELECT 1; DROP TABLE Album"
refused [multiple-statements]: only one statement may be sent at a time; a second statement cannot ride along

$ sqlite-mcp query chinook.db "PRAGMA journal_mode = WAL"
refused [not-read-only]: this database is served read-only; the statement asked to change it
```

That last one is the case a keyword scan waves through. There is no `DROP`, no
`DELETE`, no `UPDATE` anywhere in it, and it changes how the database writes to disk.

Afterwards the file is byte-for-byte what it was. 59 customers, 347 albums, same
checksum.

## Was it worth building?

For the case you're probably picturing, no. You, at a terminal, with credentials you
already have, should just write the SQL. A server there is overhead. I'd reached the
same conclusion about a different tool in this project a few weeks ago and it applies
here too.

It changes when the situation does:

- **The consumer isn't you.** A colleague who doesn't write SQL, an agent running
  unattended, a client with no shell. The guards are what make handing it over
  acceptable.
- **The work is a loop.** "Why did last night's load drop by a third" is fifteen
  queries, each depending on what the last one returned. Pasting SQL back and forth
  fifteen times is miserable.
- **The access needs governing.** Someone else is asking, and there has to be a limit
  and a record of it.

None of those is really about MCP. They're about who's asking and how often. What the
protocol contributes is narrower: write the server once and every client can use it,
which is genuinely the problem it exists to solve, and why the spec keeps comparing
itself to the language server protocol.

## Loose ends

Two things I'd tell someone doing this next. Build a server rather than a client,
because the server side is where the design decisions live. And point it at something
the model can't already reach — a database qualifies, a folder of text files doesn't,
and if you build the second one you'll spend a week discovering it was redundant.

Then test the round trip rather than the registration, which caught a real bug for me.
In the Python SDK a tool annotated `-> dict` registers with no output schema and
returns nothing structured. Annotated `-> dict[str, object]`, it works. Nothing
errors, nothing warns, and the tool looks identical when you list it. The only test
that catches it is one that goes through a client and reads what comes back.

The thing I'm still unsure about is the shape. Three tools felt obvious, but MCP has
resources and prompts too, and a schema listing is arguably a resource I turned into a
tool without thinking about it. Resources get included when the client decides; tools
get called when the model decides. Which is right here probably depends on the client,
and I don't have a number either way. There's a lab in the backlog to measure it. I
haven't run it.

<div class="colophon">
  <p><strong>The code.</strong> <code>sqlite-mcp</code> lives in
  <a href="https://github.com/pavanrao/data-tools">pavanrao/data-tools</a> under
  <code>tools/sqlite-mcp/</code>. Its design record is
  <code>docs/009_sqlite-mcp.md</code>, and the protocol concepts it exercises —
  along with the ones it does not — are in <code>docs/010_mcp-concepts.md</code>.</p>
  <p><strong>How this was made.</strong> <code>sqlite-mcp</code> and this write-up were
  both built with Claude Code, across two afternoons. The commit history shows the
  shape of that if you are curious. Every error message quoted here was copied out
  of a terminal rather than recalled.</p>
  <p>Written against MCP protocol revision 2026-07-28, which removed the initialize
  handshake and protocol-level sessions. If a tutorial you're reading mentions either,
  it predates this.</p>
</div>
