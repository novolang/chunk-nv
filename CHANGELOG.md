# Changelog

All notable changes to chunk-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- markdown-nv: `^0.0.1` to `^0.1.0`.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `chunksize` — the load-bearing interface. The tokenizer is a
  `fn(Str) -> Int` VALUE carried inside the `ChunkByTokens` unit, not a
  dependency and not a trait.

  Not a dependency, because a package that depended on one would force
  every consumer onto a single model's vocabulary, and a pipeline
  chunking for one embedding model and one chunking for another need
  different counts from the same document.

  Not a trait, and the reason is the effect system rather than taste:
  SPEC § 5.6 gives effect parameters to traits and not to free
  functions, so a `core` function that called an effectful trait method
  would inherit a concrete row and leave the `core` budget — and a
  tokenizer that loads its vocabulary from a file is exactly such a
  method. A `fn(Str) -> Int` cannot carry an effect, so the caller
  spends its `[fs]` once and hands in a closure.

  The unit is carried ONCE on `ChunkLimit` and the size and the overlap
  are both under it, because "500 with 50 of overlap" is only
  meaningful if both numbers are in the same unit.
- `chunkspan` — a chunk is a pair of byte offsets into the caller's
  source. The reason is citation and not memory: a retrieval system's
  whole job is to say where an answer came from, and a copied string
  cannot. `covers_source` and `rejoin` make the invariant checkable,
  which matters because the bug a splitter is most likely to have is
  eating a separator it never puts anywhere.
- `chunktext` — three splitters and two written-down rules, because
  nothing specifies chunking and the reference implementations
  disagree with each other on a paragraph longer than the limit: no
  chunk exceeds the limit unless one indivisible unit does, and with
  an overlap of zero the chunks concatenate back to the source. The
  separator LIST IS THE CONFIGURATION, and its order encodes what the
  document is.
- `chunkmd` — Markdown over markdown-nv's parse rather than over a
  separator list with `"\n## "` in it, which is wrong in three ways
  that all show up in real documentation: a fenced code block can
  contain blank lines and lines beginning with `#`; a table split
  between its header and its body leaves two useless halves; and a
  chunk from the middle of a section has lost the heading that says
  what it is about. `ChunkMdPiece` carries the span AND the heading
  trail SEPARATELY, because prepending the headings makes the text no
  longer contiguous with the source and collapsing the two would
  destroy the citation.
- `chunkfault` — seven refusals, all about the configuration rather
  than the text. There is no text this package refuses. An overlap not
  smaller than the size is the one worth naming: the step is the
  difference, so that configuration emits the same chunk for ever, and
  clamping it would be choosing a retrieval parameter for the caller
  silently.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  chunk-nv.<module>.<fn>`.
- **tokenizers-nv is named in the README and not depended on.** Its
  counting function is what `chunksize.by_tokens` takes, and a program
  that chunks by characters carries no vocabulary at all.
- **Semantic chunking is not declared.** Splitting where the meaning
  changes needs embeddings of candidate splits, which is
  `embeddings-nv`'s work and a package on top of both.
- **Sentence segmentation is a heuristic.** The separator lists hold
  `". "` and `"。"`, which is wrong on abbreviations. A trained
  segmenter is a different package.
