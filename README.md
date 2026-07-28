# Evidence-Based Psychiatry RAG Assistant

A production-grade Retrieval-Augmented Generation system that answers psychiatry and mental-health
questions **exclusively from retrieved, citable evidence** drawn from open clinical knowledge sources.

> **The language model is never allowed to answer from its own parametric memory.** If the retriever
> cannot supply sufficient supporting evidence, the system returns
> `"I could not find sufficient evidence in the indexed medical literature."`

Everything lives in one Colab notebook: `Evidence_Based_Psychiatry_RAG_Assistant.ipynb`.
No manual uploads, no dataset downloads, no API keys.

> ⚠️ **Not a medical device.** This is an information-retrieval engineering demonstration. It is not
> clinically validated and must not be used for diagnostic or treatment decisions. If you or someone
> else may be at risk of harm, contact your local emergency services or a crisis line immediately.

---

## Quick start

1. Open the notebook in Google Colab.
2. `Runtime → Change runtime type → GPU` (T4 or better; it also runs CPU-only in reduced-capability mode).
3. `Runtime → Run all`.
4. A Gradio app launches at the end with a public share link.

First run takes roughly 35–50 minutes (ingestion, embeddings, evaluation, model download). Every artefact
is cached to Google Drive, so later runs start in under a minute.

**Optional but recommended:** add an `HF_TOKEN` to Colab Secrets (key icon in the left sidebar).
All models used are public — the token only lifts Hugging Face's rate limit, which is the difference
between a 2-minute and a 55-minute model download on a cold runtime.

---

## Architecture

```
                    ┌──────────────────────── AUTOMATED INGESTION ────────────────────────┐
                    │  PubMed (E-utilities)   Europe PMC (OA full text)   ICD-11 (WHO)    │
                    │  MedlinePlus (NLM)      NIMH (NIH)      WHO mhGAP / reports         │
                    └───────────────────────────────┬─────────────────────────────────────┘
                                                    │
                            Cleaning → Normalisation → Deduplication → Metadata
                                                    │
                                   Semantic (section-aware) recursive chunking
                                                    │
                    ┌───────────────────────────────┴─────────────────────────────────────┐
                    │                        UNIFIED CORPUS (chunks)                      │
                    └───────────┬───────────────────────────────────┬─────────────────────┘
                                │                                   │
                  BAAI/bge-large-en-v1.5 dense vectors        BM25 sparse index
                                │                                   │
                          FAISS (IndexFlatIP)                rank_bm25 (Okapi)
                                └──────────────┬────────────────────┘
                                     Hybrid fusion (0.5 dense / 0.5 sparse)
                                               │
                              Cross-encoder rerank (bge-reranker-large)
                                               │
                          Confidence gate  →  grounded prompt  →  Qwen3-4B (4-bit)
                                               │
                    Citation enforcement + grounding verification + hedge detection
                                               │
                                     Gradio application (5 tabs)
```

---

## Data sources and licensing

Everything is downloaded automatically at runtime.

| Source | Access | Licence / reuse | Role |
|---|---|---|---|
| **PubMed** | NCBI E-utilities REST | Abstracts freely accessible for research | Primary — reviews, systematic reviews, meta-analyses |
| **Europe PMC** | REST `search` + `fullTextXML` | Open-access subset only, CC-BY / CC0 filtered at ingestion | Full-text sections |
| **MedlinePlus (NLM)** | `wsearch.nlm.nih.gov` web service | U.S. Government work — public domain | Topic summaries |
| **NIMH (NIH)** | Public topic pages | U.S. Government work — public domain | Disorder overviews |
| **WHO** (mhGAP, reports) | IRIS PDF + landing-page discovery | CC BY-NC-SA 3.0 IGO | Guideline full text |
| **ICD-11 Chapter 06** | WHO ICD API if credentials exist, else bundled scaffold | WHO terms; descriptors author-written | Classification backbone |
| **NICE / APA / VA-DoD** | Metadata + canonical URL only | Not openly relicensable | **Reference-only records** |

### Documented substitutions

