# folio-ask

Ask questions of your own documents, answered on your own computer. folio-ask reads a folder of PDFs, Markdown,
text, web pages and Word documents, finds the passages that answer a question, and writes the answer with the
passages it came from, numbered. No API, no key, no site: three small open models run here through the ONNX
Runtime that comes with Windows 11.

It's written in [Folio](https://github.com/samyyy2423/folio), with the
[rag](https://github.com/samyyy2423/folio-rag), [vectors](https://github.com/samyyy2423/folio-vectors),
[tokenizer](https://github.com/samyyy2423/folio-tokenizer) and [pdftext](https://github.com/samyyy2423/folio-pdftext)
packages.

```text
$ folio run --allow-read --allow-libraries ask.folio ask "How many vacation days do employees get?"
25 days [1]

Sources:
  [1] policy.docx
  [2] learn.html
  [3] VERSIONING.md
```

## Start

```text
folio get
folio run --allow-read --allow-write --allow-run --allow-libraries ask.folio models
folio run --allow-read --allow-write --allow-libraries ask.folio index docs
folio run --allow-read --allow-libraries ask.folio ask "How do I install it?"
```

`folio get` fetches the packages that `folio.packages` lists, checked against `folio.sums`. `models` fetches the
three models into `models/` with curl (about 700 MB). `index` reads every file it can under `docs` into
`docs.rag`. Then:

| | |
|:--|:--|
| `ask.folio ask "a question"` | one answer |
| `ask.folio chat` | a question a line, until an empty one |
| `ask.folio serve 8080` | a page at http://127.0.0.1:8080 with a box for the question, the answer, and each source's passage (add `--allow-net`) |
| `--kb other.rag` | another knowledge base than `docs.rag` |
| `--quotes` | the answer in the documents' own sentences, without the writing model |
| `--words` | (with `index`) passages found by their words alone, without the embedding model |
| `--model`, `--reranker`, `--writer` | another folder for a model |

## How it answers

1. **Reading.** A PDF is read a page at a time (each passage knows its page), a web page without its tags,
   scripts and styles, and a Word document from the paragraphs in its `word/document.xml`. The text is cut into
   passages of about 200 tokens, whole paragraphs where they fit, each starting with the end of the one before.
2. **Finding.** Each passage is a vector from
   [all-MiniLM-L6-v2](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2) and its words are indexed for
   BM25. A question finds the 20 closest passages both ways, and the two rankings are fused (reciprocal rank
   fusion).
3. **Reranking.** A cross-encoder, [ms-marco-MiniLM-L-6-v2](https://huggingface.co/cross-encoder/ms-marco-MiniLM-L-6-v2),
   reads the question with each of the 20 and scores how well it answers it. They're put in that order.
4. **Writing.** [Qwen2.5-0.5B-Instruct](https://huggingface.co/Qwen/Qwen2.5-0.5B-Instruct) reads the best 3,
   numbered, and the question, and writes a few sentences from them alone. A model this small often leaves out
   its citations, so each sentence without one gets the number of the passage with at least 2 in 5 of its words
   (`citations.folio`). That says where a sentence's words came from; it doesn't prove the passage says what the
   sentence does, so the sources are shown with every answer.

Without the writing model, or with `--quotes`, the answer is the 3 sentences from the best 5 passages that are
closest to the question in meaning and share the most of its words, as they're written.

## The models

| | Size | From | License |
|:--|--:|:--|:--|
| all-MiniLM-L6-v2 | 90 MB | `sentence-transformers/all-MiniLM-L6-v2`, `onnx/model.onnx` | Apache 2.0 |
| ms-marco-MiniLM-L-6-v2 | 91 MB | `Xenova/ms-marco-MiniLM-L-6-v2`, `onnx/model.onnx` | Apache 2.0 |
| Qwen2.5-0.5B-Instruct | 512 MB | `onnx-community/Qwen2.5-0.5B-Instruct`, `onnx/model_int8.onnx` | Apache 2.0 |

Each with its `tokenizer.json`, and Qwen with its `config.json`. They run through `onnx.folio`, Folio's binding
to ONNX Runtime's C API. Windows 11 has ONNX Runtime 1.17 in System32; elsewhere, put `onnxruntime.dll` (or
Linux's `libonnxruntime.so`) from [its releases](https://github.com/microsoft/onnxruntime/releases) beside the
program.

Qwen's ONNX file says it's in ONNX's format 10, which Windows' ONNX Runtime 1.17 doesn't read (it reads up to 9).
It uses nothing format 10 added, only ONNX's own operators of opset 14, so `models` changes that one byte to 9.
And with an empty cache, that runtime lets each of several tokens read at once see the ones after it, so
`writer.folio` reads a prompt's first token alone, then the rest 64 at a time.

## How well it finds the answer

`evaluate.folio` asks a knowledge base questions, each with words the passage that answers it has in it, and counts how often such a passage comes first, in the first 3, 5 and 20, and the mean of 1 over its
place in the first 10 (MRR):

```text
folio run --allow-read --allow-libraries evaluate.folio eval/folio-docs.json --kb folio.rag --misses
```

`eval/folio-docs.json` has 40 questions about Folio's README, SPEC, DESIGN, CHANGELOG and VERSIONING (132,000
words, 1,395 passages), written for this in other words than the documents use ("Can a parameter have a default
value?", "How long can a sound last?"). Each question's first evidence was written with it. Then the passage each
way below put first was read, and where it answers the question in other words, its words were added too (pooled
judgments).

| | first | first 3 | first 5 | first 20 | MRR@10 |
|:--|--:|--:|--:|--:|--:|
| BM25 (words) | 27.5% | 37.5% | 42.5% | 75.0% | 0.371 |
| MiniLM (meaning) | 52.5% | 70.0% | 75.0% | 87.5% | 0.631 |
| BM25 and MiniLM, fused | 45.0% | 67.5% | 72.5% | 90.0% | 0.564 |
| BM25, then reranked | 65.0% | 72.5% | 72.5% | 75.0% | 0.683 |
| **fused, then reranked** (what `ask` does) | 60.0% | 72.5% | 75.0% | 90.0% | 0.663 |

What it says: with questions in other words than the documents', BM25 alone finds the answer first a quarter of
the time, and meaning finds it twice as often. Fusing the two finds the most passages in the first 20, and the
reranker then puts the right one first 60% of the time. 4 questions in 40 have no answer in the fused first 20,
and those are where to look next (`--misses` lists them). The questions are 40, about one set of documents, so a
few points either way mean little. The documents aren't in this repository; make a
questions file for your own, in the same form, and run it on them.

## Speed

On a laptop (Windows 11, x86-64, on its processor):

- reading 3 files into 69 passages: 3 seconds;
- an answer in the documents' own sentences (`--quotes`): about 7 seconds, most of it loading the models;
- reranking 20 passages: under a second;
- a written answer: 20 to 40 seconds. Loading Qwen takes about 6 seconds, reading the prompt (about 700 tokens)
  about 10, and each token written about 0.4.

Writing is slow because `onnx.folio` hands ONNX Runtime the whole cache of keys and values (24 layers, as numbers)
on every token, and takes it back. Keeping it in ONNX Runtime's own memory between tokens would make each token
several times faster; that's a change to `onnx.folio` itself. `serve` keeps the models loaded, so its answers
skip the loading.

## Training your own retriever and reranker

`train.folio` trains, in Folio with learn.folio, a retriever and a reranker on a knowledge base `index` made, and
measures how often each finds the passage a question came from, beside BM25 and the embedding model. The
questions are the documents' own sentences, as written and cut down to a few words, and a tenth of them are kept
back to test with.

```text
folio run --allow-read --allow-write --allow-libraries train.folio --kb docs.rag [--steps 600]
```

- The retriever: a BPE of 4,096 tokens learned from the passages, a vector of 128 numbers for each token, a
  text's vectors averaged, a layer, and the result made 1 long. It learns each question's passage against the
  other passages in its batch (InfoNCE), from questions varied as a person might type them.
- The reranker: a logistic regression over BM25's score, the embedding model's and the retriever's closeness, the
  share of the question's words in the passage, and each one's rank.

On the same Folio documents, 600 steps in 3.6 minutes: found first, and in the first 5, of 300 questions it
hadn't learned from:

| | sentences as written | a few of their words |
|:--|--:|--:|
| BM25 | 86.7%, 100% | 81.3%, 97.7% |
| all-MiniLM-L6-v2 | 50.3%, 77% | 28.3%, 53.3% |
| BM25 and MiniLM, fused | 69.3%, 96.7% | 54%, 89.7% |
| the retriever trained in Folio | 51.3%, 71.7% | 21.3%, 39.7% |
| the retriever and BM25, fused | 62.3%, 92.7% | 40.3%, 86.7% |
| all three, reranked | 80%, 99.3% | 76.7%, 97.3% |

These questions are the documents' own words, so BM25 is hard to beat here, which the questions a person writes
(above) show isn't so in use. A retriever of 528,000 numbers trained here in minutes finds the first passage as
often as MiniLM does from whole sentences, though less often from a few words. `ask` doesn't use them yet.

## Files

| | |
|:--|:--|
| `ask.folio` | the commands: `models`, `index`, `ask`, `chat`, `serve` |
| `documents.folio` | the text of web pages and Word documents |
| `rerank.folio` | the cross-encoder |
| `writer.folio` | Qwen, a token at a time, with its cache of keys and values |
| `citations.folio` | citations put back from the words of each sentence |
| `sentences.folio` | a passage's sentences, and a text's words, for quoting |
| `knowledge.folio` | a knowledge base opened with the model it was made with |
| `evaluate.folio`, `eval/` | how well the search finds the answer |
| `train.folio` | a retriever and a reranker trained in Folio |
| `ask.test.folio` | tests that need no model: `folio test --allow-read --allow-libraries ask.test.folio` |

## Limits

- Scanned PDFs have no text to read (no OCR), and a Word document's tables come out as their cells' text, a
  paragraph each.
- Qwen2.5-0.5B is small: it answers short questions from what it's given, but it can get a passage wrong. The
  sources are always shown, and `--quotes` gives the documents' own words.
- A knowledge base is loaded whole into memory, which is fine for thousands of passages, not millions.
- `serve` answers on 127.0.0.1 only, for the computer it runs on.
- The writer needs memory for its 512 MB model as well as the knowledge base; beside another large job it can
  run out.

## License

Apache 2.0. The models have their own licenses, above.
