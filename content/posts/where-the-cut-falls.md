---
title: "Where the Cut Falls"
date: 2026-09-14T09:00:00Z
summary: "A search system stores documents in pieces, and something has to decide where one piece ends. That decision is made once, before any question exists, and it sets a ceiling no retriever can lift. chunking-lab measures the ceiling: fourteen strategies, 652 questions with marked answers, and three beliefs that did not survive the measurement."
tags: ["rag", "retrieval", "chunking", "evaluation", "python"]
categories: ["AI Engineering"]
---

<style>
/* Diagrams and the interactive chunking figure. Structural colours come from
   PaperMod's variables; the gold answer-span pair and --bad are in custom.css,
   because they carry meaning rather than decoration. */
.wcf { margin: 1.8rem 0; display: flex; flex-direction: column; gap: .8rem; }
.dia { max-width: 100%; height: auto; color: var(--content); display: block; }
.dia text { font-size: 12px; fill: currentColor; }
.dia .mono { font-family: ui-monospace, Menlo, monospace; font-size: 11px; }
.dia .lbl { font-size: 11px; fill: var(--secondary); }
.dia .cap { font-size: 11px; fill: var(--bad); }
.dia .box { fill: var(--entry); stroke: currentColor; stroke-width: 1.2; }
.dia .box-hi { fill: var(--bad-wash); stroke: var(--bad); stroke-width: 2; }
.dia .strip { fill: var(--code-bg); stroke: var(--tertiary); stroke-width: 1; }
.dia .picked { fill: var(--code-bg); stroke: currentColor; stroke-width: 2; }
.dia .holds { fill: var(--bad-wash); stroke: var(--bad); stroke-width: 2; }
.dia .goldbar { fill: var(--mark); stroke: var(--mark-ink); stroke-width: .75; }
.dia .ln { stroke: currentColor; stroke-width: 1.2; fill: none; }
.dia .ln-cut { stroke: var(--bad); stroke-width: 2; fill: none; }
.dia .ln-soft { stroke: var(--tertiary); stroke-width: 1; fill: none; }
.dia .dash { stroke: var(--bad); stroke-width: 1.4; fill: none; stroke-dasharray: 5 4; }
/* .md-content figure>figcaption sets bold and --primary on the whole caption,
   so these need the same weight of selector to win. */
.md-content figure.wcf > figcaption { font-size: .92rem; line-height: 1.55;
    font-weight: normal; color: var(--secondary); margin: 0; }
.md-content figure.wcf > figcaption b { font-weight: 600; color: var(--content); }

.lab { border: 1px solid var(--tertiary); border-radius: var(--radius);
       overflow: hidden; margin: 1.8rem 0; }
.lab-head { padding: .9rem 1.2rem; border-bottom: 1px solid var(--border);
            background: var(--code-bg); display: flex; flex-wrap: wrap;
            gap: .4rem 1.5rem; align-items: baseline; justify-content: space-between; }
.eyebrow { font-family: ui-monospace, Menlo, monospace; font-size: .68rem;
           letter-spacing: .1em; text-transform: uppercase; color: var(--secondary); }
.lab-body { padding: 1.2rem; display: flex; flex-direction: column; gap: 1.2rem; }
.controls { display: flex; flex-wrap: wrap; gap: 1.4rem; align-items: center; }
.control { display: flex; flex-direction: column; gap: .3rem;
           min-width: 12rem; flex: 1 1 12rem; }
.control label { font-family: ui-monospace, Menlo, monospace; font-size: .78rem;
                 color: var(--secondary); }
.control label b { color: var(--content); font-weight: 600;
                   font-variant-numeric: tabular-nums; }
.lab input[type="range"] { width: 100%; accent-color: var(--bad); }
.specimen { border: 1px solid var(--border); border-radius: var(--radius);
            padding: 1.1rem 1.2rem; font-family: ui-monospace, Menlo, monospace;
            font-size: .82rem; line-height: 2.1; overflow-wrap: anywhere; }
.specimen mark { background: var(--mark-wash); color: var(--mark-ink);
                 box-shadow: inset 0 -.42em 0 -.06em var(--mark);
                 padding: .05em .1em; border-radius: 1px; }
.specimen .cut { display: inline-block; width: 0; border-left: 2px solid var(--bad);
                 height: 1.35em; vertical-align: -.35em; margin: 0 .18em; }
.readout { display: flex; flex-wrap: wrap; border-top: 1px solid var(--border); }
.stat { flex: 1 1 8rem; padding: .9rem 1.1rem; border-right: 1px solid var(--border); }
.stat:last-child { border-right: 0; }
.stat .k { display: block; font-family: ui-monospace, Menlo, monospace;
           font-size: .66rem; letter-spacing: .1em; text-transform: uppercase;
           color: var(--secondary); }
.stat .v { display: block; font-size: 1.7rem; font-weight: 600; line-height: 1.3;
           font-variant-numeric: tabular-nums; }
.stat.lead .v { color: var(--bad); }
</style>
Cut this paragraph every 90 characters and one of the cuts lands inside a date. Ask
*when does the Northeast file?* and the piece that comes back ends at *"files annually
instead, on 31"*, with the month in the piece after it.

Where those cuts go is decided once, before anyone asks anything, and no later stage can
undo a bad one. `chunking-lab` measures what a given set of cuts costs. What follows is
what it found over fourteen strategies, five sets of documents and 652 questions with
marked answers — including three things we believed at the start that did not survive
being measured.

## What chunking is

A program that answers questions about your company's documents cannot read all of them
for every question — there is far too much text — so it stores them in pieces and fetches
only the pieces that look relevant. Those pieces are called
**chunks**, and cutting the documents into them is called **chunking**.

Nothing in a document says where the cuts go. The defaults we benchmarked against are
character counts — 800 characters, with 400 repeated between neighbours — and a counter
has no way to avoid landing inside the sentence that holds an answer. When it does, the
answer is spread across two chunks, and each of them reads as complete on its own.

## The four stages

Four stages run between a document and an answer. Only the first is the subject here,
and separating it from the other three is most of what makes the measurement possible.