1. **ICD-11.** The official API requires OAuth credentials, which conflicts with the no-API-keys
   constraint. The connector uses the real API when `ICD_CLIENT_ID` / `ICD_CLIENT_SECRET` are present,
   and otherwise falls back to a bundled Chapter 06 scaffold (codes, titles, hierarchy, author-written
   descriptors), with diagnostic depth supplied by public-domain NIMH and MedlinePlus content.
2. **NICE guidance** is indexed as metadata and URL only. The assistant can direct users to the
   authoritative guideline without reproducing protected text.
3. **WHO PDFs.** IRIS URLs rot. The connector tries a candidate list, then scrapes the landing page for
   the current download link, then degrades to a reference-only record. The corpus is never left empty
   because the NLM sources are independent.

---

## What the system does

| Capability | Implementation |
|---|---|
| Automated ingestion | 7 fault-tolerant connectors, retry with backoff, per-host rate limiting, full caching |
| Retrieval | FAISS dense (exact cosine), BM25 Okapi, weighted hybrid fusion |
| Reranking | `BAAI/bge-reranker-large` cross-encoder over a 50-candidate pool, depths 5 / 10 / 20 |
| Chunking | Section-aware, heading-driven, recursive with overlap; chunk-size study included |
| Generation | Transformers + Accelerate + bitsandbytes NF4, automatic model cascade |
| Hallucination control | Confidence gate, citation enforcement, sentence-level grounding, numeric containment, hedge detection |
| Evaluation | Recall@k, Precision@k, MRR, MAP, nDCG, Hit-Rate, latency, tokens — every headline number carries a 95% bootstrap interval, and hyper-parameter changes require the interval to exclude zero |
| RAG metrics | RAGAS when available, otherwise a calibrated native implementation of the same metrics |
| Reproducibility | One seed, content-addressed caches, version manifest, structured error log |
| Deployment | 5-tab Gradio app over the same pipeline object used for evaluation |

### Hallucination control, in order of execution

1. **Confidence gate** — blend of top rerank score, top-3 depth, an on-domain term (raw BGE cosine
   rescaled over a calibration band), and cross-document agreement. Below threshold the system refuses
   *before spending a generation token*.
2. **Citation enforcement** — `[n]` markers parsed and validated against the evidence block;
   out-of-range markers are hallucinated references.
3. **Sentence-level grounding** — each answer sentence is embedded and compared against its cited
   passages; unsupported sentences are listed explicitly.
4. **Numeric containment** — figures, doses and percentages in the answer must also appear in the
   evidence. A fabricated statistic is the most dangerous failure mode in clinical RAG.
5. **Hedge detection** — an answer that concedes "not explicitly mentioned in the evidence" is
   downgraded, because a sentence can faithfully restate an unrelated passage and score as
   well-grounded while answering nothing.

---

## Reference results

Colab T4 · Qwen3-4B (4-bit) · 5,762 chunks from 839 documents · 82-question held-out test split.
Fusion weights were tuned on validation; the test split was used once.

| Configuration | Recall@5 | Precision@5 | MRR | nDCG@10 | Latency (mean) |
|---|---|---|---|---|---|
| BM25 only | 0.197 | 0.105 | 0.288 | 0.184 | 18 ms |
| Dense only (FAISS) | 0.177 | 0.088 | 0.209 | 0.160 | 25 ms |
| **Hybrid (0.5 / 0.5)** | **0.221** | **0.117** | **0.328** | 0.212 | 61 ms |
| Hybrid + rerank (top-20) | 0.213 | 0.117 | 0.322 | **0.224** | 3426 ms |

With 95% bootstrap intervals:

| Configuration | MRR | nDCG@10 |
|---|---|---|
| BM25 only | 0.288 [0.215, 0.367] | 0.184 [0.140, 0.234] |
| Dense only | 0.209 [0.149, 0.275] | 0.160 [0.107, 0.219] |
| Hybrid | 0.328 [0.251, 0.409] | 0.212 [0.154, 0.273] |
| Hybrid + rerank | 0.322 [0.243, 0.407] | 0.224 [0.163, 0.287] |

