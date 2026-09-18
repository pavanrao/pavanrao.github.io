# CLAUDE.md

Hugo site with the PaperMod theme as a submodule. Pushing to `master` deploys it
through `.github/workflows/deploy.yml`.

## Before a post is published

Review it for the habits that make prose read as machine-written, using ai-sniffer
from `pavanrao/data-tools` (`tools/ai-sniffer/`):

- `ai-sniffer check content/posts/POST.md` counts hedges, emphasis words, reader
  instructions and one-sentence paragraphs.
- The ai-sniffer agent, run with Sonnet, finds the structural habits: antithesis,
  fragments, dramatic beats and section closers. In its eval on 2026-09-15 Haiku
  missed most of them.

The reviewer quotes each habit with its line and never rewrites. Fix them with real
detail, and never invent a belief, a reaction or a fact to fill a gap.

Code links in a post go to the repository that holds the code.

## Inline HTML in a post

Goldmark ends a raw HTML block at the first blank line. Anything after that blank
line is parsed as Markdown again, so a line indented four spaces or more becomes a
code block. An inline `<svg>` copied from an HTML source usually has both, and the
result is a diagram that renders down to its first blank line with the rest of its
source printed on the page. Strip the blank lines and keep indentation under four
spaces.

PaperMod styles `.md-content figure>figcaption` bold and in `--primary`. A caption
rule needs at least that much specificity or the whole caption renders bold.