<figure class="wcf">
<svg class="dia" viewBox="0 0 860 180" role="img"
  aria-label="Documents are cut into chunks by the chunker, the chunks are stored in an index, a retriever ranks them against a question, and the top few go to the model, which writes the answer. The chunker is the only stage this tool varies.">
  <defs>
  <marker id="a1" viewBox="0 0 10 10" refX="9" refY="5"
  markerWidth="7" markerHeight="7" orient="auto-start-reverse">
  <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
  </marker>
  </defs>
  <rect class="box" x="8"   y="30" width="120" height="56" rx="3"/>
  <rect class="box-hi" x="188" y="30" width="120" height="56" rx="3"/>
  <rect class="box" x="368" y="30" width="120" height="56" rx="3"/>
  <rect class="box" x="548" y="30" width="120" height="56" rx="3"/>
  <rect class="box" x="728" y="30" width="120" height="56" rx="3"/>
  <text x="68"  y="56" text-anchor="middle">Documents</text>
  <text x="248" y="56" text-anchor="middle">Chunker</text>
  <text x="428" y="56" text-anchor="middle">Index</text>
  <text x="608" y="56" text-anchor="middle">Retriever</text>
  <text x="788" y="56" text-anchor="middle">Model</text>
  <text class="lbl" x="68"  y="74" text-anchor="middle">your corpus</text>
  <text class="lbl" x="248" y="74" text-anchor="middle">picks the cuts</text>
  <text class="lbl" x="428" y="74" text-anchor="middle">stores chunks</text>
  <text class="lbl" x="608" y="74" text-anchor="middle">ranks them</text>
  <text class="lbl" x="788" y="74" text-anchor="middle">writes the answer</text>
  <line class="ln" x1="128" y1="58" x2="184" y2="58" marker-end="url(#a1)"/>
  <line class="ln" x1="308" y1="58" x2="364" y2="58" marker-end="url(#a1)"/>
  <line class="ln" x1="488" y1="58" x2="544" y2="58" marker-end="url(#a1)"/>
  <line class="ln" x1="668" y1="58" x2="724" y2="58" marker-end="url(#a1)"/>
  <text class="lbl" x="156" y="50" text-anchor="middle">text</text>
  <text class="lbl" x="336" y="50" text-anchor="middle">chunks</text>
  <text class="lbl" x="516" y="50" text-anchor="middle">reads</text>
  <text class="lbl" x="696" y="50" text-anchor="middle">top k</text>
  <rect class="box" x="548" y="128" width="120" height="34" rx="3"/>
  <text x="608" y="150" text-anchor="middle">Question</text>
  <line class="ln" x1="608" y1="126" x2="608" y2="90" marker-end="url(#a1)"/>
  <line class="ln" x1="788" y1="88" x2="788" y2="124" marker-end="url(#a1)"/>
  <text x="788" y="150" text-anchor="middle">Answer</text>
  <text class="cap" x="248" y="106" text-anchor="middle">the only stage this tool moves</text>
  </svg>
<figcaption><b>Chunking happens once, before anyone asks anything.</b> By the time a question arrives the boundaries are fixed, and neither the retriever nor the model can put back what a cut separated. It is also why judging the final answer tests a chunker poorly: three stages contributed to that answer, and it does not say which one to blame.</figcaption>
</figure>

## Why smaller chunks are not the fix

The obvious correction is tighter chunks, so that less irrelevant text comes back around
each answer. Shrink them far enough and answers get severed more often instead, and a
chunk carrying no surrounding context can be unreadable by itself: one starting *"It
files annually instead"* has lost the word *Northeast*. Where the trade falls depends on
the documents and on the questions people ask.

## Six ways to decide where to cut

Every chunking strategy answers one question — *what tells me where a boundary goes?* —
and the answers get more expensive as they get better informed. A character counter is
free; a model reading the document and choosing costs one call per window.

The six families, ordered by what supplies the boundary signal, which is also the cost
ordering:

| family | what decides a boundary | cost |
| --- | --- | --- |
| fixed-size | a character counter | free |
| recursive | a priority list of separators: paragraph, then line, then sentence | free |
| structural | the document's own markup — headings, tables, code fences | free |
| semantic | where the meaning of consecutive sentences changes | one embedding pass |
| LLM | a model reads the text and picks | one call per window |
| context-augmenting | **nothing — it does not move the boundaries at all** | varies |

The last row belongs to a different kind of thing. It leaves the boundaries where they
were and separates two units that are normally the same one: **what gets stored for
searching** and **what gets handed to the model**.

<figure class="wcf">
<svg class="dia" viewBox="0 0 760 285" role="img"
  aria-label="Families one to five move the chunk boundaries: two strips of the same document show cuts in different places. Family six leaves the boundaries identical and instead indexes one small piece while returning a wider one.">
  <text class="mono" x="8" y="16">FAMILIES 1–5 — the boundaries move</text>
  <rect class="strip" x="8" y="30" width="640" height="26" rx="2"/>
  <line class="ln-cut" x1="168" y1="26" x2="168" y2="60"/>
  <line class="ln-cut" x1="328" y1="26" x2="328" y2="60"/>
  <line class="ln-cut" x1="488" y1="26" x2="488" y2="60"/>
  <text class="lbl" x="660" y="48">every 400 chars</text>
  <rect class="strip" x="8" y="76" width="640" height="26" rx="2"/>
  <line class="ln-cut" x1="96"  y1="72" x2="96"  y2="106"/>
  <line class="ln-cut" x1="286" y1="72" x2="286" y2="106"/>
  <line class="ln-cut" x1="410" y1="72" x2="410" y2="106"/>
  <line class="ln-cut" x1="556" y1="72" x2="556" y2="106"/>
  <text class="lbl" x="660" y="94">at each heading</text>
  <line class="ln-soft" x1="8" y1="128" x2="752" y2="128"/>
  <text class="mono" x="8" y="158">FAMILY 6 — the boundaries stay; the unit changes</text>
  <rect class="strip" x="8" y="172" width="640" height="26" rx="2"/>
  <line class="ln-cut" x1="168" y1="168" x2="168" y2="202"/>
  <line class="ln-cut" x1="328" y1="168" x2="328" y2="202"/>
  <line class="ln-cut" x1="488" y1="168" x2="488" y2="202"/>
  <text class="lbl" x="660" y="190">identical cuts</text>
  <rect class="holds" x="168" y="172" width="160" height="26" rx="2"/>
  <path class="ln-cut" d="M168,206 L168,214 L328,214 L328,206"/>
  <text class="cap" x="248" y="230" text-anchor="middle">stored for searching</text>
  <path class="dash" d="M8,240 L8,248 L488,248 L488,240"/>
  <text class="lbl" x="248" y="264" text-anchor="middle">handed to the model</text>
  </svg>
<figcaption><b>Only the bottom row separates the two units.</b> It indexes one sentence, short enough to match a question cleanly, and returns that sentence together with its surroundings, so the model has enough to read it. A score for this family has to say which of the two units it measured.</figcaption>
</figure>

## The measurement

Running the whole system and judging its answers confounds the chunker with the
retriever and the language model, and tells you nothing about any of them separately.
The alternative is to mark the exact characters that answer each question, then ask a
narrower question about the cuts alone: given where this chunker put its boundaries,
what is the best any retriever could do?

That number is **Precision Ω**, "precision omega". Take every chunk containing any part
of the answer, assume you retrieved all of them, and ask what fraction of the text you
are holding is the answer. It is a ceiling. No retriever can beat it, because the
boundaries were fixed before it ran.

