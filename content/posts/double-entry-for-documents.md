---
title: "Double-Entry for Documents"
date: 2026-09-15T00:00:00Z
summary: "When an AI system reads a folder of files, some of them don't make it. The pipeline reports no error and the answers keep coming. ingest-ledger counts every file twice, by two independent routes, and refuses to answer when the two counts disagree."
tags: ["rag", "document-processing", "python", "evaluation", "data-engineering"]
categories: ["AI Engineering"]
---

When an AI system reads a folder of files, some of them don't make it. The pipeline
reports no error, and the answers keep coming. This is a way to find out what was
missed.

Take one spreadsheet, `pricing.xlsx`. Its own structure declares four tabs. The
extractor returned three. Those two numbers come from different code, which is why the
gap is visible at all: one tab came back empty and said nothing about it.

## Where documents go missing

You have a folder of business documents — contracts, spreadsheets, scanned forms, a zip
file somebody emailed you — and you want to ask questions about them in plain English.
The standard way to build that is RAG, retrieval-augmented generation, and it works in
four steps:

1. **Extract.** Open each file and pull the text out of it. A PDF becomes a string of
   words; a spreadsheet becomes rows.
1. **Chunk.** Cut that text into passages a few hundred words long, because whole
   documents are too big to work with at once.
1. **Index.** Convert each passage into a list of numbers — an embedding — that captures
   what it is about, so that passages about similar topics end up with similar
   numbers.
1. **Retrieve and answer.** When you ask a question, turn the question into numbers the
   same way, find the passages whose numbers are closest, and hand those passages to a
   language model to write an answer from.

The design works: the model answers from your documents instead of from memory, and it
can cite which passage it used. It's the standard shape for "chat with your documents"
products, and the trouble starts in step one, where it stays invisible all the way to
step four.

### Five ways extraction fails without telling you

- **The scanned page.** A PDF made by a photocopier holds images of the page with no
  text layer. Ask a library for its words and the call returns an empty string rather
  than an error. To the next stage of the pipeline, a three-page document that yielded
  nothing looks like a three-page document that was blank to begin with.
- **The other tabs.** The common snippet for reading a spreadsheet opens the *active*
  sheet, whichever tab happened to be selected when the file was last saved. A workbook
  with pricing on tab four gives up its cover page and nothing else.
- **The footnotes.** A Word file is a zip archive of parts. The main body is one part;
  footnotes, headers, and the contents of text boxes are others. `python-docx` walks the
  body paragraphs, and a footnote can carry the clause that changes the meaning.
- **The file that was too big.** Parsing a large XML document can consume more memory
  than the machine will give it. The operating system kills the process. If the
  surrounding code catches that and carries on with whatever it had, you get a fragment
  presented as a whole.
- **The zip inside the zip.** Archives get unpacked one level. Whatever was nested
  deeper never reaches the corpus.

### Why nobody catches it

None of these raise an exception. Extraction "succeeds" and returns something. Every
downstream stage assumes the stage above it did its job, and nothing in the pipeline is
responsible for asking whether the input arrived whole.

The model reports what it was given, and what it wasn't given never appears, so the
output is faithful to the input it received. The obvious check is to ask the AI what it processed —
*"summarise which files you read"* — but a pipeline that silently dropped a file can
just as silently fail to mention it, which is why the count has to come from outside
the pipeline.

## Counting twice, by two different routes

Accountants solved a version of this problem in the fifteenth century. Double-entry
bookkeeping records every transaction twice, by two routes that must agree. What it buys is
in the comparison: a single entry has nothing to be checked against, while an error that
reaches only one of two columns shows up as a mismatch.

`ingest-ledger` applies that to documents. Every file is counted twice:

<style>
.dg { max-width: 100%; height: auto; margin: 1.5rem 0; }
.dg text { font: 500 12px ui-sans-serif, system-ui, sans-serif; fill: currentColor; }
.dg .dg-code { font-family: ui-monospace, Menlo, monospace; }
.dg .dg-m { font-size: 11px; opacity: .7; }
.dg .dg-num { font: 600 20px ui-sans-serif, system-ui, sans-serif; }
.dg .dg-verdict { font: 600 13px ui-monospace, Menlo, monospace; letter-spacing: .04em; }
.dg rect { fill: none; stroke: currentColor; stroke-opacity: .45; }
.dg .dg-box-w { stroke-opacity: .9; stroke-width: 1.5; }
.dg path.dg-line { fill: none; stroke: currentColor; stroke-opacity: .5; }
.dg .dg-arrow { fill: currentColor; fill-opacity: .5; stroke: none; }
</style>

