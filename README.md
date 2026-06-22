# Delphi

**Grounded biomedical synthesis** — ask a question, get a cited, quote-verified answer over open-access literature, enriched with structured entity and target evidence.

[![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![PyTorch](https://img.shields.io/badge/PyTorch-EE4C2C?logo=pytorch&logoColor=white)](https://pytorch.org/)
[![Transformers](https://img.shields.io/badge/%F0%9F%A4%97%20Transformers-FFD21E)](https://huggingface.co/docs/transformers)
[![vLLM](https://img.shields.io/badge/vLLM-self--hosted-4B8BBE)](https://github.com/vllm-project/vllm)
[![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![License](https://img.shields.io/badge/use-research%20only-lightgrey)](#status)

> Research tool, not medical advice.

## What it does

Biomedical questions don't have answers — they have *literatures*. The honest answer to "what are the resistance mechanisms to drug X?" is scattered across dozens of papers that partly agree, partly conflict, and constantly update. General-purpose chat models will happily synthesize that for you, but they hallucinate citations, paraphrase claims into something the source never said, and can't tell you when the evidence simply isn't there.

**Delphi** answers biomedical questions strictly from the open-access literature and proves its work. Every substantive claim is tied to a verbatim quote from a real paper, and every quote is checked — character for character — against the exact source passage the model was shown. Fabricated or drifted quotes are flagged rather than trusted. When the retrieved evidence is thin or contradictory, Delphi says so and abstains instead of guessing.

The result is a synthesis you can audit: a readable answer, the sources behind each sentence, and a structured evidence panel that grounds the named entities in curated biomedical knowledge bases.

## How it works

Questions drive **live retrieval** — there is no fixed corpus. Each query searches the Europe PMC open-access full-text subset, pulls and caches the matching papers on the fly, and synthesizes an answer grounded in what it just retrieved.

```
question
   │
   ▼  query expansion + scope/open-access filter
search (Europe PMC) ──► fetch full text · chunk · entity-ground   (parallel, cached)
   │
   ▼  hybrid retrieval — BM25 lexical + MedCPT dense, fused with Reciprocal Rank Fusion
top-k source chunks
   │
   ▼  grounded synthesis (LLM): source-only answering, per-claim verbatim quotes
draft answer + citations
   │
   ▼  local quote verification — each quote must appear in its cited source chunk
   ▼  entity & target enrichment (normalized IDs, structured evidence)
cited, verified answer  +  evidence panel  +  entity graph
```

- **Retrieval is hybrid and candidate-scoped.** BM25 ranks every chunk cheaply; the MedCPT biomedical dense encoders then re-embed and rerank only the top candidates, and the two rankings are fused with Reciprocal Rank Fusion. Embeddings are cached so any chunk is encoded at most once — the difference between embedding tens of passages instead of hundreds on a CPU box.
- **Grounding is enforced, not requested.** The model returns each claim with a verbatim supporting quote; a local verifier fuzzy-matches every quote back to the specific source passage it claims to cite. This is an explicit anti-hallucination contract, independent of which LLM produced the text.
- **Entities are normalized and enriched.** Recognized biomedical concepts are linked through entity-normalization services and joined against curated knowledge bases, surfacing structured target/variant evidence alongside the prose answer.
- **The LLM backend is pluggable.** A single OpenAI-compatible interface drives either a self-hosted open-weights model (served with vLLM) or a hosted inference provider, switchable by configuration alone — no code changes.

## Highlights

- **Quote-level verification** — citations aren't decoration; unverifiable quotes are caught and marked, so the answer is auditable end to end.
- **Calibrated abstention** — explicitly declines on unanswerable or evidence-poor questions rather than fabricating, validated with a deliberate "unanswerable" control probe.
- **Domain-specific dense retrieval** with MedCPT, fused with classic lexical search for the best of recall and precision.
- **Knowledge-graph enrichment** — answers are paired with normalized entities and structured evidence from curated biomedical resources, plus an interactive entity graph in the UI.
- **Backend-agnostic inference** — runs the same pipeline against a self-hosted vLLM model on a SLURM HPC cluster (data-private) or a hosted API, behind one interface.
- **License-aware corpus** — only the open-access subset is touched, and each article's license is recorded for correct downstream attribution.
- **Ships as a single container** with the biomedical encoders baked into the image, so there's no multi-hundred-megabyte model download on cold start.

## Evaluation

Delphi includes a faithfulness-focused evaluation harness, inspired by the **MIRAGE / MedRAG** line of biomedical RAG benchmarks, that measures what a citation-grounded tool must get right:

- **verify rate** — are cited quotes actually present in the sources? (fully automatic)
- **citation coverage** — did the answer cite its claims at all?
- **groundedness** — does each quote genuinely support the claim it's attached to? (LLM-judge)
- **abstention** — does the system correctly decline on an unanswerable control question?

A standard multiple-choice biomedical benchmark can be plugged in for answer-accuracy scoring; the metrics above target grounding, which is Delphi's core promise and is fully automatable.

## Tech stack

| Layer | Technology |
| --- | --- |
| API & orchestration | Python 3.12, FastAPI, Uvicorn |
| Dense retrieval | PyTorch, Hugging Face Transformers, MedCPT encoders |
| Lexical retrieval | BM25, fused via Reciprocal Rank Fusion |
| LLM serving | vLLM (self-hosted, OpenAI-compatible) or hosted inference, behind one interface |
| Literature & knowledge | Europe PMC open-access full text; entity normalization + curated target/variant evidence |
| Storage | SQLite chunk + embedding cache |
| Frontend | Static HTML/CSS/JS with an interactive entity graph |
| Packaging & CI | Docker, GitHub Actions |

## Architecture notes

- **Same pipeline, two entry points.** A command-line interface and the FastAPI web app share one orchestration core, so the UI and scripted/eval runs always exercise identical logic.
- **Heavy model serving runs on a SLURM HPC cluster.** Large open-weights models are served with vLLM on cluster GPUs and reached over a private tunnel; the application stays decoupled via the OpenAI-compatible boundary and falls back to a hosted provider when the cluster isn't in use.
- **Cost-aware retrieval.** Lazy embedding, candidate-scoped reranking, parallel fetch/ground, and a persistent cache keep latency and compute bounded even on modest hardware.

## Status

Active research project. Delphi is built as a careful, auditable approach to biomedical literature synthesis — the emphasis is on *trustworthy* answers (real sources, verified quotes, honest abstention) rather than fluent-sounding ones. It is a research tool and explicitly not a source of medical advice.