<figure class="wcf">
<svg class="dia" viewBox="0 0 760 300" role="img"
  aria-label="Recall, precision and IoU are all divided by the chunks the retriever happened to pick, so they change when the retriever changes. Precision Omega is divided by every chunk containing part of the answer, which is decided by the boundaries alone, with no retriever involved.">
  <defs>
  <marker id="a3" viewBox="0 0 10 10" refX="9" refY="5"
  markerWidth="7" markerHeight="7" orient="auto-start-reverse">
  <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
  </marker>
  </defs>
  <text class="mono" x="8" y="18">RECALL · PRECISION · IoU</text>
  <text class="lbl" x="8" y="36">divided by whatever the retriever picked</text>
  <rect class="strip" x="8" y="52" width="680" height="34" rx="2"/>
  <rect class="picked" x="144" y="52" width="136" height="34"/>
  <rect class="picked" x="280" y="52" width="136" height="34"/>
  <line class="ln-soft" x1="144" y1="52" x2="144" y2="86"/>
  <line class="ln-soft" x1="280" y1="52" x2="280" y2="86"/>
  <line class="ln-soft" x1="416" y1="52" x2="416" y2="86"/>
  <line class="ln-soft" x1="552" y1="52" x2="552" y2="86"/>
  <rect class="goldbar" x="330" y="58" width="70" height="22" rx="2"/>
  <text class="lbl" x="76"  y="104" text-anchor="middle">chunk 1</text>
  <text class="lbl" x="212" y="104" text-anchor="middle">chunk 2</text>
  <text class="lbl" x="348" y="104" text-anchor="middle">chunk 3</text>
  <text class="lbl" x="484" y="104" text-anchor="middle">chunk 4</text>
  <text class="lbl" x="620" y="104" text-anchor="middle">chunk 5</text>
  <line class="ln" x1="212" y1="128" x2="212" y2="94" marker-end="url(#a3)"/>
  <line class="ln" x1="348" y1="128" x2="348" y2="94" marker-end="url(#a3)"/>
  <text x="280" y="144" text-anchor="middle">the two the retriever chose</text>
  <text class="lbl" x="700" y="74">the answer</text>
  <line class="ln-soft" x1="8" y1="168" x2="752" y2="168"/>
  <text class="mono" x="8" y="196">PRECISION Ω</text>
  <text class="lbl" x="8" y="214">divided by every chunk that holds part of the answer</text>
  <rect class="strip" x="8" y="230" width="680" height="34" rx="2"/>
  <rect class="holds" x="280" y="230" width="136" height="34"/>
  <line class="ln-soft" x1="144" y1="230" x2="144" y2="264"/>
  <line class="ln-soft" x1="280" y1="230" x2="280" y2="264"/>
  <line class="ln-soft" x1="416" y1="230" x2="416" y2="264"/>
  <line class="ln-soft" x1="552" y1="230" x2="552" y2="264"/>
  <rect class="goldbar" x="330" y="236" width="70" height="22" rx="2"/>
  <text class="cap" x="348" y="286" text-anchor="middle">chosen by the cuts, not by the retriever</text>
  <text class="lbl" x="700" y="252">the answer</text>
  </svg>
<figcaption><b>The denominator is what separates these four numbers.</b> The top three change if you swap the retriever, tune the query or ask for more chunks, so they measure the whole system. Precision Ω divides by every chunk holding part of the answer, a set the boundaries alone decide.</figcaption>
</figure>

<figure class="lab">
  <div class="lab-head">
  <span class="eyebrow">Live · fixed-size chunking</span>
  <span class="eyebrow" id="lab-spec">90 characters, no overlap</span>
  </div>
  <div class="lab-body">
  <div class="controls">
  <div class="control">
  <label for="size">chunk size <b><span id="size-v">90</span> chars</b></label>
  <input type="range" id="size" min="30" max="400" step="10" value="90">
  </div>
  <div class="control">
  <label for="ov">overlap <b><span id="ov-v">0</span> chars</b></label>
  <input type="range" id="ov" min="0" max="200" step="10" value="0">
  </div>
  </div>
  <div class="specimen" id="doc" style="box-shadow:none;border-color:var(--rule)"></div>
  </div>
  <div class="readout">
  <div class="stat lead">
  <span class="k">Precision Ω</span>
  <span class="v" id="m-omega">—</span>
  </div>
  <div class="stat">
  <span class="k">chunks holding the answer</span>
  <span class="v" id="m-touch">—</span>
  </div>
  <div class="stat">
  <span class="k">answer severed</span>
  <span class="v" id="m-sev">—</span>
  </div>
  <div class="stat">
  <span class="k">chunks in total</span>
  <span class="v" id="m-n">—</span>
  </div>
  </div>
  </figure>

<script>
(function () {
  const TEXT =
    "The Northeast filing deadline is the one exception. Every other region reports " +
    "quarterly, on the fifteenth day of the month following the close of the quarter. " +
    "The Northeast files annually instead, on 31 March, because its state-level " +
    "requirements were never harmonised with the federal calendar. Regional managers " +
    "should not expect Northeast figures in the quarterly consolidation.";
  const ANSWER = "The Northeast files annually instead, on 31 March";
  const GOLD = [TEXT.indexOf(ANSWER), TEXT.indexOf(ANSWER) + ANSWER.length];

  const doc = document.getElementById("doc");
  const sizeEl = document.getElementById("size");
  const ovEl = document.getElementById("ov");

  function chunks(size, overlap) {
    const stride = Math.max(1, size - overlap);
    const out = [];
    for (let start = 0; start < TEXT.length; start += stride) {
      out.push([start, Math.min(start + size, TEXT.length)]);
      if (start + size >= TEXT.length) break;
    }
    return out;
  }

  // Union of ranges, so a character is never counted twice.
  function union(ranges) {
    if (!ranges.length) return [];
    const sorted = ranges.slice().sort((a, b) => a[0] - b[0]);
    const merged = [sorted[0].slice()];
    for (const [s, e] of sorted.slice(1)) {
      const last = merged[merged.length - 1];
      if (s <= last[1]) last[1] = Math.max(last[1], e);
      else merged.push([s, e]);
    }
    return merged;
  }
  const width = (rs) => rs.reduce((n, [s, e]) => n + (e - s), 0);

  function render() {
    const size = +sizeEl.value;
    const overlap = Math.min(+ovEl.value, size - 10);
    ovEl.max = Math.max(0, size - 10);

    const cs = chunks(size, overlap);
    const touching = cs.filter(([s, e]) => s < GOLD[1] && e > GOLD[0]);
    // Precision omega: the answer, over every chunk that holds any of it.
    const omega = touching.length ? (GOLD[1] - GOLD[0]) / width(union(touching)) : 0;

    // A cut strictly inside the answer severs it.
    const severed = cs.filter(([s]) => s > GOLD[0] && s < GOLD[1]).length;

    document.getElementById("size-v").textContent = size;
    document.getElementById("ov-v").textContent = overlap;
    document.getElementById("lab-spec").textContent =
      size + " characters, " + (overlap ? overlap + " overlap" : "no overlap");
    document.getElementById("m-omega").textContent = (omega * 100).toFixed(1) + "%";
    document.getElementById("m-touch").textContent = touching.length;
    document.getElementById("m-sev").textContent = severed ? "yes" : "no";
    document.getElementById("m-n").textContent = cs.length;

    // Draw: boundaries are the distinct chunk starts, past zero.
    const cuts = new Set(cs.map(([s]) => s).filter((s) => s > 0));
    let html = "";
    for (let i = 0; i < TEXT.length; i++) {
      if (cuts.has(i)) html += '<span class="cut"></span>';
      const inGold = i >= GOLD[0] && i < GOLD[1];
      const ch = TEXT[i].replace("&", "&amp;").replace("<", "&lt;");
      html += inGold ? '<mark>' + ch + '</mark>' : ch;
    }
    doc.innerHTML = html;
  }

  sizeEl.addEventListener("input", render);
  ovEl.addEventListener("input", render);
  render();
})();
</script>