<svg class="dg" viewBox="0 0 720 260" role="img" aria-label="A file is measured by two independent paths. The upper path counts declared units from the file's structure without reading content. The lower path extracts content and counts what came back. The two counts are compared, producing a coverage figure and a status of partial at seventy-five percent.">
  <defs>
    <marker id="ar" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto">
      <path d="M0,0 L10,5 L0,10 z" class="dg-arrow"></path>
    </marker>
  </defs>
  <rect x="8" y="102" width="118" height="56" rx="3"></rect>
  <text x="67" y="126" text-anchor="middle" class="dg-code">pricing.xlsx</text>
  <text x="67" y="145" text-anchor="middle" class="dg-m">one input file</text>
  <path d="M126 130 L168 130 L168 58 L212 58" class="dg-line" marker-end="url(#ar)"></path>
  <path d="M126 130 L168 130 L168 202 L212 202" class="dg-line" marker-end="url(#ar)"></path>
  <rect x="218" y="30" width="200" height="58" rx="3"></rect>
  <text x="318" y="53" text-anchor="middle">Read the structure</text>
  <text x="318" y="72" text-anchor="middle" class="dg-m">tab names only, never content</text>
  <rect x="218" y="174" width="200" height="58" rx="3"></rect>
  <text x="318" y="197" text-anchor="middle">Extract the content</text>
  <text x="318" y="216" text-anchor="middle" class="dg-m">in a supervised subprocess</text>
  <path d="M418 58 L446 58" class="dg-line" marker-end="url(#ar)"></path>
  <path d="M418 202 L446 202" class="dg-line" marker-end="url(#ar)"></path>
  <text x="462" y="53" class="dg-num">4</text>
  <text x="480" y="53" class="dg-m">declared</text>
  <text x="462" y="197" class="dg-num">3</text>
  <text x="480" y="197" class="dg-m">read</text>
  <path d="M462 68 L462 112 L534 112" class="dg-line"></path>
  <path d="M462 212 L462 148 L534 148" class="dg-line"></path>
  <path d="M534 112 L534 130 L556 130" class="dg-line" marker-end="url(#ar)"></path>
  <rect x="562" y="96" width="150" height="68" rx="3" class="dg-box-w"></rect>
  <text x="637" y="121" text-anchor="middle" class="dg-verdict">PARTIAL 75%</text>
  <text x="637" y="140" text-anchor="middle">quarantined,</text>
  <text x="637" y="156" text-anchor="middle">not indexed</text>
</svg>

The upper path never reads content. It opens the file and looks only at its skeleton:
how many pages the page tree declares, what the tabs are named, how many entries the
archive lists. This is cheap, and it is structurally different work from extraction.
When the lower path dies of a memory limit, the upper path's answer is still standing.

If both numbers came from the same code, their agreement would prove nothing. The rest
of the tool is built to keep the two routes apart: the structural count never opens the
content, and the extraction count never reads the structure.

### What gets counted

A "unit" is whatever subdivision of a file can be counted from structure alone. It
differs per format, deliberately:

| Format | Unit | Counted from |
| --- | --- | --- |
| `.pdf` | page | the document catalog's page tree |
| `.xlsx` | sheet | the workbook's tab list, hidden tabs included |
| `.docx` | block | paragraphs across body, footnotes, headers and text boxes |
| `.xml` | node | top-level children, counted by streaming |
| `.csv` `.txt` `.md` | row / line | non-blank lines, counted on raw bytes |
| `.zip` | member | the archive index, recursively |

## Five stages, in order

Each stage consumes what the last produced:

1. **Manifest.** Walk the input, descending into archives up to four levels deep. For
   each file: a SHA-256 hash, the true format sniffed from its leading bytes, and a
   **declared unit count** read from structure. Nothing is extracted here, which makes
   this stage safe to run on anything.
1. **Extract, under supervision.** Each file is extracted in **its own subprocess**,
   with a memory cap and a timeout. The subprocess is there so that a parse
   killed for using too much memory produces an exit code the parent can see, rather
   than a half-filled variable inside a `try` block.
