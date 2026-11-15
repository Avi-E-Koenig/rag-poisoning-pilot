# Verification Plan — "Can Validated Data Stop RAG Poisoning?"

Goal: decide how far the claims in `report.md` can be trusted, by (a) auditing code and data
statically, (b) reproducing the run locally, and (c) testing whether the conclusions survive
fixes and variations of the method.

Status: **draft**, branch `verify/plan`, 2026-11-15.

---

## 0. Findings from the first pass (static audit, nothing executed yet)

| # | Severity | Finding | Evidence |
|---|---|---|---|
| F1 | **Critical** | Arm 7 ("content verification") rejects poison **by its ID**, not its content: `if "poison" in d["id"]: return False`. The 0/20 headline result for arm 7 is therefore true by construction. Whether real content checking would catch the poison is never tested. | `run_experiment.py` (`verified()`) |
| F2 | High | Arm 2 (the allowlist) is also true by construction: the poison is simply never added to the corpus. That is a fair description of an allowlist, but "0/20" here does not measure anything about the attack. | `arms["2_allowlist"]` |
| F3 | High | The "compromised source" is only relabeled poison sitting next to the real Wikipedia chunks. The scenario in which content verification would actually be tested, where the canonical source itself is edited, is not modeled. If the attacker controls the source, re-fetching it returns the poison. | Report §6 admits the relabeling |
| F4 | Medium | The corpus covers **45** articles, not the 50 claimed: the `ARTICLES` list has 45 entries and `corpus.json` contains 45 distinct titles. | `build_corpus.py`, `corpus.json` |
| F5 | Medium | Arm 7 kept 596/859 docs. That means **163 trusted chunks (21%) were dropped**, which is more than the single Great Barrier Reef failure the report mentions. The drops could come from failed re-fetches or from Wikipedia text changing since ingestion. The report's claim that this causes "no measurable accuracy cost" rests on only 10 control questions. | `results.json` |
| F6 | Medium | Poison quality is uneven. 19/100 poison docs are under 20 words (one-line Q&A). `poison::1971::3` actually states the **correct** answer (1969). 5/100 never mention the attacker's answer. | `poisons.json` |
| F7 | Medium | Scoring is a raw substring match. The report acknowledges the β-lactam/beta-lactam case. Correctness on the target questions in the defended arms is never reported: the allowlist arm answers polio → "Maurice Hilleman", and only 18/20 target answers are correct. | `results.json` |
| F8 | Medium | n=20 targets, one poison generation (temperature 0.7, unseeded), one generator model (0.5B) and no confidence intervals. The differences of 16, 17 and 18 out of 20 between arms 3–6 are noise-level. | `gen_poison.py` |
| F9 | Low | The hybrid RRF uses `1/(rank+1)` (k=0) instead of the usual k=60. `texts.index()` maps duplicate texts to the first occurrence. Either could bias the "hybrid doesn't help" result. | `HybridRetriever` |
| F10 | Low | The corroboration gate marks 806/817 docs as uncorroborated, so it is effectively a no-op rather than evidence that "corroboration fails". τ=0.75 was never tuned. | `results.json` |
| F11 | Low | The near-duplicate filter keeps the first doc of every cluster by design, so one poison per cluster always survives. That is consistent with the report's "one survivor suffices" result, but it is partly an artifact of the design. | `screen()` |
| F12 | Low | Housekeeping: the docstring says 5 arms (the code has 7). `trusted_corpus` in the screening info is actually the full corpus size of 859. The report says the code lives in a `rag-poisoning/` directory that does not exist. | — |
| F13 | ? | Two cited sources need existence checks: arXiv 2603.18034 and arXiv 2607.10712 (the "Distributed Denial of Science" study). The INSS memo URL also needs checking. | report §8 |

### 0b. Provenance findings (from the Perplexity transcript)

Source: [`verification/context/perplexity-thread.md`](verification/context/perplexity-thread.md). This is the shared Perplexity "Computer" session that designed the experiment, ran it, wrote the report and pushed this repo.