Drag the sliders. The answer is 49 characters long, and everything retrieved alongside
it is padding. Overlap is meant to stop answers being severed, and it does, at the cost
of carrying every duplicated character in the denominator.

## Why recall lies

The intuitive measure is **recall**: did the answer come back at all? On its own it can
be satisfied by returning everything, and one of the strategies we tested does that by
accident.

`structural` splits documents at their headings. Run it on a transcript of a speech,
which has no headings, and it produces a single chunk: the whole document, holding the
answer to every question anyone can ask of it, and scoring 100% on recall.

On the State of the Union transcript — 76 questions with marked answers, BM25 retrieval:

| strategy | recall | Precision Ω |
| --- | --- | --- |
| structural | 100.00 | 0.39 |
| recursive:200 | 68.53 | 83.80 |
| fixed:800/400 | 89.80 | 12.42 |

To answer *when does the Northeast file?*, that strategy hands over the entire speech. A
metric treating a chunk as one yes-or-no unit scores it a hit, because it never looks
inside the chunk to see how much of what came back was the answer.

## Two ways a chunk goes wrong

We built a set of documents designed to break specific strategies, to check that the
measurements caught what they were meant to. That turned up a distinction we had been
running together, though only one of them is about where the boundary went.

<figure class="wcf">
<svg class="dia" viewBox="0 0 760 300" role="img"
  aria-label="Severing: a boundary falls inside the answer, so neither chunk holds all of it. Orphaning: the answer sits intact inside one chunk, but the heading naming its subject is in the chunk before, so the chunk is correct and still unusable.">
  <defs>
  <marker id="a4" viewBox="0 0 10 10" refX="9" refY="5"
  markerWidth="7" markerHeight="7" orient="auto-start-reverse">
  <path d="M0,0 L10,5 L0,10 z" fill="var(--cut)"/>
  </marker>
  </defs>
  <text class="mono" x="8" y="18">SEVERING — the answer is cut in two</text>
  <rect class="strip" x="8" y="34" width="680" height="34" rx="2"/>
  <line class="ln-soft" x1="188" y1="34" x2="188" y2="68"/>
  <line class="ln-soft" x1="528" y1="34" x2="528" y2="68"/>
  <rect class="goldbar" x="268" y="40" width="180" height="22" rx="2"/>
  <line class="ln-cut" x1="358" y1="26" x2="358" y2="76"/>
  <text class="cap" x="358" y="92" text-anchor="middle">the boundary lands inside the answer</text>
  <text class="lbl" x="8" y="118">Fix: put the boundary somewhere else — bigger chunks, better separators, overlap.</text>
  <line class="ln-soft" x1="8" y1="140" x2="752" y2="140"/>
  <text class="mono" x="8" y="170">ORPHANING — the answer is whole, and still useless</text>
  <rect class="strip" x="8" y="186" width="680" height="34" rx="2"/>
  <line class="ln-soft" x1="248" y1="186" x2="248" y2="220"/>
  <line class="ln-soft" x1="588" y1="186" x2="588" y2="220"/>
  <rect class="holds" x="60" y="192" width="150" height="22" rx="2"/>
  <text class="mono" x="135" y="207" text-anchor="middle">## Northeast</text>
  <rect class="goldbar" x="330" y="192" width="200" height="22" rx="2"/>
  <text class="lbl" x="430" y="207" text-anchor="middle">files on 31 March</text>
  <path class="dash" d="M330,232 C300,258 180,254 140,224" marker-end="url(#a4)"/>
  <text class="cap" x="248" y="274" text-anchor="middle">the subject it depends on is in the chunk before</text>
  <text class="lbl" x="8" y="296">Fix: none at the boundary. The boundary is already right.</text>
  </svg>
<figcaption><b>The second failure is why the context-augmenting family exists.</b> No placement of the boundary rescues a chunk reading “files on 31 March”, because the region name was never inside it. The remedies are to widen what the model receives, or to write the missing context into the chunk before storing it.</figcaption>
</figure>

Five of those documents are marked `hostile` in the test suite. If a strategy stops
failing one of them, the test fails, on the reading that the fixture has drifted.

## What we got wrong about Precision Ω

Precision Ω comes from a technical report by Chroma. We implemented it from a
description of the report rather than from the report, because the session doing the
research could not reach the original. The description was ambiguous in one place, and we
guessed.

We took it to mean *the smallest set of chunks that covers the answer*, where it means
*every chunk containing any part of the answer*. The two readings agree until chunks
overlap, at which point the second set is larger and the score is lower.

The report publishes its own numbers, and Precision Ω needs no retriever to compute, so
both readings could be run against it. Over the same 472 questions, recursive character
splitting:

| size / overlap | published | correct reading | our first guess |
| --- | --- | --- | --- |
| 800 / 400 | 6.7 | 6.7 | 8.6 |
| 400 / 200 | 13.9 | 13.9 | 16.8 |
| 400 / 0 | 17.7 | 17.7 | 17.8 |
| 200 / 0 | 29.9 | 29.9 | 30.7 |

With no overlap our reading is within 0.1 of the published figure. With overlap it is
about 28% high — and overlap is the setting people reach for when answers keep getting
severed. The wrong version passed every test we had thought to write, and it would have
recommended turning overlap up.

A published table caught what our own tests could not. The download was 1.6 MB.

## What the evidence says

### The boring default is hard to beat

Splitting on paragraph and sentence boundaries at around 200 tokens, with no overlap,
sits near the top on every measure we ran. It never won outright and it never placed
badly. The elaborate alternatives are competing for what is left.

### Overlap is not free

Duplicating text between neighbouring chunks does reduce severed answers. It also
inflates the index and drags the ceiling down: 13.9 against 17.7 for otherwise identical
settings.