1. **Reconcile.** Join the two counts. Assign each file complete, partial, failed,
   unsupported or empty, with a coverage percentage. Anything below the threshold is
   **quarantined**, and never passed to the next stage.
1. **Index, including the gaps.** Only reconciled content gets chunked and indexed. A
   second index is built alongside it, of everything still known about the files that
   *failed*: filename, the archive it came from, its tab names, the error. The last
   stage reads from this second index.
1. **Ask, and sometimes refuse.** Every question is scored twice: against what was read,
   and against the descriptions of what wasn't. When the best match is something the
   pipeline failed to read, it **refuses** and says which file it could not read.

### What the second index is for

Ordinary retrieval has a trap here. A file that failed to extract contributes *no
passages*, so it's invisible to similarity search: there's nothing to rank, and the
system never knew it existed. It answers from whatever else it has, at full confidence.

A file you couldn't read still leaves a residue. You have its
name, the name of the tab that came back empty, and the error. That residue is often the
same vocabulary a question uses: somebody asking about payment milestones and a
spreadsheet tab called `Payment Schedule` share their most important words. So the gaps
get embedded too and ranked against the question alongside the real passages, and the
tool refuses when a gap wins.

## What it prints

All output below is real, produced by `make demo` against a corpus of deliberately
broken files the repository generates for the purpose:

```text
$ ingest-ledger report /tmp/demo

file                                declared     read    cov  status
---------------------------------------------------------------------
bom_mixed.csv                         4 rows        4  100%  COMPLETE
liar.pdf                                   -        -    0%  UNSUPPORTED
                                -> extension claims pdf, content is html - content not indexed
multi_sheet.xlsx                    4 sheets        3   75%  PARTIAL
                                -> missing sheets: Notes
nested.zip!inner.zip!terms.txt      30 lines       30  100%  COMPLETE
oversized.xml                   240000 nodes   240000  100%  COMPLETE
scanned.pdf                          3 pages        0    0%  PARTIAL
                                -> missing pages: 1, 2, 3
textbox_footnote.docx              11 blocks       11  100%  COMPLETE

files: 6 of 9 complete, 3 quarantined
  pages       25.0%   <-- gap
  sheets      75.0%   <-- gap
```

Coverage is reported per unit kind and never summed across kinds. An early version
averaged everything into one figure, and 240,000 XML nodes drowned out three lost PDF
pages: the headline read 100% directly above a list of quarantined files, because a page
and a node were being added together as though they were the same quantity.

```text
$ ingest-ledger compare /tmp/demo

input                  naive extractor           ingest-ledger
-----------------------------------------------------------------------------
bom_mixed.csv          ok, 53 chars              COMPLETE (100%)
liar.pdf               ok, 26 chars              UNSUPPORTED (0%)            <--
multi_sheet.xlsx       ok, 34 chars              PARTIAL (75%)               <--
nested.zip             ok, 21 chars              2 members, 311 chars        <--
oversized.xml          ok, 29378855 chars        COMPLETE (100%)
scanned.pdf            ok, 0 chars               PARTIAL (0%)                <--
textbox_footnote.docx  ok, 135 chars             COMPLETE (100%)             <--
whitebox.pdf           ok, 76 chars              COMPLETE (100%)

5 of 8 inputs would enter the index silently damaged under the naive extractor.
```

The left column is a deliberately ordinary extractor, and every shortcut in it is one
you can find in shipped code. It says `ok` on all eight, including `ok, 0 chars` for a
three-page scanned document, raising no exception at any point.

```text
$ ingest-ledger ask "who owns the intellectual property?"

[ANSWERED] Answered from 2 indexed passages; 1 unread file(s) in this
corpus do not bear on it.

  0.346  general_terms.md@0
         Intellectual property created under this agreement vests in the State...

$ ingest-ledger ask "what are the payment schedule milestones?"

[ABSTAINED] Refused: the most relevant material for this question was not
successfully read. pricing.xlsx (missing sheets: Payment Schedule).
Re-run ingestion for these files before trusting an answer.

  GAP 0.258  pricing.xlsx - missing sheets: Payment Schedule
```

