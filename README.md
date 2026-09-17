# chunk-nv

A retrieval system stores a document in pieces, embeds each piece, and
searches the embeddings. Cutting the document into those pieces is
called chunking, and how it is done decides what the system can find.
This package splits text for that purpose. The reference
implementations it is measured against are the Rust crate
[`text-splitter`](https://docs.rs/text-splitter) and
[LangChain's text splitters](https://python.langchain.com/docs/how_to/#text-splitters).

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What chunking is

A **chunk** is a piece of a document small enough to embed. Every
embedding model has a maximum input length, so the limit is real, and
smaller chunks make a search more precise and less complete: a question
whose answer spans two chunks finds neither.

The **size** of a chunk is measured in a **unit**: characters, bytes,
or **tokens**. A token is what a model's tokenizer produces, and it is
the unit that matters, because it is the one the model's limit is in.
Counting them requires the model's own vocabulary, so this package
takes a counting function from the caller rather than carrying a
tokenizer.

**Overlap** is how much of one chunk's tail the next chunk repeats. It
exists because a sentence that falls on a boundary is in neither
chunk's middle, and a chunk that begins mid-sentence embeds badly. The
distance between the start of one chunk and the start of the next is
the **step**, which is the size minus the overlap.

There are four ways this package decides where a chunk ends.

| Splitter | Cuts at |
| --- | --- |
| Fixed | Wherever the limit is reached |
| Recursive | The first separator in an ordered list that produces pieces that fit |
| Token | The same, measured by the caller's counter |
| Markdown | The document's own structure: headings, and never inside a code block or a table |

Every chunk is reported as a **span**: a pair of byte offsets into the
source the caller still holds, not a copied string. That is what lets a
retrieval system say which part of which document an answer came from,
which is most of what such a system is for.

## Install

```
novo pkg add chunk-nv
```

## Example

```novo
use std.str
use std.list
use chunksize
use chunktext
use chunkspan

fn main() [io]
    let document = "First paragraph.\n\nSecond paragraph.\n"

    // The tokenizer is a function the caller supplies. This one is a
    // stand-in; a real one closes over a vocabulary the caller loaded.
    let count_tokens = s => str.len(s) / 4

    // 256 tokens a chunk, 32 of overlap. Both numbers are in tokens,
    // because the unit is carried once.
    match chunksize.limit(chunksize.by_tokens(count_tokens), 256, 32)
        Err(e) => println("bad configuration: ${e.message()}")
        Ok(limit) =>
            match chunktext.split_prose(document, limit)
                Err(e) => println("cannot split: ${e.message()}")
                Ok(pieces) =>
                    println("${list.len(pieces)} chunk(s)")

                    // A chunk is a span, so it can be cited. This is
                    // the one place the text is copied.
                    let first = list.get(pieces, 0)
                    println("${first.span.start}..${first.span.end}")
                    println(chunkspan.piece_text(first, document))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: chunk-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `chunkfault` | The configurations that have no answer, and the codes for them. |
| `chunkspan` | A span, a chunk, and the invariants a list of chunks holds. |
| `chunksize` | The three units, the caller's token counter, and the size-and-overlap limit. |
| `chunktext` | The fixed, recursive and token splitters, and the default separator lists. |
| `chunkmd` | The Markdown splitter, over markdown-nv's parse. |

## How to choose an entry point

**`chunktext.split_by_tokens` is the ordinary call.** A size, an
overlap and a counter, with the prose separators.

**`chunktext.split_prose` takes a `ChunkLimit` you built.** Use it when
several documents share one configuration.

**`chunktext.split_recursive` takes your own separator list.** The
order of that list is the whole configuration: prose wants
paragraph-then-sentence, a transcript wants speaker-then-line.

**`chunktext.split_fixed` respects nothing but the measure.** Use it
for a log, a transcript with no punctuation, or a language this package
has no separators for.

**`chunkmd.split` is for Markdown.** It never splits inside a code
block or a table, and it puts the enclosing headings in front of each
chunk.

## The rules a user needs

1. **The tokenizer is yours.** It is a `fn(Str) -> Int` that counts,
   and nothing else. Load your vocabulary once in your own code and
   close over it; this package performs no input and cannot load one.

2. **The size and the overlap share a unit.** `ChunkLimit` carries the
   unit once, so "500 with 50 of overlap" means the same thing in both
   numbers.

3. **The overlap must be smaller than the size.** The step is the
   difference, and a step of zero emits the same chunk for ever. An
   overlap equal to or larger than the size is refused, not clamped.

4. **No chunk exceeds the limit unless one indivisible unit does.** A
   word longer than the chunk size is emitted whole and over-size.
   `chunktext.oversize_pieces` finds them, so you know before a model
   refuses one.

5. **With an overlap of zero, the chunks concatenate back to the
   source.** Every byte is in exactly one chunk, separators included.
   `chunkspan.rejoin` and `chunkspan.covers_source` check it.

6. **A chunk is a span, not a string.** It points into the source you
   still hold. `chunkspan.text_of` is the one place a copy happens.

7. **A byte limit is an upper bound that can be undershot.** Chunks
   never cut a UTF-8 sequence in half, so a five-byte limit over
   three-byte characters produces three-byte chunks.

8. **A Markdown chunk's text is assembled, not sliced.** When the rule
   prepends headings, the text is the heading trail plus the span's
   own text, and the span alone is still the citation.

9. **A separator that is the empty string does not terminate.** The
   recursion is bounded and answers `ChunkRecursionTooDeep` rather than
   running for ever.

## What is not included

- **A tokenizer.** Counting tokens needs a model's vocabulary.
  tokenizers-nv is the package that has one, and a caller passes its
  counting function in. This package does not depend on it, so a
  program that chunks by characters carries no vocabulary at all.
- **Semantic chunking.** Splitting where the meaning changes needs
  embeddings of candidate splits, which is embeddings-nv's work and a
  package on top of both.
- **Splitting code by its syntax tree.** That needs a parser per
  language. `chunktext.code_separators` is a crude fallback and says so.
- **HTML-aware splitting.** html-nv parses HTML; a splitter over it is
  the same shape as `chunkmd` and is a later release.
- **Sentence segmentation by a trained model.** The separator lists
  hold `". "` and `"。"`, which is a heuristic and wrong on
  abbreviations.
- **Embedding, storing or searching.** embeddings-nv embeds,
  vectorstore-nv stores, hnsw-nv searches.

## Related packages

**markdown-nv** parses the Markdown. Its pull parser's events carry
byte ranges into the caller's source, which is the same shape a chunk's
span is, so nothing is copied between the two.

**tokenizers-nv** produces tokens. Its counting function is what
`chunksize.by_tokens` takes.

**embeddings-nv** turns a chunk into a vector. **vectorstore-nv** and
**hnsw-nv** store and search those vectors.

## Tests

There is no specification for chunking, so the suite asserts this
package's own rules, which are the two stated above.

- Eleven characters at a limit of five with no overlap is three
  chunks, and they rejoin to the source exactly.
- Four three-byte characters at a five-byte limit is four chunks of
  three bytes each: the limit is undershot rather than a character cut
  in half.
- A word longer than the limit appears in `oversize_pieces`.
- An overlap equal to the size is refused with `overlap_not_smaller`,
  and an empty separator list with `no_separators`.
- A fenced code block containing a blank line is one atomic span, and
  an offset inside it is atomic — which is the case a separator list
  with `"\n\n"` at the front gets wrong.
- The heading trail at an offset is the enclosing headings, outermost
  first.

Today every one of those assertions reaches a `not implemented` panic.

## Implementation status

| Area | Status |
| --- | --- |
| Spans, pieces and the coverage invariants | declared, not implemented |
| Units, the caller's counter, and limits | declared, not implemented |
| Fixed, recursive and token splitting | declared, not implemented |
| Markdown-aware splitting | declared, not implemented |
| Semantic chunking | not declared |
| Code splitting by syntax | not declared |
| HTML-aware splitting | not declared — a later release |

## Licence

Apache-2.0. See [LICENSE](LICENSE).