### "Semantic" chunking depends on the documents

Using an embedding model to find topic boundaries is the expensive option. The study
cited at the foot of this post found no consistent benefit from it, and the finding
underneath that headline is the more useful one: on documents that jump between
unrelated topics it wins by a wide margin — 81.9 against 69.5 on one benchmark — and on
documents where related sentences sit near each other anyway, it loses. Which of those two
your documents look like is not something a benchmark can tell you.

## Things you can check without questions

Some faults show up from the document and the proposed cuts alone, with no questions and
no model. They are cheap enough to run on every candidate configuration before deciding
which ones are worth evaluating properly:

- **Does any text vanish?** A chunker that drops a trailing paragraph produces a run
  that succeeds and scores that look fine.
- **Do cuts land inside sentences?** On one document, fixed-size cutting landed on a
  natural boundary 6% of the time. Splitting at headings: 83%.
- **Are tables and code blocks being cut in half?** A table severed from its header row
  is unusable, and nothing about the average chunk size will tell you.
- **Do chunks start with "as described above"?** A chunk opening with a reference to
  something outside itself has been cut off from what it needs.

These checks can show that a configuration is broken. Ranking two workable
configurations needs the questions, because "good chunking" is defined relative to what
people ask. That was our position before we tested it. The test is the next section, and
it held for a different reason than we expected.

## Do the cheap checks work?

If the free checks are any good, they should put configurations in the order the
expensive measurement does. So we ran fourteen strategies over two unrelated sets of
documents and 652 questions with marked answers, and took a rank correlation between
each cheap signal and the Precision Ω score. Three of the four answers were not what we
expected.

### Chunk size "predicts" the score, and the prediction is arithmetic

Median chunk length correlates with Precision Ω at −0.95 on every corpus. It is also
close to circular: Precision Ω divides the answer by the chunks holding it, so smaller
chunks raise the number by construction. Reporting that as evidence that cheap screening
works would be reporting the definition back. Every other signal then had to be
re-checked with size held constant, which asks a sharper question: once you know how big
the chunks are, does this signal add anything?

### The check that looked useless was the one that held up

*Do cuts land inside sentences?* correlated at about +0.15, close enough to nothing that
we nearly wrote it off. With size held constant it rises to **+0.44**, and it holds on
both sets of documents. Size had been masking it: strategies that cut cleanly also cut
larger, and the two effects were cancelling out, which is how a correlation of +0.15
can hide a relationship of +0.44.

### The table check does not work, and the reason is the useful part

*Are tables being cut in half?* predicts nothing. Not on either set of documents, not
against any score, and the sign flips depending on which documents you use. It looked
like a broken signal until we drew it.

<figure class="wcf">
<svg class="dia" viewBox="0 0 760 300" role="img"
  aria-label="A table kept whole sits in one large chunk, so the answer is a small fraction of it and Precision Omega is low. Cutting the table into smaller chunks damages the row but shrinks the text around the answer, so Precision Omega goes up. The ceiling metric rewards the damage.">
  <text class="mono" x="8" y="18">TABLE KEPT WHOLE — one chunk</text>
  <rect class="picked" x="8" y="34" width="520" height="38" rx="2"/>
  <rect class="goldbar" x="230" y="42" width="80" height="22" rx="2"/>
  <text class="lbl" x="119" y="60" text-anchor="middle">header + rows</text>
  <text class="lbl" x="419" y="60" text-anchor="middle">more rows</text>
  <text class="cap" x="270" y="90" text-anchor="middle">the answer row</text>
  <text x="560" y="50">chunk 520 chars</text>
  <text x="560" y="68">answer 80 chars</text>
  <text class="mono" x="560" y="90">Precision Ω = 15%</text>
  <line class="ln-soft" x1="8" y1="120" x2="752" y2="120"/>
  <text class="mono" x="8" y="150">CUT EVERY 130 CHARACTERS — the row is severed</text>
  <rect class="strip" x="8" y="166" width="130" height="38" rx="2"/>
  <rect class="holds" x="138" y="166" width="130" height="38" rx="2"/>
  <rect class="holds" x="268" y="166" width="130" height="38" rx="2"/>
  <rect class="strip" x="398" y="166" width="130" height="38" rx="2"/>
  <rect class="goldbar" x="230" y="174" width="80" height="22" rx="2"/>
  <line class="ln-cut" x1="268" y1="158" x2="268" y2="212"/>
  <text class="cap" x="268" y="228" text-anchor="middle">the cut lands inside the row</text>
  <text x="560" y="182">2 chunks = 260 chars</text>
  <text x="560" y="200">answer 80 chars</text>
  <text class="mono" x="560" y="222">Precision Ω = 31%</text>
  <text class="cap" x="380" y="264" text-anchor="middle">The row is now unusable — and the score went UP.</text>
  <text class="lbl" x="380" y="284" text-anchor="middle">A ceiling on precision cannot see damage that makes the text around the answer smaller.</text>
  </svg>
<figcaption><b>The signal is measuring damage the score cannot see.</b> Cutting through a table row destroys it — the numbers lose their column headings — and it also shrinks the text surrounding the answer, which is what raises a precision ceiling. The damage is to whether a retrieved chunk is <em>usable</em>, and Precision Ω does not measure that.</figcaption>
</figure>

A result like this is about a pairing: the signal and the score it was checked against.
"Tables are being shredded" is still a good reason to reject a configuration; it is not a
prediction of where that configuration will place, and we would have gone on assuming it
was.

So the cheap checks earn their place by disqualifying: text that vanished, chunks too
big for the embedding model, a code example cut in half. Two of them — duplication from
overlap, and whether cuts land cleanly — carry information about quality as well. The
rest are faults to fix, and they do not rank anything.

## You don't have the questions yet

Everything above ranks chunking strategies against questions with marked answers. On the
day you start, there is no retriever and nobody has asked anything.

The objection is sharper than it looks, because the ranking does depend on the
questions. Here is one set of documents, fourteen strategies and one set of cuts, ranked
three times by what the question was asking for — 180 questions with marked answers,
over generated documentation:

| strategy | looking up a table row | finding a code example | a sentence of prose |
| --- | --- | --- | --- |
| sentence-window:1 | 1st | 11th | 1st |
| structural | 12th | 2nd | 3rd |
| recursive:200 | 14th | 4th | 6th |
| fixed:200 | 9th | 6th | 5th |

`sentence-window:1` is first for prose and for table lookups, and eleventh of fourteen
for code examples. `structural` is near the top for code and near the bottom for table
rows. `recursive:200` comes last for table lookups while sitting mid-table for
everything else. Nothing about the documents changed, and nothing about the cuts
changed. Only the question did.

Reproduce it with `chunking-lab score --corpus-dir … --by-question-type`. The tool
refuses to print a ranking by question type unless the corpus says what kind each
question is.

### Why the question decides