| # | Severity | Finding | Evidence |
|---|---|---|---|
| P1 | **High** | **AI-generated end to end, with no human review of the method.** The research framing, the code, the run, the report and the git push were all done by Perplexity's agent in one session (about 34 min of runtime). The user's only inputs were "ok, lets see what you've got", then questions about where the files are and a request to push to GitHub. | Transcript; upstream commits carry `Co-authored-by: Perplexity <noreply@perplexity.ai>` |
| P2 | **High** | **Arms 6 and 7 were added after the results came in (post-hoc).** The plan written before the run had 5 arms and *expected* arms 4/5 to "partially recover". They didn't (17/20). The agent then added arm 6 ("Before writing up, I'm adding a sixth arm…"). When arm 6 also failed, it added arm 7 ("one more cheap, realistic defense worth test[ing]"). The report presents all 7 as the design from the start, and its hypotheses were reworded to fit. This is a textbook garden-of-forking-paths problem. | Transcript, the "One-Day Testable Framing" turn vs the run narrative |
| P3 | High | **`results.json` mixes outputs from different code versions.** Arms 1–3 ran first, then the script crashed (`'list' object has no attribute 'sum'`). The code was edited and the script resumed with `already done, skipping`. It was edited again for arms 6 and 7, and the earlier arms were never rerun. The shipped `run_experiment.py` is therefore *not* the exact code that produced arms 1–6. | Transcript tool output; the resume logic in `main()` |
| P4 | High | **Arm 7 first kept 0/859 docs.** The verification was then "fixed" (retries and pacing). Whether the `"poison" in id` shortcut (F1) was added in that fix or existed from the start is not visible, but either way the shortcut is what produces the final result. | Transcript: `content verification: kept 0/859 docs` |
| P5 | Medium | The agent noticed that a poison doc stated the correct answer ("Note v3 of the Moon landing question says 'July…'") but neither removed it nor disclosed it. | Transcript; F6 |
| P6 | Medium | The "50 articles" figure was reported without being checked. The script prints per-article chunk counts and a total, never an article count. The corpus actually has 45 titles. "Light bulb" was redirected to "Electric light". There is no "Johannes Gutenberg" article, even though one target question is about him. | `corpus.json` titles |
| P7 | Medium | Every citation and claim in the report traces back to an earlier Perplexity research conversation (an attached `.md` file we could not open). Those include the "Distributed Denial of Science" figures (≈50% success, 6% detection, provenance audit → 0%) and "Before the Ballot" (Oct 2026). They need independent checking, because LLM research summaries can fabricate or distort sources. | Transcript, Part 3 summary |
| P8 | Low | The chart was made by inline code that isn't in the repo. The original environment: Python 3.14.8, 2 CPU cores, no GPU, llama.cpp `b12389` CPU build, `Qwen/Qwen2.5-0.5B-Instruct-GGUF` q4_k_m (491,400,032 bytes), `--threads 2 -c 4096`, run on 2026-11-15 around 03:44 UTC. | Transcript tool output |
| P9 | Info | The planning turn promised a "publishable-grade result" and predicted the outcome in advance ("Arms 2 and 5 should show ASR near zero"). The README now presents the work as validated research. | Transcript |

---

## 1. Environment setup

Target machine: WSL2, Python 3.12, RTX 4050 Laptop GPU (6 GB).

1. Create `.venv` and install pinned `fastembed rank_bm25 requests matplotlib numpy` into `requirements.lock`.
2. Generator: run `llama-server` (llama.cpp, CUDA build or the `llama-cpp-python[server]` package) with `Qwen2.5-0.5B-Instruct-Q4_K_M.gguf` from the official Qwen HF repo. Record the model's sha256.
3. Pin and record the llama.cpp version, seed, and CPU vs GPU. Temperature 0 is only *mostly* deterministic across backends.
4. Isolation: run the scripts with `python -I` from the venv. Network egress goes only to `en.wikipedia.org` and `localhost:8080` (already confirmed by reading the code).

## 2. Reproduction tiers

| Tier | What | Pass criterion |
|---|---|---|
| R1 | Re-score the shipped `results.json` (no model) | The report's tables match the raw JSON. *Already matches, except F4 and F5.* |
| R2 | Re-run `run_experiment.py` on the **shipped** `corpus.json` + `poisons.json` | Per-arm ASR is within ±1 of the report, and per-question answers agree on most questions |
| R3 | Full rebuild: `build_corpus.py` with a fresh Wikipedia fetch, then `gen_poison.py` ×5 seeds, then run the experiment | Same qualitative ordering of the arms. Report the variance across seeds. |

## 3. Methodology stress tests (do the conclusions survive?)