RAG quality (native backend): faithfulness 0.887 · context precision 0.759 · context recall 0.687 ·
answer relevancy 0.842 · answer correctness 0.740. Grounded verdicts 83.3%, mean citation coverage 0.793.

### A negative result: cross-encoder reranking is not measurably better here

This is the finding the project is most confident about, because it is the one that was tested properly.

| Metric | Hybrid | + rerank | Delta | 95% CI (paired) | Resolvable at n=82 | Verdict |
|---|---|---|---|---|---|---|
| Recall@5 | 0.2207 | 0.2134 | −0.0073 | [−0.0569, +0.0441] | ±0.0505 | not distinguishable from noise |
| Precision@5 | 0.1171 | 0.1171 | −0.0000 | [−0.0268, +0.0268] | ±0.0268 | not distinguishable from noise |
| MRR | 0.3279 | 0.3166 | −0.0113 | [−0.0960, +0.0730] | ±0.0845 | not distinguishable from noise |
| nDCG@10 | 0.2119 | 0.2239 | +0.0120 | [−0.0325, +0.0565] | ±0.0445 | not distinguishable from noise |

Earlier development runs of this same system reported MRR gains from reranking of **+16%, +34% and +17%**
on three different test splits — all computed without intervals. A fourth split gave **−3%**. That spread
across splits *is* the noise the intervals measure. The point estimates were never evidence of anything.

**"Not significant" is not the same as "no effect."** Every observed delta is smaller than what an
82-question benchmark can resolve, so the honest conclusion is that this evaluation is **underpowered**
to confirm or refute a reranking benefit of the size a cross-encoder plausibly delivers. Resolving it
needs a larger, ideally expert-annotated, evaluation set — which is the top item on the roadmap.

**Engineering consequence.** Reranking costs ~3.4 s per query, roughly 60× the hybrid retrieval latency,
for no measurable benefit on this benchmark. The notebook leaves it enabled and configurable rather than
claiming it as a win, and the app exposes it as a toggle so the trade-off is visible.

**Abstention behaviour.** Out-of-domain and unanswerable questions are refused at the retrieval gate with
0 generation tokens:

| Query | Confidence | Outcome |
|---|---|---|
| "What are the diagnostic features of generalised anxiety disorder?" | 0.759 | answered, grounded |
| "What does the evidence say about lithium in bipolar maintenance?" | 0.831 | answered, grounded |
| "Which medications are first line for OCD?" | 0.832 | answered, grounded |
| "What is the best way to replace a car's timing belt?" | 0.375 | **refused** (4.0 s, 0 tokens) |
| "What will the 2035 global suicide rate be, precisely?" | 0.488 | **refused** (3.2 s, 0 tokens) |

> Figures come from one reference run. Re-running regenerates them, and values shift because PubMed and
> Europe PMC return different articles over time. Treat any difference whose interval spans zero as
> unproven — that rule is enforced in the notebook, not just recommended here.

## Evaluation methodology

There is no public psychiatry-RAG benchmark matching this corpus, so the notebook **synthesises one from
the corpus itself** using three pseudo-query generators (template, title-anchored, keyphrase), then splits
it 70/15/15.

**The split is grouped by document, not by question.** Questions derived from the same document share
near-identical evidence, so a random question-level split would leak the answer document across folds and
inflate every metric. Groups are allocated greedily by question count so the target ratios hold even
though groups carry very unequal numbers of questions. The notebook asserts zero document overlap between
splits and prints actual-vs-target ratios.

* **Train (70%)** — free exploration: prompt iteration, error analysis.
* **Validation (15%)** — hyper-parameter selection: fusion weights, rerank depth, thresholds.
* **Test (15%)** — touched once, for the reported numbers.

**Depth-matched comparisons.** A configuration returning k results cannot be scored at depths beyond k,
so those cells are masked rather than reported as low values, and the reranking comparison is made against
a baseline returning the same number of results. Comparing a 10-item reranked list against a 20-item
baseline penalises the reranker by construction — an easy way to manufacture a false negative.

