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
9. **Citations**: check that every source in §8 resolves and actually says what it is cited for.

## 4. Deliverables

- `verification/` folder holding scripts, lockfile, model hash, and the raw results from each tier
- `VERIFICATION_REPORT.md` giving a per-claim verdict (Confirmed / Partially / Not supported / By construction) with evidence
- Optional upstream issue/PR to Blobi92 covering F1, F4 and F12

## 5. Proposed order

Phase A: R1 (done) and citation checks (cheap). Phase B: environment setup and R2. Phase C: stress tests 1, 2, 4 and 5 (the core credibility questions). Phase D: R3 and tests 3, 6, 7 and 8 (robustness).
