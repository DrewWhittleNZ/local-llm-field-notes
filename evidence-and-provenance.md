# Evidence and provenance

## Public-safe extracts for the local-LLM field report

**Prepared by:** Codex  
**Last updated:** 15 October 2026  
**Companions:** [technical report](from-tokens-per-second-to-decision-quality.md) and [plain-language report](from-faster-answers-to-better-decisions.md)  
**Scope:** Selected numerical extracts and provenance boundaries.

This appendix reproduces selected aggregate and numerical rows. It excludes holdings, cost bases, raw company excerpts, complete model prompts, tool credentials, host paths and full logs. A **sanitised extract** contains values from a fuller retained record; the omitted material is described where it affects interpretation. Source filenames, run IDs and commits identify provenance. The evidence available for inspection here is the reproduced data, not those identifiers alone.

Evidence levels below distinguish **extract** (numbers copied from a retained machine-readable record), **internal summary** (contemporaneous notes, with no sanitised call-level export here), and **pending** (not yet suitable for a public numerical claim). These are public-verifiability labels, not verdicts about whether the original observers were careful.

## E01 — M1 total-sequence slowdown: extract

The controlled sweep used Qwen3.5-4B, Ollama 0.32.15 and Q8_0 KV cache. It used nonce-controlled prompts to avoid treating a warm prefix as fresh prefill. The row below is one of four retained sweeps, run `f61f8a5ef385` (internal source `probe_runs/M1/context_sweep.jsonl`). Decode rate is generated tokens divided by Ollama's native generation duration, **not** wall time divided by completion tokens. Approximate sequence positions are given because each request generated its own answer.

| Approximate prompt tokens | Decode tokens/s, run `f61f8a5ef385` |
|---:|---:|
| 150 | 16.86 |
| 1,850 | 14.55 |
| 3,650 | 13.67 |
| 7,250 | 11.34 |
| 14,450 | 8.49 |
| 21,770 | 6.92 |

The other three sweeps show the same direction and similar endpoints (approximately 16.64–17.04 down to 6.88–7.03 tokens/s). This establishes the measured slowdown under that M1/runtime/configuration; it does not by itself identify the implementation mechanism. The separate 6,000-token long-generation record remains internal (`probe_runs/M1/long_gen.jsonl`); its window-by-window export is still needed before treating the equal-prompt/equal-output-cost claim as a standalone public result.

## E02 — M6 dense and MoE screen: extract

These are model-screen rows, **not** end-to-end ticker times or quality scores. The dense screen used 64K context, Q8_0 KV cache and flash attention on the 32 GB M6; all three models were reported 100% GPU-resident. Internal source: `model_screen/M6/m6-baseline/results.md`, captured 5 October 2026. The MoE screen used the same nominal 64K/Q8_0 setting; internal source: `model_screen/M6/m6-qwen36-moe-baseline/results.md`. The latter is a historical pre-refresh runner measurement, not a measurement of the later NVFP4/MLX production build.

| Screen / model | Reported size | Decode tokens/s | Prefill tokens/s |
|---|---:|---:|---:|
| Dense Qwen3.5-4B | 4.9 GB | 43.8 | 482.6 |
| Dense Qwen3.5-9B | 7.9 GB | 27.9 | 262.0 |
| Dense Qwen3.5-27B | 18.0 GB | 9.1 | 80.7 |
| Qwen3.6-35B-A3B MoE, earlier runner | 22.0 GB | 72.3 | 88.3 |

The screen supports the narrow point that full residency did not make dense 27B fast and that the tested MoE had a different speed/capacity profile. It does not establish report quality, all-context performance, or a general MoE advantage. The separate 80K production-allocation check and frozen financial panel need their own safe extracts if presented as independently verifiable headline results.

## E03 — Direct compaction gate: extract

The retained decision summary is run `1e01eec2cb14` (internal source `probe_runs/M1/portfolio_decision_compaction-answerable-v2-direct-q8_0-64k-6500-1e01eec2cb14-summary.json`). It is a **single paired control**, not a replicated estimate. Full and compact arms used the same frozen, answerable KPG decision case and direct-answer mode. The score is the harness's required-check pass fraction, not an investment-return metric. Both arms still had one serious failure.