1. **Honest arm 7**: remove the ID shortcut and verify by content only (substring match / hash against the canonical text). Expected result: still 0/20 when the source is clean. This turns a tautology into a measurement.
2. **Real compromised source (arm 8)**: inject the poison *into* the canonical text, i.e. a simulated malicious Wikipedia edit, and verify against that compromised canonical copy. Hypothesis: verification fails. If so, report §7 conclusion 3 is overstated.
3. **Snapshot verification (arm 9)**: verify against a signed or hashed ingestion-time snapshot instead of a live re-fetch. This fixes the F5 drift and isolates what verification actually protects against.
4. **Scoring**: normalize answers (β→beta, case, aliases). Add the target-question correct-rate metric. Add a blind manual review of all ~300 generated answers.
5. **Statistics**: Wilson 95% CIs per arm, McNemar's test between paired arms, 5 poison seeds × 3 generator seeds.
6. **Generator sensitivity**: Qwen2.5-0.5B vs Qwen2.5-7B / Llama-3.1-8B Q4 (these fit in 6 GB). A 0.5B model is unusually easy to hijack.
7. **Poison quality**: re-run with degenerate and self-defeating poisons filtered (F6), and with poison budgets of 1 and 3 per question.
8. **Defense parameters**: sweep τ_dup (0.85–0.95), σ for stuffing (2–5), and the corroboration τ (F10). Use the standard RRF with k=60 (F9).
9. **Citations**: check that every source in §8 resolves and actually says what it is cited for. Also check the claims carried over from the earlier conversation (P7): the "Distributed Denial of Science" figures, "Before the Ballot", OWASP AISVS, the ACM survey's "43 countermeasures", and PoisonedRAG's "≈90% with 5 texts".
10. **Pre-registered rerun (P2)**: decide the arms, the metrics and the stopping rule *before* running. Report every arm that is run.
11. **Single-code-version rerun (P3)**: delete `results.json` and run every arm with one commit. Record the git SHA in the results.
12. **Match the original environment (P8)**: use llama.cpp `b12389` on CPU with `--threads 2 -c 4096`, to separate environment effects from methodology effects.

## 4. References

Primary artifacts:
- Upstream repo: https://github.com/Blobi92/rag-poisoning-pilot (commits `a7e1074`, `8595a09`, `21d541f`)
- Generating session transcript: [`verification/context/perplexity-thread.md`](verification/context/perplexity-thread.md)
- Model: https://huggingface.co/Qwen/Qwen2.5-0.5B-Instruct-GGUF (`qwen2.5-0.5b-instruct-q4_k_m.gguf`)
- llama.cpp build used: https://github.com/ggml-org/llama.cpp/releases/tag/b12389
- Embedder: https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2 (via `fastembed`)
- Wikipedia extracts API: https://www.mediawiki.org/wiki/Extension:TextExtracts

Literature cited by the report (to be checked in Phase A):
- PoisonedRAG: https://arxiv.org/abs/2402.07167
- Corpus-Dependent Poisoning Attacks and Defenses in RAG Systems: https://arxiv.org/html/2603.18034v1
- "Distributed Denial of Science" provenance audit: https://arxiv.org/html/2607.10712
- OWASP RAG Security Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/RAG_Security_Cheat_Sheet.html
- INSS Data Poisoning Primer (Memo 256): https://www.inss.org.il/wp-content/uploads/2026/07/Memo-256_digital.pdf
- Threats to Training (ACM CSUR): https://dl.acm.org/doi/full/10.1145/3538707
- Cited in the earlier conversation only: "Machine Learning Security against Data Poisoning: Are We There Yet?", "Before the Ballot" (Oct 2026), OWASP AISVS

Methodology references for the stress tests:
- RRF (k=60): Cormack, Clarke & Büttcher, SIGIR 2009, "Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods"
- Wilson score interval and McNemar's test for paired proportions (standard)
- Garden of forking paths: Gelman & Loken (2013)

## 5. Deliverables

- `verification/` folder holding scripts, lockfile, model hash, and the raw results from each tier
- `VERIFICATION_REPORT.md` giving a per-claim verdict (Confirmed / Partially / Not supported / By construction) with evidence
- Optional upstream issue/PR to Blobi92 covering F1, F4 and F12

## 6. Proposed order

Phase A: R1 (done) and citation checks (cheap). Phase B: environment setup and R2. Phase C: stress tests 1, 2, 4 and 5 (the core credibility questions). Phase D: R3 and tests 3, 6, 7 and 8 (robustness).