A chunk's job is to be **the smallest self-contained thing that answers the question**.
Every word of that is load-bearing, and "self-contained" is the one that moves with the
question.

<figure class="wcf">
<svg class="dia" viewBox="0 0 760 260" role="img"
  aria-label="One document containing a rate-limit table followed by another section. Three different questions want three different amounts of it: a single row, the whole table, or two whole sections. No single set of cuts is ideal for all three.">
  <rect class="strip" x="260" y="36" width="480" height="36" rx="2"/>
  <line class="ln-soft" x1="320" y1="36" x2="320" y2="72"/>
  <line class="ln-soft" x1="360" y1="42" x2="360" y2="66"/>
  <line class="ln-soft" x1="440" y1="42" x2="440" y2="66"/>
  <line class="ln-soft" x1="480" y1="42" x2="480" y2="66"/>
  <line class="ln-soft" x1="520" y1="42" x2="520" y2="66"/>
  <line class="ln-soft" x1="560" y1="36" x2="560" y2="72"/>
  <line class="ln-soft" x1="610" y1="36" x2="610" y2="72"/>
  <rect class="goldbar" x="400" y="42" width="40" height="24" rx="2"/>
  <text class="lbl" x="440" y="92" text-anchor="middle">the rate-limit table</text>
  <text class="lbl" x="675" y="92" text-anchor="middle">the next section</text>
  <text class="mono" x="8" y="122">“What is the limit for /v1/documents?”</text>
  <text class="cap" x="8" y="138">wants one row</text>
  <path class="ln-cut" d="M400,116 L400,124 L440,124 L440,116"/>
  <text class="mono" x="8" y="176">“Which endpoints are under 60/min?”</text>
  <text class="cap" x="8" y="192">wants the whole table</text>
  <path class="ln-cut" d="M320,170 L320,178 L560,178 L560,170"/>
  <text class="mono" x="8" y="230">“How do limits relate to quotas?”</text>
  <text class="cap" x="8" y="246">wants two whole sections</text>
  <path class="ln-cut" d="M320,224 L320,232 L740,232 L740,224"/>
  </svg>
<figcaption><b>One document can have three right answers.</b> Cuts tight enough to isolate a single row sever the table for anyone asking about the table as a whole. Cuts loose enough to keep two sections together bury every single-row answer in padding. No set of boundaries serves all three, which is why the question mix has to be part of the choice.</figcaption>
</figure>

### The objection is two objections

They have different answers. Separating them is most of the way to knowing what to do,
and the second one is where the work is.

#### "There is no retriever yet"

Precision Ω is computed from the cuts and the marked answers, with no retriever involved
at any point, so it is available as soon as you can chunk a document. The same goes for
every cheap check above.

#### "Nobody has asked anything yet"

Three things help with this one, and they compound. None of them needs a question anyone
has asked yet.

**Some faults are faults for every question.** Text that landed in no chunk at all is
lost, whatever anyone asks; a chunk too large for the embedding model is silently
truncated for everybody; a code example cut in half is broken for all of them. Those can
be ruled out on day one, for free.

**You know more about the questions than you think.** You know whether you are building
a support bot that answers narrow factual lookups or a research assistant that has to
pull threads across sections. In the table above, that single piece of knowledge moves
`sentence-window:1` between first place and eleventh.

**You can manufacture questions, at increasing fidelity.** Write a few documents where
you plant the answers yourself, so their positions are known. Then have a model
read your real documents and write questions about them — one bill, once, after which
the question set is a file you keep. Then, once there is traffic, use what people asked.

### The trap in the middle step

Having a model generate questions is easy. Getting the *answer positions* is where it
goes wrong, and the obvious approach is the dangerous one.

Ask a model "which characters answer this?" and it returns two numbers. Models are bad
at counting characters, so those numbers are often off, and a wrong position looks like
a right one. Every strategy is then scored against the wrong stretch of text, and the
evaluation is meaningless while appearing to work.

So don't ask for positions. Ask the model to **quote the answer word for word**, then go
and find that quote in the document yourself. Now a wrong answer cannot hide: the quote
either is in the document or it isn't, and if it isn't, the question is discarded.

That costs data, and the price is worth knowing before you budget for it. Running it
against Llama 3.1 8B on a laptop, over two real documents:

| outcome | count | what happened |
| --- | --- | --- |
| kept | 2 | quote found in the document, position verified |
| discarded | 5 | the model paraphrased instead of copying |
| discarded | 1 | the reply wasn’t readable |

Two of eight kept. Five of the six discarded questions failed the same way: told to copy
the text, the model rewrote it slightly.

That number measures how much was lost, not how much was corrupted. The two
questions that survived are right. Asking for positions instead would have produced
eight questions, six of them scored against the wrong characters, with nothing to
indicate it. A low yield is a reason to use a better model for this one step, and a cheap
step to upgrade, because it runs
once per set of documents and the result is a file you keep.

The dishonest version of all this is a tool that prints a single "chunk quality score".
That number is a ranking against some distribution of questions, and if it does not say
which, the assumption is still there and you cannot see it.

So you don't choose once. Rule out the broken configurations for free, take a default —
recursive splitting at around 200 tokens placed near the top of everything we ran — and
ship. Re-rank against real queries when you have them, which is the only ranking that
was ever going to be authoritative, and which the measurements above shorten to an
afternoon's work.

## Is chunking the part worth fixing?

A search system has other moving parts, the retriever most obviously. If swapping that
moves the score further than any amount of work on boundaries, this is advice about the
wrong setting. So each run varies one of the two, holds the other still, and records how
far the score moved.

Across 5 strategies × 3 retrievers × 5 sets of documents, the distance between the best
and the worst setting, as a multiple:

| what you change | typical effect | biggest seen | mattered more than 2× |
| --- | --- | --- | --- |
| the chunking | 5.6× | 14.4× | 15 of 15 times |
| the search engine | 1.3× | 14.9× | 4 of 25 times |

The typical effect says chunking. The largest effect either one ever produced is nearly
the same — 14.4 against 14.9 — and the difference between them is in how often it
happens.