| Arm | Prompt tokens | Wall seconds | Required checks passed | Serious failures |
|---|---:|---:|---:|---:|
| Full | 12,528 | 174.5 | 16/20 | 1 |
| Compact | 9,393 | 163.4 | 17/20 | 1 |

Complete input fell 26.5%; wall time fell 6.3%. The predeclared gate required at least 10% latency reduction, at least 90% compact-arm check pass, and no compact-arm serious failure. The retained summary records `production_compaction_authorized: false`. Earlier thinking-enabled arms hit their output limit without answering; they are diagnostic failures, not additional successful controls. No five-repeat follow-up was run.

## E04 — M6 F16 non-replication: extract

The M6 Q8_0/F16 comparison is retained internally as `probe_runs/M6/context_sweep.jsonl`. Each arm was one shuffled sweep on Ollama 0.35.1, Qwen3.5 Q4_K_M weights, 65,536-token model context, flash attention and a 28,672 MB wired-memory ceiling. The row mappings below use the *server-verified* KV setting, not merely the client environment. The endpoint is approximately **21.8K prompt tokens**; an earlier working table mistakenly called it 24K and gave different 4B endpoint values. The technical report has been corrected against the retained JSONL.

| Model | Server KV | Run ID | Baseline prompt tokens / decode tokens/s | Endpoint prompt tokens / decode tokens/s |
|---|---|---|---:|---:|
| 4B | Q8_0 | `f90147f211f6` | 148 / 45.33 | 21,772 / 40.12 |
| 4B | F16 | `50646c4e8d38` | 148 / 45.60 | 21,770 / 38.95 |
| 27B | Q8_0 | `eb42f194d559` | 150 / 8.98 | 21,776 / 7.90 |
| 27B | F16 | `74a4b282899c` | 147 / 9.05 | 21,771 / 7.87 |

These rows support **no detected F16 advantage on this M6 configuration**, not an intrinsic Q8_0 speed advantage. The full raw records are excluded because they contain machine paths and an embedded model licence.

### The separate M1 result and a provenance trap

The M1 discovery arms (internal source `probe_runs/M1/context_sweep.jsonl`) used Ollama 0.32.15 and the 32K Qwen3.5-4B tag. The later independent F16 replicate, run `da0c2c751e71`, used a separate test server; its retained startup line (`probe_runs/M1/f16-server-da0c2c751e71.txt`) records `OLLAMA_KV_CACHE_TYPE:f16`, flash attention and the 6,500 MB wired-memory ceiling. The probe's generic `server_OLLAMA_KV_CACHE_TYPE` metadata field, however, says `q8_0` because it read the *default* server log, not that isolated server's log. This is why the separate startup witness matters. Do not silently treat that stale metadata field as verification of the test server.

| M1 arm | Run ID | ~150-prompt-token decode tokens/s | ~21.8K-prompt-token decode tokens/s |
|---|---|---:|---:|
| Discovery Q8_0 | `c69971861804` | 16.57 | 7.09 |
| Discovery F16 | `64af12ed846d` | 17.14 | 14.25 |
| F16 replicate, separate-server startup witness | `da0c2c751e71` | 17.32 | 14.09 |

This supports a bounded M1 F16 advantage and its later F16-only replication. It does not isolate hardware from the Ollama/Metal generation when compared with the M6, and it does not turn F16 into a universal recommendation.

## E05 — Historical hidden-token proxy: sanitised aggregate extract

The retained internal `reports/data/llm_call_log.csv` contains call timing and usage fields but also ticker, role, run identifiers and operational metadata. **The entire CSV is not part of this publication set.** This table removes those fields and reproduces the historical cohort from the earliest 321 rows with `finish_reason=stop` and populated `response_chars` and `completion_tokens`, sorted by `timestamp_utc`. The last included timestamp is `2026-09-23T03:00:44+00:00`. Tool-call rows are excluded because they lack a comparable visible answer. The source file continued to grow after this historical snapshot, so rerunning the formula across the whole current file would answer a different question.

