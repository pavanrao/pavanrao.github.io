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