<figure class="wcf">
<svg class="dia" viewBox="0 0 760 250" role="img"
  aria-label="Changing the chunking moved the score by more than double in every one of fifteen cases. Changing the search engine did so in only four of twenty-five, but when it did the effect was as large as chunking's largest.">
  <text class="mono" x="8" y="18">CHANGING THE CHUNKING — 15 cases</text>
  <text class="lbl" x="8" y="36">each block is one case; filled means it moved the score more than 2x</text>
  <rect class="holds" x="8"   y="48" width="42" height="26" rx="2"/>
  <rect class="holds" x="56"  y="48" width="42" height="26" rx="2"/>
  <rect class="holds" x="104" y="48" width="42" height="26" rx="2"/>
  <rect class="holds" x="152" y="48" width="42" height="26" rx="2"/>
  <rect class="holds" x="200" y="48" width="42" height="26" rx="2"/>
  <rect class="holds" x="248" y="48" width="42" height="26" rx="2"/>
  <rect class="holds" x="296" y="48" width="42" height="26" rx="2"/>
  <rect class="holds" x="344" y="48" width="42" height="26" rx="2"/>
  <rect class="holds" x="392" y="48" width="42" height="26" rx="2"/>
  <rect class="holds" x="440" y="48" width="42" height="26" rx="2"/>
  <rect class="holds" x="488" y="48" width="42" height="26" rx="2"/>
  <rect class="holds" x="536" y="48" width="42" height="26" rx="2"/>
  <rect class="holds" x="584" y="48" width="42" height="26" rx="2"/>
  <rect class="holds" x="632" y="48" width="42" height="26" rx="2"/>
  <rect class="holds" x="680" y="48" width="42" height="26" rx="2"/>
  <text class="cap" x="8" y="94">every single time</text>
  <line class="ln-soft" x1="8" y1="116" x2="752" y2="116"/>
  <text class="mono" x="8" y="146">CHANGING THE SEARCH ENGINE — 25 cases</text>
  <text class="lbl" x="8" y="164">same scale; hollow means it barely moved the score at all</text>
  <rect class="holds" x="8"   y="176" width="42" height="26" rx="2"/>
  <rect class="holds" x="56"  y="176" width="42" height="26" rx="2"/>
  <rect class="holds" x="104" y="176" width="42" height="26" rx="2"/>
  <rect class="holds" x="152" y="176" width="42" height="26" rx="2"/>
  <rect class="strip" x="200" y="176" width="42" height="26" rx="2"/>
  <rect class="strip" x="248" y="176" width="42" height="26" rx="2"/>
  <rect class="strip" x="296" y="176" width="42" height="26" rx="2"/>
  <rect class="strip" x="344" y="176" width="42" height="26" rx="2"/>
  <rect class="strip" x="392" y="176" width="42" height="26" rx="2"/>
  <rect class="strip" x="440" y="176" width="42" height="26" rx="2"/>
  <rect class="strip" x="488" y="176" width="42" height="26" rx="2"/>
  <rect class="strip" x="536" y="176" width="42" height="26" rx="2"/>
  <rect class="strip" x="584" y="176" width="42" height="26" rx="2"/>
  <rect class="strip" x="632" y="176" width="42" height="26" rx="2"/>
  <rect class="strip" x="680" y="176" width="42" height="26" rx="2"/>
  <text class="cap" x="8" y="222">4 of 25 — but those four were as big as anything chunking did</text>
  <text class="lbl" x="470" y="222">(15 of the 25 shown)</text>
  </svg>
<figcaption><b>Chunking moved the score in every case tested; the retriever moved it four times in twenty-five.</b> When the retriever did move it, the size of the move matched anything chunking produced.</figcaption>
</figure>

Which is different advice from "chunking matters more". Fix the chunking first, because
it always pays, and don't assume the retriever is settled. All four of the bad cases
were the same failure: the keyword retriever, which matches on shared words, lost the
answer, scoring 1.3 where a meaning-based retriever scored 19.7 on identical chunks.

One explanation we tested and dropped: that the retriever matters more when the chunking
is bad, so that a good chunking makes the retriever irrelevant. It fits the two worst
cases, and we had written the paragraph that way before running the correlation. Across
all 25 measurements, the relationship between how good a chunking was and how much the
retriever mattered is **−0.04**.

## The bug that looked like a discovery

Having built the question generator, the next question was which model should run it. So
we ran four of them on the same documents and counted how many questions each produced
that we could verify — 20 attempts each, on a laptop:

| model | size | usable questions |
| --- | --- | --- |
| qwen2.5 | 7B | 80% |
| phi4 | 14B | 80% |
| llama3.1 | 8B | 70% |
| qwen2.5 | **14B** | 55% |

The bigger Qwen did worse than the smaller one — a result about model size worth
writing up, with a ready explanation: bigger models being worse at copying text verbatim,
perhaps because they are more inclined to improve what they read. The cause turned out to be a
setting we never chose.

### The setting nobody set

Models have a dial called **temperature**, which controls how much randomness goes into
choosing each next word. High temperature is what you want for writing, where variety is
the point. Low is what you want for anything that must come out the same way twice.

We never set it, so it took the server's default, which is tuned for chat. We were
asking models to reproduce text character for character while leaving a little
randomness switched on.

A slightly reworded quote reads well, and it is no longer in the document.
That is also why it looked like a size effect: a bigger model has more ways to phrase
something plausibly, so randomness costs it more.

<figure class="wcf">
<svg class="dia" viewBox="0 0 760 270" role="img"
  aria-label="With randomness on, the 14B model scored 55% and the 7B scored 80%, suggesting bigger is worse. With randomness off, both score 75% and the smallest model leads at 90% — the apparent size effect disappears.">
  <text class="mono" x="8" y="18">RANDOMNESS ON — the server default</text>
  <line class="ln-soft" x1="150" y1="30" x2="150" y2="118"/>
  <line class="ln-soft" x1="700" y1="30" x2="700" y2="118"/>
  <text class="lbl" x="150" y="132" text-anchor="middle">0%</text>
  <text class="lbl" x="700" y="132" text-anchor="middle">100%</text>
  <text class="lbl" x="8" y="48">llama3.1 8B</text>
  <rect class="strip" x="150" y="36" width="385" height="16" rx="2"/>
  <text class="lbl" x="545" y="48">70%</text>
  <text class="lbl" x="8" y="72">qwen2.5 7B</text>
  <rect class="strip" x="150" y="60" width="440" height="16" rx="2"/>
  <text class="lbl" x="600" y="72">80%</text>
  <text class="lbl" x="8" y="96">qwen2.5 14B</text>
  <rect class="holds" x="150" y="84" width="302" height="16" rx="2"/>
  <text class="cap" x="462" y="96">55% — “bigger is worse?”</text>
  <line class="ln-soft" x1="8" y1="150" x2="752" y2="150"/>
  <text class="mono" x="8" y="180">RANDOMNESS OFF — same models, same documents</text>
  <line class="ln-soft" x1="150" y1="192" x2="150" y2="256"/>
  <line class="ln-soft" x1="700" y1="192" x2="700" y2="256"/>
  <text class="lbl" x="8" y="210">llama3.1 8B</text>
  <rect class="goldbar" x="150" y="198" width="495" height="16" rx="2"/>
  <text class="cap" x="655" y="210">90%</text>
  <text class="lbl" x="8" y="234">qwen2.5 7B</text>
  <rect class="strip" x="150" y="222" width="412" height="16" rx="2"/>
  <text class="lbl" x="572" y="234">75%</text>
  <text class="lbl" x="8" y="258">qwen2.5 14B</text>
  <rect class="strip" x="150" y="246" width="412" height="16" rx="2"/>
  <text class="lbl" x="572" y="258">75% — the gap is gone</text>
  </svg>