| Sanitised cohort | Calls | Completion tokens | Visible response characters | Approximate unexposed share |
|---|---:|---:|---:|---:|
| Earliest qualifying text responses | 321 | 1,526,265 | 1,974,381 | 67.7% |
| Portfolio-decision subset | 32 | Mean 8,878 per call | Mean 4,305 per call | Not separately calculated here |

The share is a **proxy**, calculated as `1 - (visible characters / 4) / completion tokens`, then rounded to one decimal percent. Four characters per visible token is an explicit approximation, not tokenizer output; the 67.7% is therefore **estimated unexposed generation**, not directly counted hidden-reasoning tokens. Separate native-API probes show thinking text can account for such a discrepancy, but this CSV alone cannot prove every unexposed token was reasoning. The cohort rule and aggregate fields above allow recalculation of the proxy; the exhibit excludes ticker, prompts, outputs, holdings, agent role, run identifiers and host information.

## E06 — Results presently supported only by internal summaries or partial source pairs

| Working-report result | Public-evidence state and necessary boundary |
|---|---|
| 67.7% estimated unexposed-token share across 321 calls | E05 now provides the sanitised aggregate, cohort rule and approximation. This is a proxy, not a direct token-level partition of hidden thought. |
| Original M1 QFIN reasoning-off answers and timing | The 23 September probe saved only its table in the internal TODO archive; raw-response saving began in October. The 3,237-versus-67-token and 220.6-versus-5.2-second comparison is therefore **internal-summary evidence only**. Later M6 raw probes are different model/runtime treatments and cannot replace that missing M1 record. The public claim is a case result, not a global reasoning-off error rate. |
| M1 F16 advantage | The M1 subsection below contains selected A/B rows and the separate-server startup witness. The complete raw record is not part of this extract. |
| Frozen KPG/three-ticker financial checks and GPT-OSS net-debt reversal | The internal frozen KPG fixture and saved GPT-OSS/9B reports are retained. The GPT-OSS FACTS block says net cash of about NZD 10.4 million while the frozen figures imply net debt of NZD 1,211,330,992 (`1,221,732,992` debt minus `10,402,000` cash). This is one negative-control case, not a population error rate. The predeclared three-ticker rubric is in the internal TODO archive rather than a standalone scored-answers file. Complete reports are excluded because they contain portfolio detail. |
| Older 4B AFFO inversion | The archived old KPG report says AFFO contracted 7.7%; KPG's FY26 results announcement says AFFO **rose 8.0% to NZD 100.2 million**. The first tracked M6 report at commit `27d2aa5` says AFFO expanded 7.9% to NZD 100.2 million. Internal provenance: `reports_archive/m1-last-run-2026-10-05/reports/markdown/KPG_NZ.md`, `corporate_vault/KPG/announcement_472753_20260825_200040.md`, and that commit's `reports/markdown/KPG_NZ.md`. This is a sanitised three-way case extract; the complete reports are excluded because they contain portfolio detail. |
| SMI.NZ frozen cash comparison | The frozen yfinance block states total cash AUD 335,650,400, while the 4B report says net cash about AUD 188.3 million. However, the issuer's FY26 filing states cash AUD 188,333,447 **at 30 June 2026** (internal source `corporate_vault/SMI/announcement_480789_20260930_073708.md`). The 4B report did not date or clearly attribute that older number and labelled gross cash as net cash. This is a source-selection/attribution and metric-label failure against the frozen snapshot, **not** evidence that 188.3 million was invented. It should not be counted as an invented figure. |
| Buried CMO/NZX/GNE net debt; NTL gross/net mistake | Primary-source and internal-report case reviews. Publish short source pointers, operands, expected direction/units and scored outputs only after removing portfolio context. Do not treat one case as a comparative error rate. |
| NVFP4/MLX approximately 10–15% faster on two earlier *live* NZ/ASX panels with release workaround | Contemporaneous call/run summaries; live inputs and output lengths varied. These historical live percentages remain summary evidence. E07 now gives separate frozen per-pass timing for the unload intervention, but not a public per-call trace or general answer-quality equivalence. |
| 220.8-minute first full-watchlist attempt | Operational internal summary with one rejected report and several simultaneous changes. It is neither an all-valid published run nor an isolated treatment effect. |
| 50-document vault audit, 28 cut at the earlier cap | No manifest was saved. The internal `measure_vault_truncation.py` at commit `a5487f0` is the reproducing method against the vault at that revision. The 28/50 result remains a contemporaneous summary until the historical vault state is rerun and a sanitised per-document manifest is exported. The KPG passage illustrates coverage loss, not a general answer-quality estimate. |
| 27B wrapper interval and host sleep | The exact-window `pmset` check was described contemporaneously, but its filtered output was not retained and cumulative asleep time was not calculated. The precise event count is omitted from the public reports; this remains a plausible, internally observed explanation rather than a numerically closed public timing result. |
| QFIN dividend comparison despite reasoning on | Claude's deterministic `dividend_change_basis` replay over 359 saved report versions flagged three QFIN errors; two used the adjacent-payment 41% result and one called the 0.78-to-0.46 fall 57%. This is a contemporaneous internal replay summary, not a sanitised per-report export or a reasoning-mode A/B. |
| Exact scenario list changed in narration | An internal overview review found one reversed polarity (OGC), one omitted offset, and one assessed ticker called unassessed. The list was moved to Python rendering in `2300b0b`. This is a case review, not an estimated error rate. |
| Strict guard with downstream disappearance | The 14-15 October run rejected SEK.NZ and SMI.NZ, but their prior reports then disappeared from the dashboard/overview input. The consumer path was corrected in `3e37e3a`. This case does not change the frozen-batch guard counts. |

