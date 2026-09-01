# Biomedical Reranker Benchmark

Does a hosted reranker (Cohere) beat a domain-specific local stack (S-PubMedBert +
MedCPT) on biomedical retrieval? A controlled comparison across two datasets, with
retrieval held constant so only the reranking stage varies.

## Why this exists

[MediLink](https://github.com/MediLinkorg/Medilink_rag) is a biomedical RAG system I
helped build, using a local embedder and cross-encoder reranker in production. This
repo asks a specific question that system doesn't answer on its own: how does that
local stack compare against Cohere's hosted embedding and reranking models, on the
same candidates, scored the same way?

## Structure

```
notebooks/
  01_medilink_rag_pipeline.ipynb     Reference. The production RAG pipeline this
                                      benchmark is measured against. Adapted from
                                      team research at MediLinkorg — see Attribution.
                                      Not runnable standalone (needs Weaviate, the
                                      PubMed corpus, and API keys).

  02_retrieval_stack_benchmark.ipynb The comparison. Runs standalone. 2 embedders
                                      (S-PubMedBert, Cohere embed-v4) x 5 rerankers
                                      (none, MedCPT, Cohere rerank v3.5/v4-fast/v4-pro)
                                      on BEIR NFCorpus, scored with nDCG@10, Recall@K,
                                      MRR. A second pass runs the reranker axis on the
                                      real MediLink corpus (LUMEN).

  03_build_lumen_qrels.ipynb         Builds relevance judgments for the LUMEN corpus,
                                      which ships with no labels. LLM-judged (Claude
                                      Haiku) using standard TREC pooling. Run this
                                      before notebook 02's LUMEN section.

lumen_qrels/                         Output of notebook 03: labels, query set, and
                                      pool metadata for the LUMEN corpus.

results/                             Metric tables and charts from notebook 02.
```

## Reproducing this

Notebook 03 first, then notebook 02.

```bash
pip install -r requirements.txt
```

You'll need a Cohere API key (`COHERE_API_KEY`) and an Anthropic API key
(`ANTHROPIC_API_KEY`, used only in notebook 03 for judging). Both notebooks prompt for
these interactively if not already set as environment variables.

**Cost**, at published list prices, on a Cohere production key:
- Notebook 03 (judging ~900 pooled passages with Haiku): under $1
- Notebook 02, full NFCorpus split (323 queries, 2 embedders, 3 rerank tiers): ~$3.64

A free Cohere trial key works too — set `N_QUERIES = 100` and `RERANK_SLEEP = 6.5` in
notebook 02's config cell to fit inside the trial's 1,000 calls/month and 10 req/min
rerank limit.

## Method

**Candidate generation is dense-only** for both embedders — no BM25, no hybrid
retrieval. This isolates the reranking comparison from retrieval-fusion choices, at the
cost of not matching MediLink's production pipeline (which blends BM25 and dense at
`alpha = 0.60`, varied per query intent). Numbers here are not MediLink's production
numbers.

**NFCorpus** (BEIR) ships human relevance judgments, so results there are the grounded
half of this comparison: nDCG@10, Recall@{1,3,5,10}, and MRR@10, computed the standard
way.

**LUMEN** is the actual 77,649-passage PubMed corpus behind MediLink, published at
[`mohamedkhaledmk7/LUMEN-rag-data`](https://huggingface.co/datasets/mohamedkhaledmk7/LUMEN-rag-data).
It has no human judgments. Labels in `lumen_qrels/` are LLM-generated via pooled
assessment (see notebook 03) — TREC-style pooling with Claude Haiku as the judge, graded
0–3, judged blind (no arm identity, shuffled order), with a self-consistency check
against re-judged samples. These are weaker evidence than NFCorpus's human qrels and are
reported separately, never averaged together.

**A caveat inherent to pooling, not a bug**: any passage no arm ranked in its top-10
during pool construction is never judged and counts as non-relevant. This is how TREC
qrels have always been built, but it means recall on LUMEN is measured only within what
these five arms found, and any future system scored against these qrels is mildly
disadvantaged if it retrieves something all five missed.

**Both rerankers are score-normalized where relevant.** MedCPT's raw logits are
sigmoid-mapped to [0,1] to match Cohere's relevance-score range — this doesn't change
ranking (sigmoid is monotonic) but matters if a score-based cutoff is ever applied.

## Results

_To be filled in after notebook 02 runs. Numbers, charts, and the duo comparison
(S-PubMedBert+MedCPT vs. embed-v4+rerank-v4-pro) go here._

## Attribution

`notebooks/01_medilink_rag_pipeline.ipynb` is adapted from team research at
[`MediLinkorg/Medilink_rag`](https://github.com/MediLinkorg/Medilink_rag), part of the
MediLink healthcare platform. It's included here as reference context for what this
benchmark is measured against — it is not runnable standalone and requires a Weaviate
instance, the PubMed corpus, and API keys to execute.

## License

Code in this repo: MIT. The LUMEN corpus and its labels carry separate terms — see the
[dataset card](https://huggingface.co/datasets/mohamedkhaledmk7/LUMEN-rag-data).