<figcaption><b>One setting, and the difference disappears.</b> With randomness switched off the two Qwen models score the same and the 8B model leads the group. Nothing about the models changed between the two charts.</figcaption>
</figure>

### What was true

At temperature zero the 14B scores the same as the 7B and takes twice as long. The best
performer was the 8B model that was already installed before any of this started.

All four land between 75% and 90%, which makes them look interchangeable. What differs
is how faithfully they copy:

| model | matched exactly | needed spacing fixed up |
| --- | --- | --- |
| llama3.1 8B | 15 | 8 |
| qwen2.5 7B | 6 | 9 |
| qwen2.5 14B | 6 | 11 |
| phi4 14B | 2 | 15 |

A question whose answer matched character for character rests on firmer evidence than
one recovered by normalising the spacing. llama3.1 matched 15 of the 23 quotes it
recovered; phi4 matched 2 of 17. The yield percentage does not separate those two.

A default you never made a decision about is still a variable in the experiment, and
this one produced a publishable result that was wrong. What unstuck it was asking whether the problem was in
how we were asking.

## The words, in plain language

- **chunk** A piece of a document, stored and fetched as a unit. Cutting the documents
  into them is **chunking**.
- **boundary / cut** Where one chunk ends and the next begins. A chunking strategy is a
  rule for placing them.
- **span** A chunk described by *where it is*: character 4,120 to 4,533. Storing
  positions instead of strings is what makes it possible to ask how much of a chunk is
  the answer.
- **retriever** The component that picks which chunks to fetch for a question. Ours is
  deliberately simple and never changes, so any difference we measure comes from the
  chunking.
- **k** How many chunks get fetched per question. Raising it raises recall and lowers
  precision, so a score quoted without its k cannot be compared with another.
- **embedding** A list of numbers representing a passage's meaning, so that passages
  about similar things end up near each other. It is how a system matches "retries" to
  text that says "backoff".
- **token** The unit a model reads — a short word or a fragment of one. Chunk
  sizes are usually quoted in tokens; we measure in plain characters, which needs no
  model and cannot drift.
- **overlap** Repeating the end of each chunk at the start of the next, so an answer
  sitting on a boundary appears whole somewhere. It works, and it inflates your storage
  and lowers the precision ceiling.
- **gold span** The characters that answer a question, marked in advance. The unit is
  characters: not which document, and not which chunk. Everything here is scored against
  these.
- **recall** Of the answer, how much came back. Easy to take to 100% by returning
  everything, which is why it is never quoted alone.
- **precision** Of what came back, how much was the answer. The rest is padding the
  model has to read past.
- **IoU** "Intersection over union" — one number balancing the two above, by dividing
  what you got right by everything involved either way.
- **Precision Ω** The **ceiling**. Given where the boundaries fell, the best any
  retriever could do. It involves no retriever at all, which is what makes it a
  statement about the chunking and not about the system around it.
- **severing** A boundary lands inside the answer, so no single chunk holds all of it. A
  boundary problem, fixable by moving the boundary.
- **orphaning** The chunk holding the answer is intact and useless anyway, because the
  heading, table header or noun it depends on is in a different chunk. The boundary is
  already in the right place, so moving it does not help.
- **intrinsic / extrinsic** Intrinsic checks need only the document and the proposed
  cuts. Extrinsic checks need questions with marked answers. Intrinsic checks **screen**:
  they can tell you a configuration is broken. Only extrinsic checks can **rank** two
  reasonable ones, because good chunking is defined relative to the questions people ask.
- **rank correlation** One number, −1 to +1, for "do these two things put the list in
  the same order?" Order, not value, because the question is whether screening would
  have picked the same winner.
- **keyword vs meaning search** Keyword search ranks by words the question and the text
  share, which is fast and exact and blind to synonyms. Meaning-based search compares numeric
  representations, so "retries" can match text that says "backoff". Combining them by
  position, not by score, avoids having to make two incompatible scales comparable.
- **temperature** How much randomness a model uses when choosing each next word. High
  for writing, where variety is the point. Zero for anything that must come out the same
  way twice, such as copying text or extracting fields. Leaving it at a default tuned for
  chat is an invisible way to break an extraction task.
- **yield** Of the questions a model was asked to produce, the fraction that could be
  verified against the document. It turns "is this model good enough?" into a number, and
  a low one means there is less evaluation data while what survives is still sound.
- **question mix** The distribution of things people ask — narrow lookups, comparisons
  across a document, open synthesis. It moves a strategy between first and eleventh place
  on the same documents, which makes it an input to choosing a chunker.
- **controlling for something** Asking what is left of a relationship once a known cause
  is removed. Here: chunk size affects every signal below it, so the useful question is what a
  signal tells you after you already know how big the chunks are. Holding size constant
  raised the sentence-boundary signal from +0.15 to +0.44.

## The tool

`chunking-lab` is the instrument all of the above came out of. It takes documents, runs
them through a dozen chunking strategies, and scores each one. The whole default path is
deterministic and offline, and needs neither an API key nor a model.

```text
# screen strategies with no questions and no model
chunking-lab metrics report.md \
    --strategy fixed:800/400 --strategy recursive:200 --strategy structural

# rank them against marked answers
chunking-lab score --strategy recursive:200 --strategy structural

# check the headline metric against a published table
make chunking-benchmark
```

None of the mechanism is new. Screening chunkers on cheap structural signals is
published work, and so is correlating those signals against retrieval quality. What this
one does differently is needing no model to do it, and reproducing someone else's
published numbers to the decimal, so you can tell whether to believe it.

Measuring changed three of our beliefs. The metric we had implemented from a description
was 28% high wherever overlap was switched on. The table-integrity check, which looked
like the most principled of the cheap signals, carried no ranking information at all.
And the sentence-boundary check that correlated at +0.15 was the sturdiest signal we
have, once chunk size was held constant.

<div class="colophon">
  <p><strong>The code.</strong> <code>chunking-lab</code> lives in
  <a href="https://github.com/pavanrao/data-tools">pavanrao/data-tools</a> under
  <code>tools/chunking-lab/</code>, with its design record in <code>docs/</code>.
  Precision Ω is defined in Chroma's <em>Evaluating Chunking Strategies for
  Retrieval</em>; the benchmark it reproduces is
  <code>brandonstarxel/chunking_evaluation</code> (MIT). The semantic chunking result
  is Qu, Tu &amp; Bao, <em>Is Semantic Chunking Worth the Computational Cost?</em>
  (NAACL 2025 Findings).</p>
  <p><strong>How this was made.</strong> <code>chunking-lab</code> and this write-up
  were both built with Claude Code. Every figure here is output from a run against the
  corpora the repository builds.</p>
</div>