**Honest caveats.** Questions are generated from the corpus, so they measure retrieval self-consistency
rather than agreement with clinical judgment — absolute values are optimistic and the *relative*
comparisons are the trustworthy signal. Relevance labels are heuristic, not expert-annotated. Several
template questions have dozens of labelled-relevant chunks, which mathematically caps Recall@5.

---

## Known limitations

* **The benchmark is underpowered for component-level A/B claims.** At ~80 test questions the smallest
  resolvable effect is roughly ±0.05 nDCG. Differences below that — which includes the entire reranking
  effect — cannot be confirmed or refuted. The system reports intervals rather than pretending otherwise.
* **Abstract-heavy corpus.** Most PubMed content is abstracts, which compress away the operational detail
  (doses, monitoring schedules, contraindications) clinicians most need.
* **The highest-value guidance is excluded by licence.** NICE, APA and VA/DoD guidelines are
  reference-only records.
* **Grounding is not correctness.** The verifier measures whether an answer restates its evidence, not
  whether that evidence is correct, current, or applicable. A worked example is in the notebook: a query
  about depression prevalence returned *anxiety* prevalence figures, faithfully cited. Hedge detection
  now flags that class of answer, but it is a mitigation, not a solution.
* **Retrieval quality varies by topic.** Queries about post-self-harm follow-up surface adjacent material
  (e.g. voluntary-assisted-dying literature) because the corpus lacks the guideline that would answer them.
* **Confidence is a heuristic**, not a calibrated probability. Thresholds are tuned, not validated
  against clinician judgement.
* Reranking costs ~3.3 s per query; generation dominates end-to-end latency on a T4.
* No multi-hop reasoning, query decomposition, or conversational memory.
* **RAGAS frequently fails to install on Colab.** The notebook falls back to a native implementation of
  the same metrics, calibrated over the same similarity band as the confidence gate, and always reports
  which backend produced the numbers.

---

## Roadmap

| Priority | Improvement | Rationale |
|---|---|---|
| High | Clinician-annotated benchmark (100–200 graded questions) | Removes the self-consistency bias in every current metric |
| High | Full-text ingestion via PMC OA bulk packages | Replaces abstracts with methods-and-results depth |
| High | Query decomposition + multi-hop retrieval | "Compare first-line treatments for OCD and PTSD" needs two evidence sets |
| Medium | Fine-tuned domain reranker on the benchmark's train split | Cross-encoders gain most from in-domain supervision |
| Medium | NLI-based faithfulness (DeBERTa-MNLI entailment per claim) | Stricter than embedding similarity |
| Medium | Calibrated confidence via isotonic regression | Turns the heuristic score into an actual probability |
| Medium | GRADE-aware recency and evidence-quality weighting | A 2024 meta-analysis should outrank a 2015 narrative review |
| Low | HNSW / IVF-PQ index + incremental ingestion | Needed beyond ~10⁵ chunks |
| Low | vLLM or TGI serving, structured JSON citations | Throughput and machine-readable output |

---

## Repository layout

```
.
├── Evidence_Based_Psychiatry_RAG_Assistant.ipynb   # the entire system
├── README.md
├── requirements.txt                                # for local/non-Colab runs
├── .gitignore
└── LICENSE
```

The notebook installs its own dependencies in Colab; `requirements.txt` is for running it locally.

### Caching and reproducibility

Every expensive stage goes through a `CacheManager.get_or_build` contract, and artefacts are
**content-addressed** so stale caches cannot survive a code change:

* `pipeline_version` — ingestion, cleaning and chunking artefacts
* `benchmark_version` — benchmark generation and split logic
* **text fingerprint** (chunk ids + text) — embeddings, FAISS, BM25
* **metadata fingerprint** (chunk ids + topic + section) — benchmark and all evaluation

Set `CONFIG.force_rebuild = True` to invalidate everything. Seeds are fixed for Python, NumPy, PyTorch,
the benchmark split, and the MinHash permutations.

---

## Licence

Code: see `LICENSE`. Ingested content remains under its original licences — see the data-sources table
above. The notebook records a per-document `license_note` and surfaces it in the Corpus Explorer tab.
Anyone redistributing a built corpus is responsible for honouring those terms.