The same corpus and index, with two questions. The first can be answered from files that
were read in full; the second can't, and the tool says so instead of answering from
the remainder. It exits with code `3` on a refusal, so a script can branch on it.

### Two things found by building it

**The test corpus caught a bug in the tool itself, on its first run.** One fixture is an
HTML page saved with a `.pdf` extension. The PDF library opened it without complaint,
returned plausible-looking text, and the file reconciled as complete. A forgiving parser
handed wrong content to a checker that had no way to know it was wrong. That is now a
separate stage: every file's leading bytes are checked against its extension, and a
mismatch is never handed to a parser at all.

**pdfmux measures fidelity, not coverage.** PDF page-level checking is
delegated to [pdfmux](https://github.com/NameetP/pdfmux), which is good at finding pages
where text existed in the source but the extractor returned nothing. Run against the
three-page scanned fixture, it returns `PASS` at `coverage 1.00`, and that is correct.
pdfmux measures *fidelity*: did the extractor drop text that was there? A scanned page
has no text layer, so nothing was dropped. Measured against the declared page count, the
same file yielded nothing at all, which is the hole an index would inherit. Both
questions are worth asking, so pdfmux's verdict is recorded as *evidence* in the ledger
rather than as the status, and a page counts as missing if either signal says so.

## What this does not do

These are the limits I know about:

- **Gap matching is lexical.** The default embedding is a dependency-free hashing
  function, chosen so the refusal logic can be tested offline on any machine with no
  model and no API key. It matches on shared vocabulary, so a question phrased only
  in synonyms of a gap's description will not trigger a refusal. Swapping in a trained
  encoder is a one-line change behind the same interface, and whether it improves
  refusals is untested.
- **The thresholds are guesses.** The constants deciding "related enough to disclose"
  and "relevant enough to refuse" were tuned against the demo corpus, not derived from
  evaluation data. They are policy, and they deserve a proper eval set before anyone
  trusts them on real documents.
- **Counting units says nothing about their quality.** A page that extracts to one
  garbled line still counts as a page recovered. This catches wholesale loss; scrambled
  reading order, mangled tables and OCR errors are real problems it doesn't measure.
- **There is no answer generation.** `ask` returns ranked passages and a verdict. Wiring
  a language model to write prose from them is deliberately left to the caller.
- **The format list is short.** PowerPoint, email archives, images and HTML aren't
  covered. An uncovered format is reported as unsupported, which puts it in
  the ledger as a known gap without doing anything to close it.

## Where this goes next

- **An evaluation set for the thresholds.** Questions paired with known-correct
  refuse-or-answer decisions, so the constants above stop being guesses and start being
  measured. This is the next thing I'd build.
- **A pluggable real encoder.** The interface exists; a local sentence-transformer
  behind it would test how much of the refusal behaviour survives paraphrase.
- **Sub-unit reconciliation.** Today a spreadsheet is counted in tabs. Counting used
  rows within each tab would catch a sheet that opened but returned only its header.
- **Quality signals alongside completeness.** Reading order, table structure and OCR
  confidence are a different axis from "did it arrive", and existing tools already
  measure some of them well.
- **Exposure over MCP.** The repository's wider goal is a set of composable tools;
  publishing reconcile-and-ask as a Model Context Protocol server would let any AI
  assistant check document coverage before answering from a corpus.
- **Incremental runs.** Every file is already content-hashed, so re-reconciling only
  what changed, and distinguishing "this file changed" from "this file finally
  extracted", is mostly bookkeeping.

### Why count twice

A system that answers questions from your documents makes an implicit promise: that it
read them. Nothing in the four steps above checks that promise, and the failures are
the quiet kind that produce plausible output. Counting twice by two routes is an old,
unglamorous answer, and it works here for the same reason it worked for bookkeepers: one
number on its own has nothing to disagree with.

<div class="colophon">
  <p><strong>The code.</strong> <code>ingest-ledger</code> lives in
  <a href="https://github.com/pavanrao/data-tools">pavanrao/data-tools</a> under
  <code>tools/ingest-ledger/</code>, with its design record in <code>docs/</code>.</p>
  <p><strong>How this was made.</strong> <code>ingest-ledger</code> and this write-up
  were both built with Claude Code. Every figure here is output from a run against the
  demo corpus the repository builds.</p>
</div>