## E07 — Frozen NZ/ASX and US batches: sanitised extract

These tables are selective exports from retained internal `eval_fixtures/batch_20261013_night1_matrix.csv`, `batch_20261014_night2_matrix.csv`, and their rubric-scoring notes. They omit prompts, holdings, raw Equibles payloads, host paths and complete reports. Per-pass elapsed minutes are the sums of the ticker rows, including rejected runs; peak `held` is the maximum runner-reported value in that pass, **not** summed. Publication fractions are reports saved after the guard. The ticker-level files and private `server.log` remain internal. These rows permit checking the stated pass spread, but do not independently reproduce every call join or semantic score.

The NZ/ASX fixture covered KPG.NZ, SMI.NZ and IDX.AX with one recorded manifest hash prefix `9ae0ab022b83d129`. A/B used NVFP4/MLX manifest prefix `f52f2b9a2363`, with ticker unloading off/on respectively; C used Q4_K_M/llama.cpp `3cd1c281e02f` with ticker unloading on. Ollama was 0.40.2 with 80K model context. Its server environment requested Q8_0 KV cache and flash attention. llama.cpp logged both as applied; MLX logged neither its KV precision nor flash-attention setting, so those settings are unverified for A/B. The differing C build and runner cannot identify which of those changes caused a difference.

| Pass | A no unload: min / published / peak held GiB | B ticker unload: min / published / peak held GiB | C Q4_K_M: min / published |
|---:|---|---|---|
| 1 | 76.2 / 3 of 3 / 31.79 | 54.1 / 3 of 3 / 28.51 | 60.0 / 2 of 3 |
| 2 | 71.2 / 3 of 3 / 31.78 | 54.3 / 3 of 3 / 28.72 | 61.3 / 3 of 3 |
| 3 | 72.5 / 3 of 3 / 31.49 | 57.1 / 3 of 3 / 28.90 | 61.2 / 3 of 3 |

Claude, an AI reviewer, scored the frozen reports against the predeclared rubric; Drew did not independently re-score them. An automated figure trace and checks against saved sources assisted the review. It found wrong/invented stated facts A 0, B 0, C 2; completeness gaps A 0, B 3, C 1; checker banners A 0, B 1, C 3. Completeness gaps were flagged by keyword and then read; a missing keyword can reflect phrasing, although B's three appeared to be omissions. C's two factual cases were a $99.99M AFFO value conflated with a 99.9% vote, and an AUD 1,950/oz cost figure recast as a US-dollar gold-price threshold. C1 KPG was rejected for a missing required field. KPG risk scores in A were 46, 69 and 58 on the same fixture. All 278 server requests were assigned one-to-one in the private *completion-time* join; that script does not cross-check token counts or build identity per request. For MLX, prefill/decode rates were *derived* from phase timestamps and API counts, while llama.cpp printed native rates. A matching count alone is not a public audit of every association.

