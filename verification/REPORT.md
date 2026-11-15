# Auditing an AI-Generated Research Result: Does "Content Verification" Really Stop RAG Poisoning?

*Apart Research — AI x Epistemics Sprint, November 2026 · Tracks: 2 (Trust Infrastructure), 4 (Open)*

**Authors:** _[names / affiliations]_

## Abstract

_(to finalize after results; 150–250 words)_

AI agents can now produce a complete research artifact, from question to code, data, report and public repository, in under an hour. We ask whether such a result survives independent verification. Our case study is a public RAG-poisoning pilot generated end to end by an autonomous research agent during this sprint. It claims that validated-source ingestion stops external poisoning (18/20 → 0/20) and that "content verification against the canonical source" is the only defense that stops a compromised trusted source (18/20 → 0/20). We combine (i) a static audit of code and data, (ii) a provenance audit of the agent's session transcript, (iii) a citation check, and (iv) a pre-registered reproduction and extension. Static audit: the headline 0/20 for content verification is **produced by the code itself**, since poison documents are rejected by their ID string, not by their content. Provenance: the two arms that "worked" were added after the planned arms failed, and the results file mixes outputs from several code versions. Citations: 3 of 6 references carry errors, including an arXiv ID that resolves to an unrelated radiotherapy paper. Our reproduction (original binary, identical model hash) _[R2 result]_. Under pre-registered tests across three poison sets, _[Phase D result: honest 7′, compromised canonical 8 vs 8-noverify]_. We release a one-command reproduction and a checklist for auditing agent-generated research.

## 1. Introduction

_[draft]_ Research agents lower the cost of producing research-shaped artifacts faster than they lower the cost of checking them. When an agent produces a clean table, a chart, a report and a GitHub repo, the usual surface signals of rigor (hypotheses, multiple arms, a limitations section, citations) are all present. Whether the claims hold is a separate question. We treat that gap as an epistemic infrastructure problem: what does it take to verify an AI-generated result, and what kinds of failure should a reviewer expect?

Contributions:
1. A four-layer audit protocol (code/data, session provenance, citations, pre-registered reproduction) applied to a real agent-generated study.
2. Evidence that the study's headline result is produced by its own code, and that the threat model needed to test it was never modeled.
3. A pre-registered extension showing what content verification does and does not protect against (honest verification; a compromised canonical source; snapshot manifests).
4. A one-command reproduction kit and a reviewer checklist.

## 2. Related work

- **RAG poisoning.** PoisonedRAG (Zou et al., arXiv:2402.07867; USENIX Security 2025) reports ~90% ASR with five injected texts per target question. Semantic Chameleon (Thornton, arXiv:2603.18034) shows that hybrid BM25+vector retrieval mitigates *gradient-guided* poisoning, and reports 0% attack success on a Wikipedia corpus.
- **Provenance auditing.** "Distributed Denial of Science" (Gyevnár et al., arXiv:2607.10712) shows that a five-check provenance audit prompt cuts indirect dataset-poisoning success for research agents from 49.56% to 0%.
- **Guidance.** The OWASP RAG Security Cheat Sheet and INSS Memo 256 (Hershkovitz, April 2026) recommend defense in depth.
- **Meta-science.** Forking paths (Gelman & Loken, 2013) and pre-registration.

## 3. Methodology

### 3.1 Subject of the audit
The upstream repository is `Blobi92/rag-poisoning-pilot` (commits `8595a09`, `21d541f`; co-authored by Perplexity). Its setup: 759 Wikipedia chunks, 100 LLM-generated poison documents (5 per target question), 20 target questions and 10 clean controls. The generator is Qwen2.5-0.5B-Instruct (llama.cpp) and the embedder is MiniLM-L6-v2. It has 7 defense arms.

### 3.2 Layer 1 — static audit of code and data
We read the code and checked every reported number against the raw `results.json`, `corpus.json` and `poisons.json`.

### 3.3 Layer 2 — provenance audit
We recovered the agent's session transcript and the earlier research conversation that framed the question. We compared the plan written before the run with the reported design.

### 3.4 Layer 3 — citation verification
We resolved every reference. Metadata came from arXiv and Crossref, and full text from PDFs. Each cited figure was checked verbatim.

### 3.5 Layer 4 — reproduction and pre-registered extension
- **R2 anchor run:** the unmodified upstream script, with the identical llama.cpp build (`b12389-1ae047c92`), the identical model file (sha256 `74a4da8c…`) and the original thread settings.
- **Pre-registration** (commit `81bfacb`, made before any new result): arms 7′ (content verification without the ID shortcut; poison claims the target article's URL), 8 (poison spliced into the canonical article, then verified against that same poisoned canonical), 8-noverify (control) and 9 (ingestion-time SHA-256 manifest). Alias-based normalized scoring. Three poison sets (the shipped set S0, plus seeds 1 and 2). Wilson 95% CIs and exact McNemar tests.

## 4. Results

_(filled from `RESULTS.md`)_

## 5. Discussion

_(after results)_

## 6. Conclusion

_(after results)_

## References

_(final list)_

## Appendix A — Limitations and Dual-Use / Ethical Considerations

_(required by the sprint)_

## Appendix B — Reviewer checklist for agent-generated research

_(draft)_