The US fixture covered QFIN and OGC, all on the same NVFP4/MLX build and a 12,000-character Equibles-result cap. Its recorded manifest hash prefix was `5295f535b544dfc0`. F had no unload, G ticker-only unload, and H ticker plus between-agent unload. All three arms kept the cap, so the table measures unload policy, **not cap versus no cap**.

| Pass | F no unload: min / published / peak held GiB | G ticker unload: min / published / peak held GiB | H ticker + agent unload: min / published / peak held GiB |
|---:|---|---|---|
| 1 | 75.5 / 2 of 2 / 32.79 | 72.7 / 1 of 2 / 32.95 | 51.1 / 2 of 2 / 27.70 |
| 2 | 74.6 / 2 of 2 / 31.87 | 81.6 / 2 of 2 / 33.03 | 46.2 / 2 of 2 / 27.47 |
| 3 | 76.0 / 2 of 2 / 31.99 | 46.6 / 2 of 2 / 31.81 | 46.4 / 1 of 2 / 27.16 |

US freeze witness: the Equibles ingestion worker was stopped; the before/after database-statistics diff moved only scheduler bookkeeping tables, not filings/documents/chunks. Of 589 tool calls, 41 canonical calls recurred and all 41 returned byte-identical pre-cap SHA-256 payloads; 496 distinct calls were made and only eight were common to all arms. That establishes repeated-call payload stability, **not** identical retrieval choices or coverage. The 440 server requests yielded 437 completion-time joins; three were deliberately left unassigned. The same call-identity caveat applies. The largest recorded prompt was 66,083 tokens and no context-pressure warning fired.

Claude's single-reviewer quality scoring found:

| Setup | Reports passing the guard | Checked wrong or mislabelled statements | Where found |
|---|---:|---:|---|
| F: no unload | 6 of 6 | 2 in one report | QFIN: trust payables mislabelled off-balance-sheet; dividend cover misread after a faulty cross-currency Python calculation |
| G: between-ticker unload | 5 of 6 | None confirmed | QFIN in the first pass failed the guard |
| H: between-ticker and between-agent unload | 5 of 6 | 1 | OGC: unsupported $3,000/oz breakeven label; QFIN in the third pass failed the guard |

For the dividend-cover error, Python divided CNY cash flow by USD dividends without currency conversion and supplied `102.15`; the F report rendered it as `1.02x`. This was an **input-induced failure and model misreading**, not a model-invented financial figure. The faulty input was shared by all three frozen setups. The live feeder was fixed in `f7d7164` and a later QFIN report showed 15.3x; these frozen fixtures remain unchanged. The fix also found that Yahoo's unlabelled per-share dividend fields disagreed for QFIN, so currency cannot be assumed from those fields. Two additional QFIN figures remain unverified and unscored. Across reports passing the guard, QFIN valuation scores ranged from 28 to 83 under a frozen *source state*; one H report said Sell while the others said Hold. Because agents made different tool calls, these were not repeated responses to identical final prompts. These small, single-reviewer samples do not support a general accuracy ranking or a causal claim that unloading changed the recommendation. Frozen evidence can be consistently *wrong*, as the currency bug demonstrates.

## Revision boundary

A fixed seed did not establish exact NVFP4/MLX replay in this tested runtime. These batches therefore retain three passes and field-level spread. Exact model manifest prefixes, fixture hash, runner, and Ollama version are part of the treatment identity; a model tag alone is not. The semantic scoring was performed by Claude, an AI reviewer, against a rubric, not by an independent human and not as an exhaustive sentence-by-sentence audit. The sanitised aggregate does not permit independent re-scoring of the complete reports.

The source artifacts above were inspected on 13-14 October 2026. A model tag is not an immutable build identifier: `qwen3.6-35b-a3b-80k` resolved to different weights and runners during this investigation. No commit ID is given for records whose revision was not fixed.
