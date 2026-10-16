# From Tokens per Second to Decision Quality

## What a production-shaped local investment-analysis pipeline taught us about reasoning, context, cache precision, and trustworthy evaluation

**Status:** Working GitHub field-report draft  
**Research began:** 23 September 2026  
**Prepared by:** Codex  
**Work represented:** Drew Whittle, Claude, and Codex  
**Plain-language companion:** [*From Faster Answers to Better Decisions*](from-faster-answers-to-better-decisions.md)  
**Public-safe evidence extract:** [Measurements and provenance](evidence-and-provenance.md)  
**Last updated:** 16 October 2026

> This is a field report, not a claim that the underlying inference mechanisms are new. Its intended contribution is to show how several known local-LLM problems collided inside one real system, how we separated them, which conclusions survived measurement, and why no speed optimization was allowed to bypass a financial decision-quality gate.

## Intended technical reader and requested critique

This version is for readers who understand local-model inference, evaluation design, agent systems, Apple Silicon, or analytical quality assurance well enough to challenge the work. It is deliberately more detailed than the plain-language companion because a useful reviewer needs enough information to say:

- the experiment does not isolate the variable it claims to isolate;
- a metric has been interpreted incorrectly;
- an alternative runtime or hardware mechanism better explains the result;
- the sample or replication is too weak for the wording used;
- a capacity, caching, or configuration detail has been missed;
- the quality test does not represent the production decision;
- relevant prior work changes the interpretation; or
- a simpler experiment could distinguish competing explanations.

That kind of criticism is an intended outcome, not a failure of the report. The goal is to make claims narrow and inspectable enough that a knowledgeable reader can falsify them, reproduce them, or suggest the next discriminating test.

## Executive summary

This project runs a multi-agent equity-analysis workflow locally through Ollama. The investigation began with Qwen3.5-4B on an 8 GB M1 Mac mini and later replicated the apparatus on a 32 GB M6 while screening 4B, 9B and 27B dense variants. A single ticker can require many calls across research, announcements, insider activity, valuation, risk, forecasting, thesis validation, and portfolio decision tasks. The workload is evidence-heavy, tool-using, long-context, and dominated by generated reasoning that is not visible in the final report.

The investigation produced nine main lessons:

1. **Most generated tokens were not final-answer prose.** Across 321 textual production calls, a character-to-token proxy estimates that 67.7% of generated tokens were unexposed; native probes support hidden reasoning as an important cause, but the log does not directly partition every token. For `portfolio_decision_task`, the mean was 8,878 completion tokens but only 4,305 visible characters.
2. **Turning reasoning off looked fast but failed a financial case.** A contemporaneous QFIN probe summary recorded non-thinking calls at 60-67 completion tokens and 3.8-5.2 seconds, with both selecting the wrong comparison. One reproduced the known -41% adjacent-period error instead of the correct roughly -39.5% like-for-like comparison. The original raw responses were not saved, so this remains a recorded case rather than independently inspectable public trial evidence.
3. **Longer total sequences slowed decoding severely under the deployed Q8_0 cache configuration.** Four controlled sweeps reduced decode throughput from about 16.8 tokens/s near 150 prompt tokens to about 6.9 tokens/s near 21,770 prompt tokens. A 6,000-token generation experiment showed that generated tokens and prompt tokens followed the same total-sequence slowdown curve.
4. **An M1 F16 result exposed a runtime interaction, but did not generalize to the tested M6 llama.cpp path.** On the M1 test path, replicated F16 sweeps retained substantially more throughput than Q8_0 as context grew. The retained M6 A/B found no material F16 advantage for either 4B or 27B: at about 21.8K prompt tokens, 4B measured 40.12 tokens/s with Q8_0 and 38.95 with F16, while 27B measured 7.90 and 7.87 respectively. Q8_0 remains the memory-efficient choice on that tested path; applied KV precision on the later production MLX runner is unverified.
5. **The projected compaction benefit did not survive its direct gate.** A frozen KPG.NZ comparison reduced complete input by 26.5%, but the only terminating control measured just 6.3% lower wall time and 85% compact-arm quality, below the predeclared 10% and 90% thresholds. Production compaction was rejected and the planned five-repeat phase was cancelled.
6. **Reasoning mode interacted with task complexity rather than having one globally safe setting.** With `think=true`, both full and compact KPG arms consumed all 8,192 allowed tokens in hidden reasoning and emitted no answer. With `think=false`, both terminated normally and agreed on every headline score and decision, but still omitted required facts and structure. This does not overturn the earlier QFIN result where disabling reasoning made the financial comparison wrong; it shows that reasoning mode must be evaluated per workload.
7. **The M6 made dense-model size a separate constraint from residency.** At 64K/Q8_0, 4B, 9B and 27B were all fully GPU-resident, yet decoded at 43.8, 27.9 and 9.1 tokens/s. A frozen KPG screen then took 27.8, 68.6 and 168.6 minutes respectively; 27B repeated 4B's checkable net-debt error while 9B avoided it. The completed three-ticker panel favoured 9B against its frozen yfinance answer key, but the SMI 4B cash figure was subsequently found in a dated issuer filing; that row is a source-date/label failure, not fabrication. The 9B panel required about 274 minutes.
8. **Architecture changed the capacity-speed trade-off.** Qwen3.6-35B-A3B, a 35B-total/~3B-active MoE, was fully resident at 22 GB and decoded at 72.3 tokens/s. It matched all three frozen financial figures, called tools successfully, and completed the panel in about 68 minutes. GPT-OSS 20B fit and called tools but reversed $1.211B net debt into $10.4M net cash, so it was stopped after KPG. Production moved to `qwen3.6-35b-a3b-80k` only after the selected model passed residency again at the actual 80K allocation.
9. **Frozen repeats established a narrow speed and memory result, not a quality winner.** An Ollama refresh silently moved the unchanged production tag from Q4_K_M/llama.cpp to NVFP4/MLX. In three frozen NZ/ASX passes, unloading between tickers reduced three-ticker time from 71-76 to 54-57 minutes and held peak memory below 29 GiB. In three frozen US passes, per-ticker *and* between-agent unloading took 46-51 minutes versus 74-76 without unloading, with peak held memory below 28 GiB. Both panels exposed score/output variation and occasional guard rejections; neither supports general answer-quality equivalence. MLX also accepted observed prompts beyond the configured 80K allocation and did not reproduce exact output at a fixed seed in the tested version. An upstream cap bounded oversized Equibles responses. [The sanitised per-pass evidence and limits are in the appendix](evidence-and-provenance.md#e07--frozen-nzasx-and-us-batches-sanitised-extract).

The project began by asking how to make a slow local model faster. It ended up with a more useful question:

> How do we improve a local AI system without accidentally measuring the wrong layer, changing the evidence, or making the answer worse?

## Evidence labels used in this report

The report deliberately distinguishes evidence strength.

| Label | Meaning |
|---|---|
| **Measured** | Directly recorded by an instrument or raw retained artifact |
| **Replicated** | Independently repeated under materially equivalent controls |
| **Calculated** | Derived from measured inputs using a stated method |
| **Corroborative** | Production-shaped evidence consistent with a controlled result, but not a causal A/B |
| **Inferred** | Best explanation of multiple observations, with alternatives still possible |
| **Planned** | Experiment designed but not yet completed |
| **Rejected** | Tested shortcut or interpretation that did not survive evidence or quality review |

Headline claims should not silently move between these categories.

## 1. The system under study

The application is a CrewAI-based investment-analysis system covering NZX, ASX, HKEX, and US securities. Its normal per-ticker sequence includes:

- market and filing research;
- announcement and insider-activity review;
- mining or REIT specialist analysis when applicable;
- valuation, sentiment, risk, and quantitative forecasting;
- thesis validation; and
- a final portfolio decision.

The agents use yfinance, local announcement vaults, a news-volume signal, portfolio and thesis files, and live Equibles tools for US data. Reports are generated as Markdown and HTML, while call telemetry, warnings, score history, source vaults, tests, TODO decisions, and reviewer discussion remain in the repository.

The original measured production-era runtime, which generated the reasoning/context evidence in Sections 3-8, was:

| Component | Value |
|---|---|
| Host | 8 GB Apple Silicon M1 Mac mini |
| Runtime | Ollama 0.32.15 |
| Model | Qwen3.5-4B, Q4_K_M weights |
| Production model tag | `qwen3.5-4b-64k` |
| Production context allocation | 65,536 tokens |
| Production KV cache | Q8_0 |
| Flash attention | Enabled |
| Analyzer-stage GPU wired-memory ceiling | 6,500 MB |
| Operation | Headless, scheduled batch pipeline |

This matters because the findings are configuration-specific. “Qwen,” “Ollama,” “Apple Silicon,” and even “M1” are too broad to identify the measured system.

## 2. Why this case is useful

Many local-LLM benchmarks ask a clean engine question: how many tokens per second can a model produce at a chosen context length? This project had to answer a messier operational question.

The pipeline contains tool calls, growing evidence, hidden reasoning, long generations, imperfect source data, small-model arithmetic failures, context-window limits, framework overhead, and reports that influence financial decisions. A change that improves one layer can damage another:

- fewer reasoning tokens can increase speed but reduce multi-step accuracy;
- more context can supply evidence but slow every later generated token;
- smaller context can improve residency but silently evict evidence;
- quantized cache can save memory but perform poorly on a specific runtime path;
- a larger model can still fail if the required facts were never supplied;
- a validator can accept structurally valid output that is financially wrong; and
- a live data tool can make two supposedly paired evaluations incomparable.

The useful unit of evaluation is therefore not just the model. It is the complete chain:

> source data -> deterministic processing -> tool boundary -> model prompt -> runtime configuration -> generated reasoning -> validator -> report decision

## 3. How the performance problem surfaced

### 3.1 Production telemetry showed an output mismatch

The first clue was not a benchmark. It was the mismatch between reported completion tokens and visible report text.

For 32 `portfolio_decision_task` calls:

| Metric | Mean |
|---|---:|
| Reported completion tokens | 8,878 |
| Visible response characters | 4,305 |
| Visible characters per completion token | 0.48 |

Ordinary English prose is several characters per token, so the figures could not describe final-answer text alone. Across 321 substantial textual production calls, the character-to-token proxy estimated an unexposed share of 67.7% (cohort and calculation in the [sanitised evidence extract](evidence-and-provenance.md)). This is not a direct count of hidden-reasoning tokens. A previously observed leaked `</think>` block provided a second clue that an internal reasoning stream was being generated and stripped.

This was initially an inference from telemetry. Direct native Ollama calls later returned the reasoning content separately, confirming the mechanism.

### 3.2 The tempting optimization failed its quality gate

A direct probe used a real failure case: selecting the proper year-on-year dividend comparison for QFIN rather than the adjacent payment. The table is preserved in a contemporaneous internal note; the September raw responses were not saved, so it is included as a documented diagnostic rather than a public primary artifact.

| API path | Reasoning setting | Completion tokens | Wall time | Financial result |
|---|---|---:|---:|---|
| OpenAI-compatible | Default | 3,237 | 220.6 s | Correct like-for-like comparison |
| OpenAI-compatible | `reasoning_effort=none` | 67 | 5.2 s | Wrong period; then declined to answer |
| Native Ollama | `think=true` | 1,626 | 100.2 s | Correct like-for-like comparison |
| Native Ollama | `think=false` | 60 | 3.8 s | Wrong adjacent-period comparison (-41.03%) |

The exact timings are not production estimates: the prompt was short, there was one run per condition, and the GPU ceiling differed from the scheduled analyzer configuration. The recorded outcomes were enough to reject a global switch in this project, but they cannot establish a failure rate or be independently re-scored without the missing raw responses. Reasoning-on is **not sufficient** either: a later deterministic `dividend_change_basis` check found three QFIN errors among 359 saved report versions, each with reasoning on and the like-for-like data supplied. Two used the wrong adjacent-payment 41% comparison; one called the same 0.78-to-0.46 change a 57% reduction. The 15 October report used the correct 39% comparison. This is an observational check over saved versions, not a randomized reasoning-on/off comparison; it motivates a deterministic operand-and-arithmetic guard.

**Result:** global reasoning-off was rejected. The recorded test showed that the switch worked and changed this case's answer; it did not license trading correctness for speed.

The later KPG compaction control showed the opposite failure mode on a different task. With forced `think=true`, both a 12,526-token full prompt and a 9,391-token compact prompt consumed all 8,192 allowed output tokens in hidden reasoning and emitted no visible response. With `think=false`, both terminated, agreed on every headline score and returned Watchlist/Watch, although neither met the complete quality contract. Together the cases reject a global reasoning-mode rule: QFIN needed reasoning to select the correct operands, while KPG's dense instruction contract triggered non-terminating deliberation. Reasoning mode is part of the treatment and must be tested against the actual workload.

## 4. Separating prompt ingestion from generation

### 4.1 The first production analysis mixed the two phases

An early analysis divided total wall time by completion-token count and called the result decode speed. That quotient also included prompt ingestion, model load, framework overhead, and tool-related effects. Task identity also changed with prompt length, so the data could not isolate a context mechanism.

The replacement probe held the model and question constant, varied only preceding context, disabled thinking to stabilize short outputs, and used Ollama's native counters:

- `prompt_eval_count` and `prompt_eval_duration` for prefill;
- `eval_count` and `eval_duration` for decode;
- randomized execution order;
- fixed generation settings where needed; and
- a nonce to defeat cross-request prefix reuse.

### 4.2 Prefix caching invalidated an early prefill curve

The original synthetic prompts shared a growing prefix. Ollama reused that prefix, making larger prompts appear implausibly faster to ingest. A unique nonce removed the reuse and reversed the curve.

This is a practical warning for long-context tests: if prompt identity is not controlled, a benchmark may measure cache reuse rather than context scaling.

### 4.3 Four Q8_0 sweeps reproduced the decode decline

The controlled Q8_0 sweeps measured the following approximate decode throughput:

| Prompt tokens | Run 1 | Run 2 | Run 3 | Provenanced run 4 |
|---:|---:|---:|---:|---:|
| 150 | 16.69 | 16.64 | 17.04 | 16.86 |
| 1,850 | 15.51 | 15.43 | 14.77 | 14.55 |
| 3,650 | 13.67 | 13.07 | 13.35 | 13.67 |
| 7,250 | 11.21 | 11.32 | 11.44 | 11.34 |
| 14,450 | 8.69 | 8.64 | 8.62 | 8.49 |
| 21,770 | 7.03 | 6.95 | 6.88 | 6.92 |

Each run ended at roughly 40-42% of its short-context baseline. The fourth run, ID `f61f8a5ef385`, is the first with the complete retained environment record. The preceding three are close replications but have weaker provenance.

For the largest row in run 4, native load, prefill, and decode durations summed to about 165.0 seconds against 165.3 seconds measured externally. That reconciliation showed that the phase accounting covered essentially the full request.

**Narrow claim:** under the deployed M1/Ollama/Qwen3.5-4B/Q8_0 path, decode throughput fell sharply as the total sequence grew. This was never evidence for a universal 12K “knee” or a universal Apple-Silicon curve.

## 5. Prompt tokens and generated tokens have the same downstream cost

The context sweep measured decode rate near the beginning of short generations. The production portfolio task, however, generated thousands of tokens. A second experiment asked whether those generated tokens moved the model down the same curve.

Run `3410b873627f` generated 6,000 tokens from three starting contexts. The within-generation slowdown closely matched the independent cross-prompt slowdown over the same sequence ranges:

| Starting prompt | Final/initial rate within generation | Cross-prompt rate over the same span |
|---:|---:|---:|
| 97 | 71.3% | 73.3% |
| 7,205 | 80.1% | 80.1% |
| 14,404 | 85.9% | 85.6% |

This supports total sequence length as the operative variable: a generated token adds to the cost of decoding every token after it, just as a prompt token does.

The stream chunks were not one-to-one tokens, so intermediate positions used an approximately 6% whole-response correction. Native endpoint totals remain reliable; exact within-generation window locations are approximate.

## 6. The 24% compaction projection failed its direct authorization gate

Applying the measured Q8_0 curve to the production `portfolio_decision_task` suggested that reducing a roughly 14.4K-token starting prompt to 7.2K could save about 24% for that task.

The long-generation experiment validated the integration assumption behind the calculation. It did **not** directly measure a paired production call. The 24% figure is therefore:

- **calculated**, not a measured production result;
- specific to the deployed M1/Q8_0 performance curve;
- scoped to `portfolio_decision_task`, not the whole pipeline;
- based on synthetic engine measurements with thinking disabled; and
- still conditional on a compact prompt preserving every decision-relevant fact.

The decisive comparison was subsequently built from a hash-pinned KPG.NZ transcript. Its first preflight checked twelve retained facts but missed a more basic answerability defect: the final task required a sentiment score which no upstream task had supplied. A complete output-contract audit then classified all 18 mandatory fields as carried evidence, deterministic derivation or requested model judgment. That audit also found that the compact selector had discarded the mining task's complete one-line result, `Not applicable`, and that `THESIS_STATUS` was required by the prose and rubric but absent from the output template.

Three one-pair smoke tests were retained:

| Run | Mode | Full result | Compact result | Interpretation |
|---|---|---|---|---|
| `3eb2cc2ade17` | Historical prompt, `think=true` | 8,192 output tokens; no visible answer | 8,192; no visible answer | Both looped over an unanswerable sentiment-score contract |
| `66621dc49982` | Answerable-v2, `think=true` | 8,192; no visible answer | 8,192; no visible answer | The repair moved the loop: full debated the required delimiter; compact endlessly rechecked its schema |
| `1e01eec2cb14` | Answerable-v2, `think=false` | 796 tokens; 174.5 s; 16/20 | 1,080 tokens; 163.4 s; 17/20 | Both terminated and agreed, but failed quality and latency gates |

The native request initially had no output ceiling: one aborted full-context call passed 33,330 generated tokens and reached the two-hour HTTP timeout before Ollama returned, leaving no valid result row. The corrected harness sends equal `num_predict=8192` limits, records `done_reason`, rejects length-limited output, and prevents two missing recommendations from counting as stable merely because `null == null`.

The direct-answer control isolated forced hidden reasoning as the immediate cause of the runaway for this model/task combination. It did **not** authorize compaction. Compact quality was 85% versus the required 90%; both arms omitted machine-readable `THESIS_STATUS`, exact NTA per share and tracked cost; full additionally omitted exact occupancy. Compact improved wall time by only 6.3%, below the predeclared 10%, partly because its visible answer was longer. Both arms nevertheless returned identical headline scores, Watchlist classification and Watch recommendation, illustrating why decision agreement alone is not a sufficient quality gate.

The approximately 24% value remains a valid calculation from the synthetic Q8_0 curve, but it did not predict the measured end-to-end benefit of this production-shaped pair. The tested compactor is rejected, the five-repeat phase is cancelled, and no production integration is authorized.

## 7. The F16 result changed the causal story, then the M6 narrowed it again

### 7.1 A literature clue exposed an untested variable

A later comment in a llama.cpp discussion supplied a concrete clue: a [Qwen3.6-27B/M5 Max Q8_0 context-depth benchmark](https://github.com/ggml-org/llama.cpp/discussions/12985#discussioncomment-18063341) showed generation throughput falling as context depth increased, [llama.cpp maintainer Georgi Gerganov warned that quantized cache on Metal was not optimized](https://github.com/ggml-org/llama.cpp/discussions/12985#discussioncomment-18063447), and the benchmark author's [partial F16-cache rerun](https://github.com/ggml-org/llama.cpp/discussions/12985#discussioncomment-18063832) was faster at matched depths. This did not establish the behaviour of our Ollama build, but it exposed cache precision as a variable our original experiment had not isolated. Q8_0 had been chosen to make long context fit in limited memory, so cache precision had been part of the production configuration but not part of the original causal model.

A controlled discovery sweep restarted the same private Ollama server between arms and used the same Qwen3.5-4B model tag and prompt schedule.

The retained arm identifiers are `c69971861804` for Q8_0 and `64af12ed846d` for F16. The measured rows are reproduced below and in the [evidence appendix](evidence-and-provenance.md#the-separate-m1-result-and-a-provenance-trap).

| Prompt tokens | Q8_0 decode tok/s | F16 decode tok/s | F16/Q8_0 |
|---:|---:|---:|---:|
| 147 | 16.57 | 17.14 | 1.03x |
| 1,856 | 15.19 | 16.99 | 1.12x |
| 3,657 | 13.56 | 16.49 | 1.22x |
| 7,258 | 11.21 | 16.17 | 1.44x |
| 14,456 | 8.64 | 15.09 | 1.75x |
| 21,769 | 7.09 | 14.25 | 2.01x |

At the largest tested prompt, F16 retained about 83% of its short-context baseline while Q8_0 retained about 43%.

This was a major correction to the M1 explanation, but it did not establish a portable rule:

- the Q8_0 curve was real, repeated, and represented the deployed configuration;
- F16 still slowed with context;
- the steep Q8_0 decline cannot be attributed to context length alone; and
- the defensible explanation is a cache-precision/runtime interaction on this particular M1 Metal/Ollama path.

The F16 arm was later independently replicated with server-verified provenance as run `da0c2c751e71`. At approximately 21.8K prompt tokens it produced 14.09 tokens/s versus 14.25 in the discovery run, retained 81.4% of its short-context rate versus 83%, and reproduced five of six sweep points within 2.5% (the remaining point differed by 6.3%). Its captured startup line records F16 cache, flash attention, port 11435 and the 6,500 MB GPU ceiling. This promotes the M1 F16 flatness finding from a one-run clue to a replicated result. The later M6 experiment shows why even a replicated mechanism finding must remain bounded to its tested runtime and hardware.

### 7.2 The completed M6 A/B did not reproduce an F16 advantage

The planned 32 GB M6 comparison was completed on 5 October 2026. Both Qwen3.5-4B and Qwen3.5-27B were tested from about 150 to 21.8K prompt tokens under Q8_0 and F16, with flash attention enabled. The wired-memory ceiling was raised to 28,672 MB so that 27B/F16 had genuine headroom; `ollama ps` confirmed 20 GB and 100% GPU placement before the F16 rows were accepted. A previous working table described the endpoint as 24K and gave 4B values of 39.78/38.76; re-reading the retained JSONL gives the corrected prompt-token counts and rates below.

| Model | Cache | Baseline decode tok/s | ~21.8K decode tok/s | Retained |
|---|---|---:|---:|---:|
| 4B | Q8_0 | 45.33 | 40.12 | 89% |
| 4B | F16 | 45.60 | 38.95 | 85% |
| 27B | Q8_0 | 8.98 | 7.90 | 88% |
| 27B | F16 | 9.05 | 7.87 | 87% |

No material F16 advantage was detected. The small Q8_0 leads at ~21.8K came from one shuffled sweep per arm and are not sufficient to claim that Q8_0 is intrinsically faster. They are sufficient to reject the expected portable F16 speedup on this M6 llama.cpp configuration. Q8_0 remains the memory-efficient choice for that path because it consumes less memory without a detected throughput penalty; these measurements do not establish the KV precision applied by the later MLX runner.

This does not invalidate the replicated M1 result. It narrows it: cache precision interacted strongly with the older M1 runtime path, but cache precision was not the dominant cause of long-context decline on the M6. Hardware, model, runtime implementation and cache format form one treatment unless they are isolated explicitly.

### 7.3 Effective configuration had to be observed, not assumed

Several failed residency checks were method failures rather than useful model results:

- `launchctl setenv` did not reliably reach the Ollama server process;
- the pipeline had restored `iogpu.wired_limit_mb` to the OS default before some tests;
- one server was restarted under Q8_0 while the run was still labelled F16; and
- the probe initially recorded the client environment rather than the server-effective configuration.

The resident allocation became a critical witness. An “F16” server showing the same or smaller memory footprint than a known Q8_0 load exposed configuration errors. The more reliable process is to run a private server that owns its environment, capture its startup configuration, verify the GPU ceiling, and record actual model placement and resident size.

The broader lesson is simple:

> An accepted parameter is not evidence that the runtime used it.

### 7.4 A production-shaped M1 KPG run corroborated the historical M1 effect

A later KPG.NZ run used a 35,840-context F16 tag through the full pipeline. Across ten matched final task names, its effective throughput relative to each task's historical 64K/Q8_0 median ranged from 1.36x to 2.74x, with a median of 1.74x. Codex independently reproduced the median as 1.738x.

That agreement with the M1 controlled sweep's roughly 1.75x value at production-sized contexts is historically informative. It is labelled **effective-throughput corroboration**, not a causal cache measurement or an M6 deployment prediction, because:

- the current requests were not paired with the historical requests;
- one run was compared with medians from five uncontrolled historical runs;
- `completion_tokens / duration` includes prefill and framework overhead;
- context allocation and cache precision both changed; and
- the per-task speedups varied widely.

The largest KPG call used 30,468 total tokens and none ended with a length stop. This shows the run completed within that particular observed budget; it does not establish that a 35,840 window is generally safe for future production calls.

## 8. Capacity is part of correctness

The F16 cache was fully resident at 35,840 tokens on the 8 GB M1 test setup, while the next tested allocation spilled. That made a smaller F16 production tag look attractive, but context capacity must cover the prompt **and** the generation, including hidden reasoning.

An initial historical analysis found a KPG `reit_task` using 33,589 prompt tokens plus 3,601 completion tokens: 37,190 total. That looked like a decisive counterexample. Review then found that the call predated commit `dad9636`, which replaced model interpretation of REIT metrics with deterministic Python parsing. The very next run reduced the REIT prompt to 7,118 tokens.

The corrected split was:

| | Before `dad9636` | After `dad9636` |
|---|---:|---:|
| NZ/AX calls | 252 | 209 |
| Maximum total tokens | 37,190 | 31,639 |
| Calls over 35,840 | 1 | 0 |
| Maximum prompt tokens | 33,589 | 28,798 |
| Maximum completion tokens | 12,496 | 11,735 |
| Prompts above a max-completion reserve | 6.7% | 5.3% |

The correction changes degree, not the production decision:

- the post-change sample is short and non-stationary;
- vaults grow as new filings arrive;
- completion length is not known safely in advance;
- context shifting can let a call finish after early evidence has been evicted; and
- a completed response is not proof that the full input remained available.

At that stage, general M1 production therefore remained on 64K/Q8_0. The 35,840/F16 configuration was suitable only for controlled calls whose complete token budget was known in advance. The later M6 decision also retained Q8_0, but because its direct A/B found no material F16 speed advantage rather than because F16 could not fit.

The episode produced a more interesting lesson than the original counterexample: deterministic parsing added for correctness removed about 26,000 prompt tokens as a side effect. Context size is partly a property of the evidence architecture and code version, not a permanent property of a ticker or task.

## 9. Data supply can dominate model capability

The project also compared small-model and larger-model reports in a separate one-ticker review. The purpose was not to decide whether Claude should replace Qwen; it was to ask whether a larger or more capable model could do better on the same analytical work.

The comparison was informative but not a blind model benchmark. The runs occurred on adjacent days over a weekend, when the underlying market information had not materially changed, but the larger model had access to the earlier Qwen report. Claude stated that it did not rely on that report, yet access alone means the arms were not independently blinded. One model session also performed all specialist roles, and only one ticker was examined. The result can identify failure modes and improve the evaluation design; it cannot establish a general model ranking.

The exercise found two complementary failure classes:

- the larger model produced more evidence-aware and internally coherent analysis in some areas; and
- independent review still found issues the larger model had not noticed.

More importantly, the exercise exposed questions for which the smaller model had not been supplied the necessary data. Calling those failures “model limitations” would have been misleading. A model cannot reliably infer a fact absent from its evidence, and a stronger model can merely produce a more persuasive unsupported answer.

For this report, model comparison is therefore decomposed into:

1. Was the question answerable from the supplied evidence?
2. Did retrieval and deterministic processing preserve that evidence?
3. Did the model select and combine the right facts?
4. Did the validator check factual and financial semantics, or only structure?
5. Was the final decision consistent with the preceding analysis?

This is why the frozen evaluation suite must preserve real operands and failure traps rather than simply remove unstable tools.

## 10. The frozen quality gate

### 10.1 Why live tools cannot define a paired comparison

Two live Equibles calls cannot be guaranteed to return the same document population. If full and compact prompts receive different source material, the comparison measures data drift as well as prompt treatment.

Returning `None` from the tools would be deterministic but would delete the evidence that makes QFIN valuable. It would turn a period-selection test into a missing-data test.

The agreed design is:

1. Freeze the exact upstream task outputs and inject them into `portfolio_decision_task`.
2. Build full and compact contexts from that same evidence.
3. Disable tools for both arms so neither can reach the network.
4. Freeze the answer key before viewing outputs.
5. Repeat both arms with reasoning enabled.
6. Score latency and quality separately.

Captured tool replay remains an optional integration test if keyed by canonical arguments, dated, hashed, and configured to fail loudly on any unrecorded call. A live run remains an integration smoke test only.

### 10.2 Evaluation cases

The earlier *planned compaction panel* covered different demonstrated failure modes. It is not the completed frozen NZ/ASX panel (KPG.NZ, SMI.NZ, IDX.AX) or US panel (QFIN, OGC):

| Ticker | Why it is useful |
|---|---|
| QFIN | Like-for-like period selection; known tempting -41% adjacent-period answer |
| KPG.NZ | REIT metrics, large historical contexts, deterministic extraction precedent |
| PLS.AX | Mining-specialist and evidence-integration case |
| IDX.AX | Prior larger-model comparison and report-consistency review |
| S32.AX | Mining, balance-sheet, and report-grounding case |

QFIN's answer key must identify the dates, values, correct operand pair, expected percentage, acceptable wording, and known wrong answer before either treatment output is inspected.

### 10.3 What must be scored

The comparison is not passed by valid Markdown or zero parser warnings. It needs separate scores for:

- required fact retention;
- source attribution and period matching;
- unit, sign, scale, and arithmetic correctness;
- unsupported claims and contradictions;
- appropriate abstention when evidence is missing;
- required report structure;
- recommendation and action consistency;
- instruction compliance; and
- run-to-run stability.

The frozen experiment measured one task in isolation and failed its gate. Even a successful result would not have proved that compacting the entire pipeline was safe or that 35,840 tokens fit every task.

### 10.4 A publication gate is not an identity or meaning gate

A 10 October 2026 three-ticker run produced substantive CMO.NZ, SMI.NZ and RGI.NZ analysis but rejected all three reports because the publication guard required a Markdown `##` heading after `===FINAL===`. The task instruction explicitly allowed a natural heading *if* used. This was a prompt-validator mismatch, not evidence that those analyses lacked substance. The guard was changed to accept sufficiently substantive unheaded prose and regression-tested. On the subsequent live retry, all three reports were saved; however, all three generations happened to include `##` headings. The retry proves publication succeeded, while the unit tests, not that retry, exercise the unheaded path.

A later SEK.NZ response omitted only `RISK_CATEGORY` while carrying numeric `RISK_SCORE_CARRIED: 46`. This label is a deterministic function of the score. Rather than weakening the publication guard to accept a missing required key, the pipeline now inserts that one formula-derived line immediately before validation, explicitly marks that the model omitted it, and leaves all other malformed cases to fail. A real-report removal-and-restoration regression checks both sides of the boundary. The method is to repair only a fact code can derive unambiguously, disclose the repair, then keep the guard strict.

A 14-15 October production run supplied a downstream counterexample to treating rejection as finished work. The guard rejected SEK.NZ for a missing FACTS closing marker and SMI.NZ for leaked thinking plus duplicated blocks. It kept the rejected reports from being replaced, but the two held tickers then disappeared from the dashboard and portfolio overview input. Commit `3e37e3a` corrected that consumer path. A fail-closed gate must preserve the last good result in **every** downstream view, not merely leave a file on disk.

The portfolio overview provided a separate deterministic-boundary case. Python supplied an exact list of which holdings a macro scenario helped or hurt, but model narration reversed OGC's polarity, omitted the only offset, and called an assessed ticker unassessed. The list is now rendered directly by Python (`2300b0b`) rather than passed back to the model for restatement. Supplying a fact does not preserve it if a later model stage is allowed to rewrite it.

The saved RGI.NZ report illustrates a different gate. Its researcher input correctly identified Rua Gold Inc. as `RGI.NZ`, yet the report's analysis introduced `RUG.NZ` once. The `unknown_ticker` warning caught this invented output symbol but did not block publication. The report otherwise concerned Rua Gold and did not contain `RUA.NZ` or Rua Bioscience facts. This is **not** evidence that the earlier source-company collision recurred: [NZX identifies RGI as Rua Gold](https://www.nzx.com/companies/RGI), while [RUA.NZ is Rua Bioscience](https://www.nzx.com/companies/RUA/announcements); [Rua Gold also uses RUA on the TSX](https://ruagold.com/stock-information/). The collision remains a separate, important regression case.

The same report's `valuation_na_vs_numeric` warning also needs semantic interpretation. A numeric valuation score need not contradict the statement that earnings-based multiples are unhelpful for a pre-revenue company. But calling P/B mathematically inapplicable was unsupported here: the supplied input included a P/B value of 5.6712317. Treating every warning as either a definite error or a harmless false alarm would lose that distinction.

The frozen SMI applicability flips later recurred on RGI: a report quoted its NZ$1.48/share scenario value while marking valuation N/A. For miners with a published-study scenario supplied by Python, applicability is now set deterministically (`2e89e6e`) rather than left to the model. Separately, checker changes for an operand borrowed across “while,” the word “non-existential,” and a valid SEC operating-margin label were replayed over saved reports since 1 September; the targeted false positives disappeared while known true positives remained. That replay tests the checker itself, not just the reports it judges.

Together these observations separate three checks: verify source-company identity before analysis, enforce an output contract consistent with the prompt, and inspect claims in a structurally valid report. Warning-only identity detection leaves a publication-policy question open; this single run does not settle the appropriate blocking rule.

## 11. Corrections that improved the investigation

The report intentionally preserves the following changes of mind.

| Initial claim or method | What challenged it | Revised claim |
|---|---|---|
| Wall time divided by completion tokens was called decode speed | Native prefill and decode timings | The quotient mixed multiple phases and could not establish decode behavior |
| Production logs suggested a 12K context “knee” | Controlled constant-task sweeps | Throughput declined continuously; no universal knee was established |
| Larger synthetic prompts appeared faster to prefill | Unique per-request nonce | Prefix caching had contaminated the measurement |
| The calculated 24% compaction saving was briefly called measured | Review of the long-generation scope | The mechanism was measured; 24% remains a configuration-specific calculation |
| Q8_0's steep M1 curve was treated as general context behavior | M1 F16 sweep, then M6 Q8_0/F16 A/B | F16 amplified the distinction on the M1 path, but no material cache-speed difference appeared on the M6; the interaction was not portable |
| Setting an environment variable was treated as setting the server | Resident-size and startup-log mismatch | Effective server configuration must be observed independently |
| 35,840 tokens appeared to cover NZ/ASX prompts | Prompt plus completion accounting | Capacity needs a predeclared generation reserve, not a prompt-only check |
| A 37,190-token KPG call was treated as current evidence | Commit-time split at `dad9636` | It was valid history but superseded by deterministic parsing; present capacity evidence must be version-conditioned |
| A completed call was treated as evidence of no truncation | Context-shifting behavior | Completion does not prove all early evidence remained available |
| Returning no Equibles data looked like a stable QFIN fixture | Review of what the test was meant to measure | Freeze and inject the actual evidence; keep missing-data behavior as a separate test |
| Twelve retained facts were treated as proof that the fixture was answerable | Audit of every mandatory output field | Answerability must cover the complete output contract, including whether a value is carried, derived or judged |
| An unbounded native call was allowed to demonstrate failure by timing out | More than 33,330 generated tokens and a two-hour timeout | Every evaluation needs an output ceiling, finish reason and a recorded length-limit failure |
| Equal missing recommendations counted as stable because `null == null` | Inspection of the first paired summary | Stability and agreement require real non-empty decisions |
| A 26.5% input reduction was expected to approach the 24% projection | Direct paired control | Output length and compliance dominated; measured wall reduction was 6.3% and the treatment failed quality |

These are not embarrassing footnotes. They are the reusable method:

> initial claim -> objection -> discriminating experiment -> result -> narrower claim -> remaining limitation

## 12. Current decisions

As of this draft:

- **Production uses `qwen3.6-35b-a3b-80k` on the M6, currently resolving to NVFP4/MLX.** Its original Q4_K_M/llama.cpp 80K residency check measured 22.0 GB, 100% GPU and 70.6 decode tokens/s; that historical row must not be mislabelled as a current MLX measurement. MLX is faster on the tested NZ/ASX workload only with memory-reset workarounds; seed repeatability, context enforcement and answer-quality parity are not inherited from the old build.
- **Keep the Equibles output cap and context-pressure warning; treat model reloads as removable workarounds.** The frozen US F/G/H panel supports unloading between agents as well as tickers under the tested memory behaviour. Re-test without reloads after an Ollama memory-management fix.
- **Keep the corrected cross-currency dividend-cover feeder under regression test.** The frozen QFIN input mixed CNY cash flow with USD dividends and remains unchanged as a record of what those runs saw. The live feeder was fixed in `f7d7164`: it uses trailing-year payments in the trading currency, converts free cash flow at an FX rate, and returns N/A if either input is missing. Provider dividend fields without currency labels remain unsafe to use directly.
- **Retain the M1 F16 result as bounded historical evidence.** Run `da0c2c751e71` independently reproduced the M1 behavior, but the M6 did not reproduce the advantage.
- **Do not deploy the tested compactor.** It failed the frozen quality and latency gates; the planned five repeats are cancelled.
- **Treat the historical M1 KPG 1.74x result as corroboration only.** It supports that M1 configuration's observation but does not predict the M6.
- **Eliminate dense 27B at the tested Q4_K_M/64K configuration.** It required 168.6 internal minutes for one KPG ticker, decoded at about 9 tokens/s and showed no quality advantage over the smaller candidates.
- **Retain 9B as the best tested dense baseline, not the production model.** It beat 4B on the completed three-ticker objective panel but was about four times slower than the selected MoE across those tickers.
- **Stop GPT-OSS 20B after KPG.** Residency and tool calling passed, but its net-cash/net-debt polarity reversal failed the financial-quality gate.
- **Validate the production change continuously against primary sources.** Preserve the last M1 reports and compare each checkable new figure against both the old output and the filing that supplied it.
- **Do not choose reasoning mode globally.** QFIN required reasoning for the correct financial comparison, while KPG's dense decision contract failed to terminate with forced hidden reasoning and completed with direct answering. Test the actual workload.
- **Do not infer model failure when evidence was absent.** Test answerability and data supply first.
- **Do not treat structural validity as analytical quality.** Keep semantic and decision checks separate.
- **Defer `num_batch` and higher weight precision to their own A/B tests.** Their current benefits are hypotheses or modelled estimates, not findings.

## 13. Completed M6 replication and model-size screen

The 32 GB M6 replication retained the original decomposition rather than reporting one confounded before/after number. At 64K/Q8_0, Qwen3.5-4B improved from 16.9 decode and 113.1 prefill tokens/s on the retained M1 screen to 43.8 and 482.6 on the M6: approximately 2.6x decode and 4.3x prefill. This is an observed system-generation comparison, not a chip-only causal estimate, because the operating system and Ollama versions also changed.

The same M6 screen then varied model size while keeping 64K, Q8_0, flash attention and Q4_K_M weights fixed:

| Model | Resident size | Placement | Decode tok/s | Prefill tok/s |
|---|---:|---:|---:|---:|
| Qwen3.5-4B | 4.9 GB | 100% GPU | 43.8 | 482.6 |
| Qwen3.5-9B | 7.9 GB | 100% GPU | 27.9 | 262.0 |
| Qwen3.5-27B | 18.0 GB | 100% GPU | 9.1 | 80.7 |

Full residency did not make dense 27B practical. A frozen KPG.NZ pipeline screen recorded 27.8, 68.6 and 168.6 internal minutes for 4B, 9B and 27B. All three returned Hold/Core Long-Term Holding. The automated checker found the same gross-debt-versus-net-debt error in 4B and 27B, while 9B avoided it; 27B also introduced an unsupported available-capital figure. This was a deliberate three-model by one-run elimination screen, not a replicated benchmark. It supports eliminating a clearly slow candidate with no observed quality edge; it does not prove that 9B is generally more accurate.

A parallel *pre-upgrade* Ollama MLX-tag screen did not rescue any dense size: 4B, 9B and 27B decoded at 40.7, 24.5 and 8.8 tokens/s, versus 43.8, 27.9 and 9.1 for the standard Ollama tags. This dated dense-model screen is not the present production MoE result; the later Ollama 0.40.2 NVFP4/MLX path proved faster under controlled per-ticker release (section 13.7).

The retained M1 residency screen also resolved an apparent memory discontinuity. Displayed allocation rose from 4.2 GB at 64K/100% GPU to 5.1 GB at 73K/83% GPU. The same jump appeared at a different context boundary under F16. It therefore tracked Ollama's change from GPU-only to mixed CPU/GPU placement rather than a sudden 0.9 GB increment of KV cache. Resident size must be interpreted together with placement; it is not a direct KV-growth curve by itself.

Every dense candidate in these screens used Q4_K_M **weight** quantization. That is a different variable from Q8_0 versus F16 **KV-cache** precision. The present model-size comparison does not isolate parameter count from weight precision, so any later higher-precision weight test must use the same frozen quality panel rather than being folded into a model-size claim.

### 13.1 Why the dense 27B result is probably a bandwidth ceiling

The approximately 18 GB quantized dense model must stream most of its weights for each generated token. Apple specifies 170 GB/s memory bandwidth for the [32 GB M6 Mac mini configuration](https://support.apple.com/en-us/128108). Dividing that by the footprint gives a rough 9.4 token/s ceiling before overhead, closely matching the measured 8.9-9.1. This is an inference from hardware and observed throughput, corroborated by independent long-context Apple-Silicon measurements, not a direct measurement of every byte transferred.

The implication is operationally important: raising the wired-memory ceiling prevents CPU offload but cannot create memory bandwidth. Prompt caching and batching can reduce load or prefill cost, and parallel requests may improve aggregate multi-ticker throughput, but none makes one bandwidth-bound report decode several times faster. Parallel contexts also consume memory independently.

### 13.2 Timing provenance exposed another unresolved layer

The 27B run's internal `run_times.csv` recorded 168.6 minutes, while the outer experiment wrapper spanned 428.5 minutes. The approximately 260-minute difference occurred between Ollama calls. Review ruled out a repeated CrewAI task, a logged exception and a slow local vault read. A contemporaneous look at the power log was initially recorded as ruling out sleep, but it had not checked for the relevant event signature. A later exact-window query was recorded as finding repeated Maintenance Sleep/DarkWake cycling during the interval. The filtered power-log extract was not retained, so its exact transition count and first-event time remain internal-summary observations rather than public measurements.

This strengthens a general method point: retain model-native timings, application timings and outer wall time separately. Agreement between them is evidence; disagreement is a result to investigate, not a number to silently discard.

A later unscoped M6 production run made the mechanism visible. The machine entered system sleep after its SSH connection ended even though the pipeline remained alive inside detached `screen`; roughly seven hours later, incoming network traffic caused a dark wake and normal output resumed. `ttyskeepawake 1` protects an active terminal, not a detached job after the login disappears. The persistent mitigation was `pmset -a sleep 0`.

The recorded power-log check supports host sleep as an explanation for much of the 27B wrapper interval, consistent with a separately observed overnight sleep interruption. It does not close the approximately 260-minute accounting gap numerically: cumulative time asleep was not summed and the filtered extract was not preserved. The internal 168.6-minute application measurement remains the fairer active-run comparison than the outer wall clock. The general diagnostic rule is unchanged: capture power events alongside process, application and inference timings, and distinguish active computation from suspended wall time before treating a silent gap as model latency.

### 13.3 The MoE hypothesis survived the local gates

The dense-model result motivated a bounded Mixture-of-Experts screen. Qwen3.6-35B-A3B stores 35B total parameters while activating approximately 3B per token. Its regular Q4_K_M build measured 22.0 GB, 100% GPU and 72.3 decode tokens/s at the 64K screen--faster than the fully resident dense 4B, 9B and 27B candidates. The separately tested MLX/NVFP4 build measured 23.0 GB and 49.0 tokens/s, so an external report of a large NVFP4 advantage did not reproduce in that *earlier screen* and the branch was stopped then. Later production-shaped tests on Ollama 0.40.2 reversed the operational speed comparison when MLX memory was reset between tickers (section 13.7); the older screen remains valid only for its own runtime and workload.

The same frozen KPG.NZ/SMI.NZ/IDX.AX panel then produced:

| Ticker | Qwen3.6 MoE time | Objective source check | Tool/warning result |
|---|---:|---|---|
| KPG.NZ | 20.8 min | Net debt $1,211.3M, correct | Three vault calls succeeded; one prompt-contract mismatch plus minor presentation issues |
| SMI.NZ | 21.3 min | Cash $335.6M, correct | No warnings |
| IDX.AX | 26.1 min | Net debt $545.8M, correct | No warnings |

The approximately 68-minute total compares with roughly 274 minutes for 9B. The KPG classification/recommendation warning was traced to an evaluation-contract gap: `tasks.yaml` presented the two fields as independent enums, while the allowed combinations existed only in downstream validation. A model cannot comply with an unstated constraint. A related audit found that the recommendation enum offered no Sell/Exit value even though strategy policy says exit remains available as a last resort. These are system-contract defects, not evidence that the model should infer a hidden rule.

GPT-OSS 20B provided a useful negative control. It was fully resident at 12 GB, decoded at 39.3 tokens/s, completed KPG in 17.1 minutes and made three successful vault calls, showing that its Harmony tool format was compatible. It nevertheless reported NZD $10.4M net cash against a frozen source fact of NZD $1.211B net debt. Further runs would have estimated the frequency of a failure already severe enough to reject the candidate, rather than resolving a compatibility question, so the panel stopped there.

Before production activation, Qwen3.6 was rebuilt at the actual 80,000-token allocation and re-screened: 22.0 GB, 100% GPU and 70.6 tokens/s, essentially unchanged from its 64K row. The allocation was raised because real production conversations had previously approached 70K, not simply because spare memory existed. Production then cut over to `qwen3.6-35b-a3b-80k`; the last M1/Qwen3.5-4B reports were archived verbatim for continuing paired review.

### 13.4 A correct magnitude can still have the wrong direction

The first primary-source-checked production comparison found a defect that the frozen panel had not surfaced. The archived M1/4B KPG report wrote that AFFO “contracts (down 7.7% YoY).” The new M6/Qwen3.6 report wrote “AFFO growth of 7.9% to $100.2 million in FY26.” KPG's FY26 results announcement states that AFFO increased 8% to $100.2 million, matching the new report.

The old output had preserved the approximate percentage and exact dollar amount while reversing the direction. This is the same semantic failure class as the GPT-OSS net-cash/net-debt reversal: numeric proximity is not semantic correctness. Evaluation of financial facts must score attribution, period, unit, scale, sign and direction separately. One corrected production figure does not prove that every Qwen3.6 figure is superior; it justifies the ongoing plan to compare every checkable new figure with both the archived report and the primary source.

### 13.5 Present in the prompt is not the same as operationally available

The next two production checks found the opposite attribution pattern: the new model failed, but repeated behavior exposed a prompt-interface defect as well. For CMO.NZ, the rendered source block contained:

```text
Net Debt/EBITDA: 3.31x (Net Debt: 144576000, EBITDA: 43703000)
Total Cash: NZD 10,976,000 | Total Debt: NZD 155,552,000
```

The archived 4B report stated the correct $144.6M net debt. Qwen3.6 instead reported the nearby 47.7% debt-to-equity ratio and said the exact dollar figure was not provided. NZX.NZ then reproduced the same failure shape: net debt appeared only as a raw parenthetical supporting its Net Debt/EBITDA ratio, and the report again claimed that only the ratio had been supplied.

The model's unavailability claim was false: the number was literally present. But the intended fact was raw, unlabelled with currency, unformatted and subordinate to a ratio, while peer dollar facts received dedicated labelled lines. Different models can have different sensitivity to that asymmetry. The old model noticing the parenthetical does not make the interface robust, and the new model missing it does not make presentation irrelevant.

This extends the evidence-supply lesson. Prompt review must check three separate states:

1. **Absent:** the required fact never reaches the model.
2. **Present but operationally weak:** the bytes arrive, but the fact is buried, ambiguous, unitless or presented unlike its peers.
3. **Canonical:** deterministic code supplies one authoritative labelled representation with unit, period and provenance.

The durable fix is model-agnostic: calculate net debt in code; give it its own canonical currency-formatted line; make “not provided” legal only when that field is absent; snapshot the rendered prompt; and freeze a semantic regression in which ratio-only output plus a false unavailability claim fails. The fixture should be rerun after material model/runtime changes because cross-model survival is evidence that the prompt interface is robust. Attribution remains shared: CMO/NZX are genuine Qwen3.6 misses, and their repeated shape is also evidence of a system-level representation defect.

The following nine-ticker production batch strengthened that conclusion and exposed a distinct semantic failure. GNE.NZ produced a third false-unavailability wording, showing why a validator built around sentence position or a list of known phrases was brittle. The replacement check searches the complete claim for declination language, but only treats it as a declination when the claim contains no monetary figure. NTL.NZ then described NZD 1,178,381 of gross cash as a net-cash position despite also citing NZD 93,133 of liabilities; the actual subtraction is NZD 1,085,248. This is not an evidence-absence failure. It is a gross-versus-net attribution failure that a dedicated detector still missed. Conversely, apparent SMI.NZ and RGI.NZ discrepancies proved correct after primary-source review: the reports chose dated filing figures over newer, undated yfinance snapshots. A semantic checker must therefore distinguish source date and authority as well as sign, scale and arithmetic.

### 13.6 Longitudinal production timing changed the strength, not the identity, of the speed claim

The repository had already been recording one elapsed-time row per ticker in `reports/data/run_times.csv`. The 358 M1/Qwen3.5-4B rows provide a broader observational baseline than the small cutover panel. Weekly mean runtime rose from 78.4 minutes per ticker in week 33 to 136.9 in week 39, a 1.64x increase over roughly seven weeks. That pattern is consistent with growing vault and context volume, but the log alone does not isolate causality: source volume, code, runtime and task mix also changed over the period.

Across all 358 M1 rows, mean runtime was 111.5 minutes per ticker. The first ten real M6/Qwen3.6 production runs averaged 21.4 minutes, an observed 5.2x end-to-end improvement. This is stronger production evidence than the original few-ticker cutover sample, but it remains a system-generation comparison rather than a hardware-only benchmark: the host, model architecture, prompt state, software and evidence volume changed together. The result supports the operational cutover decision and user-visible capacity claim; it does not allocate the 5.2x gain to the M6 chip alone.

### 13.7 A model refresh changed the weights and runner without changing the tag

The 10 October Ollama 0.35.1-to-0.40.2 upgrade included a refresh of the base model behind `qwen3.6-35b-a3b-80k`. Retained three-stage probes show that the same tag moved from GGUF/Q4_K_M (35.5B parameters, llama.cpp runner) to safetensors/NVFP4 (36.0B, MLX runner). The initial full MLX runs then slowed across tickers: runner-reported `held` memory rose from about 22.6 GiB to 30-31 GiB by the second ticker and 33-35 GiB during US tickers, with generation falling from about 40-50 tokens/s to roughly 25 and sometimes 12-19. The older Q4_K_M full run stayed near 40 tokens/s. The unchanged tag hid a material experimental treatment change; quantization, format, runner and runtime version must accompany the tag in future measurements.

Related behaviour is tracked upstream, not just inferred from our run. [Ollama issue #18131](https://github.com/ollama/ollama/issues/18131) documents how a fixed 8 GiB MLX prefix-cache budget can create memory pressure on a 32 GB Mac; [issue #18620](https://github.com/ollama/ollama/issues/18620) documents additional retained memory after tool-call requests in earlier builds. A [maintainer fix for the latter](https://github.com/ollama/ollama/pull/18955) merged on 12 October 2026, after the 0.40.2 installation used here. These reports give plausible mechanisms, but our telemetry does not partition its 30-35 GiB between the two. The later 0.40.4 retest in section 13.12 includes that fix: memory improved under between-agent unloading, but omitting that unload still produced a cliff. The merge does not retroactively change the earlier results or prove the whole problem is gone.

Releasing the MLX model at the start of each ticker costs about nine seconds of reload but restores the fast path for the tested NZ/ASX workload. A three-ticker sequence in the same order as an earlier no-release run measured:

| Ticker | Q4_K_M/llama.cpp | NVFP4/MLX, no release | NVFP4/MLX, release |
|---|---:|---:|---:|
| KPG.NZ | 38.3 tok/s; 20.8 min | 43.8; 20.0 min | 45.2; 18.9 min |
| CMO.NZ | 40.1; 16.6 min | 32.0; 20.3 min | 45.7; 17.3 min |
| NZX.NZ | 41.1; 16.1 min | 28.7; 28.3 min | 45.6; 15.7 min |

With release, runner-reported `held` memory restarted near 22.7 GiB per ticker and peaked at 27.0-28.2 GiB. A second live three-ticker comparison (KPG.NZ, SMI.NZ, IDX.AX) recorded 44.7-46.6 tokens/s for NVFP4/MLX against 38.4-41.7 for Q4_K_M/llama.cpp. These contemporaneous operational summaries indicate about 10-15% faster generation on the tested NZ/ASX workload with reloads; a sanitised call-level export is not yet in the public package. End-to-end minutes varied by ticker because generated output length and live inputs varied. Both builds traced every checked prose number to their respective inputs, but the MLX arm had one valuation contradiction and a gold-price/AISC mislabel that the Q4_K_M arm did not. Each ticker/build had only one live quality sample, and KPG received a new announcement between arms. That earlier live comparison did not establish answer-quality equivalence or an isolated causal speed effect; the later frozen repeats are analysed in section 13.10.

US tickers exposed a separate input-volume defect. In 200 logged Equibles calls, one `read_document_lines` result was 471,073 characters; prompt lengths reached 127-136K tokens against an 80K tag. A 12,000-character per-result cap, with explicit continuation instructions, reduced the largest observed result to 11,681 characters and QFIN's largest prompt to about 33K. An initial *live* before/after run used the cap and per-ticker release, but **not** per-task reload:

| Ticker | Earlier elapsed time | New elapsed time |
|---|---:|---:|
| QFIN | 43.9 min | 21.6 min |
| OGC | 45.1 min | 31.1 min |

This before/after changed multiple conditions and source interactions, so it does **not** isolate the cap's causal share. Memory still climbed within US tickers (QFIN about 31 GiB, OGC about 35 GiB), motivating an additional per-task reload as a removable workaround. The cap and a new near-context warning are model-independent protections; both GGUF's observed prompt truncation and MLX's observed acceptance of prompts above 80K could otherwise hide evidence loss or memory pressure. The next full-run result with both reloads is reported below. QFIN's initial new report used the supplied trailing P/E 1.48x and P/B 0.20x and computed CNY 3.26B net cash from CNY 6.81B cash less CNY 3.56B debt; prior reports had produced divergent valuation scores and USD net-cash claims. Its recommendation still varied across runs (Sell, then Hold, then Hold), so these corrected figures do not establish decision stability.

The operational lesson is to separate durable input-boundary fixes from runtime workarounds. Retain the Equibles cap and context warning; remove the reloads only after a later Ollama build demonstrates stable memory and speed without them. For quality, source presence is necessary but not sufficient: a number may be present in the inputs and still receive the wrong metric, period, unit or company label.

The first full-watchlist attempt with the cap and both reloads, reported in the internal run summary on 13 October, took 220.8 minutes versus about 424 for the preceding run. The summary records 44-47 tokens/s and 15-19 minutes for NZ/ASX tickers, 24-26 minutes for US tickers, maximum runner-reported `held` memory of 28.1 GiB, no context-pressure warnings, and a largest US prompt of 60K. These are **operational observations**, not a frozen before/after treatment comparison or a sanitised public call-level series. **One SEK.NZ report was rejected by the strict publication guard** because its model output omitted `RISK_CATEGORY`; this was not a complete set of publishable reports. A narrowly scoped, disclosed formula-derived insertion was subsequently added and tested, but was not part of that timing run. The observation supports sustained operation under the combined changes, not full report completion, each change's causal contribution, or answer-quality parity. The later frozen batches provide narrower controlled evidence (section 13.10).

### 13.8 Reproducibility is a runner property, not a seed setting

The batch design exposed two more ways a nominally frozen comparison can change beneath the experimenter. Evaluation mode originally read a previous report from the output folder that the preceding arm had just written, allowing treatment leakage. It also left some configuration, vault and telemetry paths live. The 13 October fixture work isolated outputs (including call logs) and made the previous report, vault and config files fixture-backed by default. Remaining live inputs were audited; Equibles needed a separate database freeze for the US end-to-end runs, which was implemented and checked as described in section 13.10. In those runs, **different tool choices are valid outcomes**, not grounds for rejection: the same canonical call should return stable content, while calls that differ between arms must be logged and evaluated as evidence-seeking behaviour. A prepared-evidence, tools-disabled model-answer test is a distinct experiment.

Seed passthrough was then tested against the actual Ollama 0.40.2 serving paths with the same prompt and reasoning enabled. Q4_K_M/llama.cpp produced byte-identical outputs twice at seed 42. NVFP4/MLX produced different outputs twice at that seed through both `/v1/chat/completions` and native `/api/chat`; temperature zero was only nearly repeatable. These are small checks of *this* runtime, not a universal claim that MLX can never honour seeds. They invalidate a single-seed repeatability assumption for this batch. Sample both builds across multiple runs/seeds, report per-field spread (including risk score and recommendation), and use a fixed GGUF seed only for exact debugging replay. Switching MLX to temperature zero would change the production treatment, not solve the comparison.

### 13.9 More context exposed an evidence-selection problem

The contemporaneous vault audit summary reports that 28 of 50 then-selected NZX/ASX/HKEX documents (56%) exceeded the old 10,000-character excerpt cap; among cut documents, the median excerpt retained 19% of the text. Its per-document manifest was not saved, although the method can be rerun against the historical vault revision, so those aggregate values remain **summary-only public evidence**. The missing term counts are a coverage proxy, not proof that each missing occurrence was material. The motivating KPG annual-report gearing figure was absent from the old excerpt despite being in the file. Adding `outlook` and `guidance` to the keyword list helped those terms but shifted other omissions within the same fixed budget: keyword tuning is not a free coverage gain.

A re-runnable cap sweep selected 20,000 characters as the smallest tested cap that restored KPG's needed gearing/covenant material. The 13 October checks compare this larger excerpt with an earlier 10K-cap sequence:

| Ticker | 10K cap: decode tok/s; peak GiB | 20K cap: decode tok/s; peak GiB |
|---|---|---|
| KPG.NZ | 45.2; 28.2 | 42.4; 29.0 |
| CMO.NZ | 45.7; 27.9 | 43.5; 28.2 |
| NZX.NZ | 45.6; 27.0 | 43.9; 27.3 |

These are production-shaped, not frozen, comparisons; the larger prompts cost time and reduced KPG memory headroom, but did not reproduce the earlier severe MLX throughput cliff. KPG's new report cited the annual report's covenant gearing 37.4% against a less-than-50% limit, information the old excerpt omitted. One recovered fact is not a general retrieval-quality score. This motivated the equal-budget section-retrieval A/B in section 13.11. That test improved an offline coverage proxy but did not improve the checked reports; a lower-cap memory test and held-out answerability questions remain open.

### 13.10 Frozen overnight batches: unload policy, evidence-seeking, and score variance

Two completed batches held the local fixture constant within each panel and ran three passes per arm. The [sanitised per-pass extract](evidence-and-provenance.md#e07--frozen-nzasx-and-us-batches-sanitised-extract) preserves the spread rather than choosing a best run. Both used Ollama 0.40.2 and an 80K model context. The server was configured for Q8_0 KV cache and flash attention. The llama.cpp runner logged both as applied; the MLX runner did not report its KV precision or flash-attention setting, so those settings are **unverified for NVFP4/MLX**. The NZ/ASX panel used KPG.NZ, SMI.NZ and IDX.AX; the US panel used QFIN and OGC with the same NVFP4/MLX build, a 12,000-character Equibles-result cap, and a paused Equibles ingestion worker. The two panels should not be merged into one estimate: their tickers, evidence paths, agent call counts and treatment arms differ.

The NZ/ASX panel compared these three setups:

| Setup | Build and unloading policy | Three-company pass times | Reports passing the guard | Checked wrong or invented statements |
|---|---|---|---:|---:|
| A | NVFP4/MLX, no unload | 76.2, 71.2, 72.5 min | 9 of 9 | 0 |
| B | Same NVFP4/MLX build, unload between tickers | 54.1, 54.3, 57.1 min | 9 of 9 | 0 |
| C | Q4_K_M/llama.cpp, unload between tickers | 60.0, 61.3, 61.2 min | 8 of 9 | 2 |

A and B held the weights and runner fixed; unloading policy was the main intervention. A's later tickers slowed as peak runner-reported `held` memory reached 31.5-31.8 GiB; B remained at 28.5-28.9 GiB or less and 45-48 *derived* decode tokens/s across tickers. C changed both quantization and runner, so B-versus-C is a serving-build comparison, **not** an isolated MLX-versus-llama.cpp effect. C's roughly 21.7 GiB memory figure is a load-time projection, not measured `held` memory comparable with MLX. All 278 Ollama requests in this panel were matched one-to-one to application calls by completion time within pass windows. The join script did **not** cross-check token counts or build per request, so the matching count is not independent identity proof. llama.cpp rates are native; MLX rates are derived from server-log phase timestamps and API token counts, not native prefill/decode counters.

The two checked errors in C were KPG's $99.99M AFFO value, which confused a source vote percentage with AFFO, and SMI's US$1,950/oz gold-price threshold, which relabelled an AUD/oz cost figure. B nevertheless had three completeness gaps versus A's zero, and one valuation contradiction; neither arm can be called categorically cleaner. The gaps were keyword-detected and then read by Claude; some such flags can reflect phrasing rather than genuinely absent discussion, although these three appeared to be omissions. Claude, an AI reviewer, scored the frozen reports against the predeclared rubric, assisted by automated figure tracing and source checks. Drew did not independently re-score them. These are nine reports per arm, not exhaustive factual audits or population error rates. The same frozen inputs gave KPG risk scores of 46, 69 and 58 within A; SMI's valuation applicability also flipped or contradicted itself. The result supports a speed/memory benefit from unloading and exposes decision instability, not a general quality winner.

For the US panel, each pass covered QFIN and OGC. The three setups were:

| Setup | Unloading policy | Pass times (minutes) | Peak `held` memory | Reports passing the guard | Checked wrong or mislabelled statements |
|---|---|---|---|---:|---|
| F | None | 75.5, 74.6, 76.0 | 31.9-32.8 GiB | 6 of 6 | 2 in one report; one began with a faulty Python input |
| G | Between tickers | 72.7, 81.6, 46.6 | 31.8-33.0 GiB | 5 of 6 | None confirmed |
| H | Between tickers and agents (production setting) | 51.1, 46.2, 46.4 | 27.2-27.7 GiB | 5 of 6 | 1 |

The within-ticker Equibles loop can reproduce the memory cliff before the next ticker begins; G's faster third pass, with fewer OGC calls, is important counter-variation. The H-versus-G difference is consistent with an operational benefit from between-agent unloading in this US workload, but agent call choices and output lengths also varied. Of 440 completed Ollama requests, 437 matched one-to-one; the other three were left unassigned, not silently attributed.

The US source freeze had two checks. With its ingestion worker stopped, before/after database counters changed only in scheduler bookkeeping tables, not filing/document/chunk tables. Across 589 Equibles calls with no reported errors, each of the 41 canonical calls repeated in the batch returned the same pre-cap SHA-256 payload. This supports source stability **for repeated identical calls**. It does not mean the arms saw identical evidence: they made 496 distinct canonical calls, only eight common to all three arms. Different searches and reads are legitimate end-to-end agent behavior and must remain visible in the result. The largest observed batch prompt was 66,083 tokens and no context-pressure warning fired; this supports the capped configuration, but because every arm had the cap, the batch does not isolate the cap's effect from an uncapped control.

Claude's single-reviewer scoring found three distinct problems. In setup F, a QFIN report called consolidated trust payables off-balance-sheet, contrary to the 20-F. The same report also rendered dividend cover as 1.02x after Python supplied `FCF Dividend Cover: 102.15`, calculated by dividing CNY free cash flow by USD dividends without converting currency. That second failure began upstream and included a model misreading; it was **not** an independently invented financial fact. Every US setup received the faulty frozen QFIN input, so those historical cover figures cannot be treated as valid. The live feeder was subsequently fixed (`f7d7164`); the frozen fixture was not rewritten. In setup H, an OGC report labelled about $3,000/oz as breakeven when the received AISC was $2,151/oz. Two further QFIN figures remain unverified and were not scored.

The rejected reports were QFIN in G's first pass and QFIN in H's third pass. Across reports that passed the guard, QFIN valuation scores ranged from 28 to 83 under the same frozen *source state*. One H report recommended Sell while the others said Hold. Because agents chose different tool calls, this range cannot be attributed solely to stochastic responses to identical final prompts. These small, single-reader samples do not establish that one unloading policy is more accurate or that unloading caused the Sell outlier.

The combined lesson is deliberately narrower than “MLX is better”: memory-reset policy has a repeatable operational payoff under these tested loads, while the answer-quality differences and score spread require further diagnosis. Publication guards prevented two malformed US reports and one malformed NZ/ASX report from replacing prior reports, but guard passage is not semantic correctness. The source-currency bug, since fixed in the live feeder, shows why frozen inputs are necessary for fair comparisons yet insufficient for factual validity: all arms can receive the *same wrong fact*. These findings keep the current production serving choice provisional on quality and Python-side source validation a separate priority.

### 13.11 A better vault coverage proxy did not improve the checked reports

The next frozen A/B compared the production keyword excerpter (K) with per-agent BM25 section retrieval (S). Both used the same 20,000-character excerpt budget, the night-1 KPG.NZ/SMI.NZ/IDX.AX fixture (`9ae0ab022b83d129`), production NVFP4/MLX build (`f52f2b9a2363`), and between-ticker unloading. Three passes per arm were interleaved. Vault-call hashes confirmed that K supplied identical excerpts to its four vault-reading agents, while S supplied distinct per-agent excerpts. This tested selection at an equal budget, not a smaller-cap memory intervention.

The offline whole-watchlist audit found 47 probe terms lost with S versus 60 with K, in roughly 10-17% less excerpt text. S reached KPG's 33.3% pro forma gearing paragraph at 10K and 15K characters; K needed 20K. Those are coverage observations, not answer-quality scores.

| Arm | Reports passing guard | Checked wrong/mislabelled statements | Three-company pass times | Peak runner-reported `held` |
|---|---:|---:|---|---|
| K: keyword excerpts | 9 of 9 | 0 | 58.2, 57.3, 53.6 min | Up to 29.8 GiB |
| S: per-agent BM25 sections | 9 of 9 | 2 | 58.7, 54.9, 53.5 min | Up to 29.7 GiB |

S's checked errors were SMI's NZ$3,800/oz study price labelled US dollars and KPG's roughly $0.90 share price paired with a $1.06 analyst target as though both were a current trading range. S also correctly separated IDX's AUD 545.8 million net debt from its AUD 298.9 million pre-AASB 16 covenant metric. Neither arm cited KPG's 33.3% pro forma gearing; all six KPG reports cited the current 37.4% figure supplied by the deterministic REIT facts block. All 183 Ollama requests joined one-to-one with call-log rows by the batch method, subject to the same identity-check limits noted for earlier joins. [The per-pass extract is in E08](evidence-and-provenance.md#e08--frozen-vault-section-retrieval-ab-extract).

Claude scored the reports as one AI reviewer, assisted by an automated figure trace and checks against the frozen sources; Drew did not independently re-score them. Keyword completeness flags were read, and in this batch they reflected phrasing rather than confirmed missing discussion. The evaluation fixture omitted the mining scenario block used in production, equally in both arms. With one fixture, three tickers and nine reports per arm, the two S mislabels are within the variation seen in earlier batches and do not establish that BM25 is generally less accurate. Equally, the improved coverage proxy did **not** establish a report-level benefit. A plausible mechanism is that Python already supplied the decisive facts in the checked cases, but the A/B did not ablate those fact blocks and cannot prove their causal share. Section retrieval remains opt-in and off in production. A lower-cap S arm could separately test memory and prompt-size savings while holding answer quality to the same gate.

### 13.12 Ollama 0.40.4 reduced memory under unloading but did not eliminate the cliff

Ollama 0.40.4 includes [the merged MLX tool-call recurrent-state fix](https://github.com/ollama/ollama/pull/18955). A fresh QFIN/OGC fixture was required because Equibles had ingested new filings since the 0.40.2 US batch. Thus the earlier and later **absolute** times and memory peaks are observational across different source states, not a paired software-version A/B. Within the new 0.40.4 batch, the NVFP4/MLX build remained `f52f2b9a2363`; both arms used the 12K-character Equibles cap and between-ticker unloading. Three interleaved passes compared G (no between-agent unload) with H (between-agent unload). The server environment requested Q8_0 KV cache and flash attention; as before, MLX did not verify effective KV precision or flash-attention application.

| Arm | Pass times, two tickers | Peak runner `held` per pass | Reports passing guard |
|---|---|---|---:|
| G: between-ticker unload only | 62.2, 89.7, 49.1 min | 31.68, 32.39, 30.76 GiB | 5 of 6 |
| H: between-ticker and between-agent unload | 51.1, 50.4, 40.3 min | 26.10, 26.17, 25.88 GiB | 6 of 6 |

QFIN G pass 2 took 57.4 minutes with a derived 20.5 decode tokens/s, versus 18.2-28.7 minutes and 43.9-49.5 tokens/s for H's three QFIN passes. Different agent tool choices and output lengths contributed to elapsed time; this is a whole-pipeline treatment comparison, not identical-prompt decode benchmarking. H's 24.0-26.2 GiB ticker peaks were lower than the prior 0.40.2 H range of 25.3-27.7 GiB, but the new fixture does not isolate the upgrade's causal saving. G still reached 30.6-32.4 GiB. The between-agent unload remains in production; the between-ticker unload was not removed or separately retested. Of 299 server requests, 291 joined one-to-one to call records and eight were left unassigned. The Equibles worker was stopped; the database before/after diff affected only scheduler bookkeeping, and all 29 repeated canonical calls among 375 error-free calls returned byte-identical SHA-256 payloads. [E09 preserves the per-pass extract and limits](evidence-and-provenance.md#e09--ollama-0404-us-unload-retest).

This batch was **not semantically scored**. Passing the publication guard is not evidence of financial correctness. G pass 3 QFIN carried a risk-score/prose mismatch banner. G pass 2 OGC was rejected after leaking a thinking tag and process narration and duplicating FACTS markers. That is the third observed leaked-thinking case, following NTL on 9 October and SMI on 14 October, and the first on 0.40.4. Memory pressure accompanied this occurrence but not the SMI case, so it cannot be the sole explanation. The guard retained the last good report in the report and dashboard paths; it does not cure the generation failure. Investigation is open.

### 13.13 A faster reasoning MoE failed the final-output contract, and a schema exposed a different error

A separate Ollama 0.40.4 trial compared Liquid AI's LFM2.5-24B-A2B Q4_K_M with the unchanged production Qwen3.6-35B-A3B NVFP4 tag on the same frozen KPG.NZ, SMI.NZ and IDX.AX fixture. The candidate's guard rejected five of twelve ticker attempts, counting its three-ticker gate plus three passes; production passed all nine comparison attempts and had no matching gate arm. Candidate decode rates were about 63-69 tokens/s versus 47-49 for production, but candidate completion totals were roughly 75K-100K tokens per ticker versus 40K-55K. The faster token rate was largely offset by more generation, consistent with LFM's [reasoning-only design](https://www.liquid.ai/blog/lfm2-5-24b-a2b); the trial does not isolate hidden reasoning as the sole cause. The candidate's approximately 15-17 GB screen/projection and production's 28-30 GiB runner `held` values use different memory measures and should not be read as a precisely paired saving. [E10 gives the guard counts and measurement limits](evidence-and-provenance.md#e10--a-complete-schema-still-changed-a-supplied-figure).

Raw rejected output showed genuinely absent required fields or collapsed FACTS/FINAL blocks, not merely an over-strict spelling rule. All nine candidate final-task calls in the three comparison passes had `finish_reason=stop`, with roughly 16K-18K prompt tokens under an 80K configured context and no per-call `max_tokens` value recorded. That makes output-ceiling truncation an unlikely explanation for these omissions; reducing `num_predict` or adding a stop sequence would not restore a field the model never emitted.

A narrowly scoped follow-up tried [Ollama's JSON-schema-constrained output](https://docs.ollama.com/capabilities/structured-outputs) for the KPG final decision that had previously failed. The first probe accidentally sent only the task description, omitting CrewAI's separately appended upstream task outputs; its result was discarded. The corrected probe reconstructed and checked the full context before calling the model. It returned a complete, typed object. Five of six checked carried-forward FACTS values matched the upstream material, but the position-headroom field substituted a different amount for one supplied explicitly and repeatedly, with an instruction not to recompute it. In a proportional, public-safe normalisation, source **100.00** became output **84.88**; these are not the actual portfolio figures. This is **one corrected final-call probe**, not evidence that schema enforcement would remove the trial's five guard rejections across repeated full crews. Nor was the complete object a semantically correct report.

The result sharpens the architecture boundary: schema validation can require fields and types, but must not be treated as source validation. Known quantities should be carried or checked by deterministic code, with Python rendering the report markers after validation if a typed decision boundary is adopted. Simply setting CrewAI 1.15.22 `Task(output_pydantic=...)` would not by itself reproduce this constrained-decoding probe: the installed task path can convert generated prose to a model in a subsequent LLM call. Such a conversion could fill an omitted field without proving the original final answer was grounded. No production interface was changed on the strength of this single probe.

## 14. A practical evaluation checklist

For teams investigating a slow local reasoning pipeline:

1. **Define the decision-quality gate first.** Pick real failure cases before testing a speed treatment.
2. **Inventory the complete evidence path.** Separate missing source data, retrieval failures, deterministic parsing, model reasoning, and validation.
3. **Record native phase timings.** Do not label wall-time-per-token as decode speed.
4. **Control prefix reuse.** Add nonces or otherwise prove the cache behavior you intend to measure.
5. **Record effective runtime state.** Verify the server, model tag, cache type, context allocation, memory ceiling, and actual placement.
6. **Measure total sequence behavior.** Long prompts and long generations both affect later decode.
7. **Pair treatments on identical evidence.** Live retrieval is not suitable for a causal full-versus-compact comparison.
8. **Freeze answer keys before outputs.** Include known tempting wrong answers, not just the correct result.
9. **Separate speed, capacity, and quality claims.** A fast resident configuration can still truncate evidence or make worse decisions.
10. **Version-condition historical evidence.** Deterministic code changes can materially alter prompt distributions.
11. **Retain corrections.** A report that shows how a claim narrowed is more useful than one that invents a straight path.
12. **Keep raw artifacts near the code.** Summary prose should point to runs, commits, fixtures, and reproduction commands.
13. **Read memory size with placement.** An allocation jump at the CPU/GPU boundary is not automatically KV-cache growth.
14. **Separate weight precision from cache precision.** They affect different stored values and require different A/B tests.
15. **Test reproducibility instead of assuming a seed supplies it.** Keep frozen inputs and output paths isolated; measure each runner's run-to-run spread.
16. **Measure evidence coverage under any excerpt cap.** A larger context window does not restore a passage the retrieval layer never selected.
17. **Audit representation parity.** Equally important facts need equally explicit labels, units, dates and provenance; rerun the contract after model changes.
18. **Treat longitudinal logs as observational evidence.** They can strengthen an operational result without isolating which changed component caused it.
19. **Record host power state.** A detached job can remain alive while macOS sleeps; reconcile power events before attributing silent wall-time gaps to a model.

## 15. Evidence and audit trail

The repository has been operated through a TODO-and-archive workflow rather than a retrospective lab notebook. Completed work is moved into an archive rather than deleted. That gives the authors a useful internal audit path, although it does not guarantee that every command or observation from one sitting was recorded in full detail.

The [evidence appendix](evidence-and-provenance.md) separates numerical rows reproduced from retained records from claims supported only by contemporaneous summaries. The latter are labelled as such and are not independently reproducible from this document set.

| Evidence layer | What it contributes |
|---|---|
| Active TODO | Current hypotheses, experiments, corrections, decisions, and open work |
| Completed-work archive | Archived outcomes and historical rationale |
| Reviewer discussion log | Claude-Codex objections, corrections, and agreed evidence boundaries |
| Git history | Immutable chronology of implementation, documentation, and corrected claims |
| Internal probe records | Retained raw context-sweep and long-generation records; publishable extracts are in the [evidence appendix](evidence-and-provenance.md) |
| Internal probe harness | Experiment and captured-field definitions; raw source is not required to read this report |
| Tests and frozen fixtures | Regression and decision-quality evidence |
| Generated reports and call logs | Production-shaped observations and failure cases |

No single audit layer is complete. The claim map below identifies each result's evidence status and limitation.

### Initial claim map

| Claim | Status | Primary retained evidence | Remaining work |
|---|---|---|---|
| Reasoning dominates the short QFIN probe | Contemporaneous summary only | Original September probe table; raw responses not retained | Do not present as an independently inspectable trial |
| Reasoning-off failed the QFIN comparison | Recorded case outcome | Summary table and checkable dividend operands | Repeat with saved responses if promoted beyond case evidence |
| Q8_0 decode slows sharply with sequence length | Replicated | [Sanitised sweep table](evidence-and-provenance.md#e01--m1-total-sequence-slowdown-extract), run `f61f8a5ef385` plus three prior sweeps | Full raw record remains private |
| Generated tokens follow the same slowdown curve | Measured internally | Internal long-generation run `3410b873627f` | Export window-by-window evidence or keep provisional in the public version |
| Compaction was not authorized for the tested portfolio task | Rejected | Runs `3eb2cc2ade17`, `66621dc49982`, and direct control `1e01eec2cb14`; 85% compact quality and 6.3% wall reduction | Retain as negative result; no five-repeat phase |
| F16 retained substantially more decode throughput than Q8_0 on the M1 path | Replicated but bounded | Discovery F16 run `64af12ed846d` and server-provenanced replication `da0c2c751e71` | Publish extract with the later M6 non-replication beside it |
| M6 showed no material F16 advantage | Measured | Retained M6 Q8_0/F16 context sweeps for 4B and 27B | Repeat only if a runtime upgrade creates a new treatment |
| M1 KPG effective throughput improved 1.74x median under smaller F16 context | Historical corroboration | Call log plus historical task medians; independently recomputed | Do not generalize to M6 |
| 35,840/F16 is not licensed for general production | Decision from measured constraints | Residency observations, post-`dad9636` distribution, reserve analysis | Revisit only after broader deterministic evidence packets |
| M6 64K/Q8_0 dense-model screen: 4B 43.8, 9B 27.9, 27B 9.1 decode tok/s | Measured | [Sanitised screen table](evidence-and-provenance.md#e02--m6-dense-and-moe-screen-extract) | Full raw record remains private |
| Dense 27B is unsuitable at the tested configuration | Screen decision | KPG internal time 168.6 min, checker output, frozen report | Keep one-run/one-ticker boundary explicit |
| Pre-upgrade standard tags were at least as fast as the then-tested MLX tags | Measured for that treatment, superseded for current MoE operations | Internal M6 MLX baseline screen | Read with section 13.7, not as a current MLX verdict |
| NVFP4/MLX ticker unloading removed the NZ/ASX memory cliff | Frozen repeated operational result; quality ranking unresolved | [Three-pass NZ/ASX extract](evidence-and-provenance.md#e07--frozen-nzasx-and-us-batches-sanitised-extract) | Nine reports per arm, scored by Claude without independent human re-scoring |
| Between-agent unloading bounded US memory growth | Frozen repeated operational result; quality ranking unresolved | [Three-pass US extract and freeze witness](evidence-and-provenance.md#e07--frozen-nzasx-and-us-batches-sanitised-extract) | Do not attribute the separate Equibles cap's benefit to this comparison |
| QFIN dividend-cover input mixed CNY and USD | Code-and-fixture diagnosis, not a model-only defect | US scoring note and section 13.10 case extract | Live feeder fixed in `f7d7164` with regression tests; frozen fixture retains the old input |
| Equibles output cap removed observed US prompt oversizing | Live before/after, multi-change runtime comparison | 200-call tool-output audit and 2026-10-12 QFIN/OGC logs, summarized in TODO.md | Keep cap; separately evaluate US per-task reload and answer quality |
| First full-watchlist cap-plus-both-reloads attempt stayed below observed memory cliff | Internal operational summary; one report rejected and several changes combined | 2026-10-12/13 run notes report 220.8 minutes | Publish call-level extract; use frozen batches for causal shares and report completion |
| MLX seed did not reproduce exact output in a same-prompt probe | Measured for Ollama 0.40.2, small probe | 2026-10-13 matched-prompt dual-route checks in DISCUSSION_LOG.md and TODO.md | Sample both runners across repeats; do not claim universal MLX nondeterminism |
| Vault excerpts omitted material source text at 10K cap | Internal aggregate summary and KPG case | Historical cap-audit method; no saved per-document manifest | Historical manifest remains unavailable; [equal-budget section-retrieval A/B](evidence-and-provenance.md#e08--frozen-vault-section-retrieval-ab-extract) did not improve checked reports |
| Qwen3.6 MoE bypassed the tested dense speed/capacity trade-off | Measured | 64K and 80K residency screens; frozen three-ticker reports and run times | Continue primary-source review of production figures |
| GPT-OSS failed the financial quality gate despite compatible tools | Rejected | Frozen KPG report and three successful vault calls | Preserve as negative control; no wider panel |
| Old 4B production inverted KPG AFFO direction while preserving magnitude | Primary-source-verified case evidence | Archived M1 report, new M6 report, KPG FY26 announcement | Repeat the three-way comparison for later production figures |
| Evidence representation produced model-dependent net-debt failures | Repeated production case evidence | Archived 4B reports, Qwen3.6 CMO/NZX/GNE reports, rendered prompt blocks and regression tests | Recheck the canonical interface after model/runtime swaps |
| First ten M6/Qwen3.6 production runs averaged 5.2x faster than 358 M1/Qwen3.5-4B rows | Observational production evidence | `reports/data/run_times.csv`; weekly M1 aggregation and M6 cutover rows | Publish extraction method; do not attribute the whole difference to hardware |
| Gross-versus-net and dated-source attribution remain distinct failure modes | Primary-source-checked case evidence | NTL defect plus verified SMI/RGI filing figures from the nine-ticker batch | Investigate the missed NTL detector and retain period/provenance in facts |
| Host sleep likely interrupted the 27B wrapper interval | Contemporaneous exact-window power-log summary; filtered extract not retained | Internal 168.6 minutes versus outer 428.5 minutes; later observed detached-job sleep | Do not publish a precise transition count without the source; cumulative asleep time was not reconciled |

## 16. Related work and what this project adds

Prior work already establishes the main individual mechanisms:

- autoregressive KV cache grows with sequence length and consumes memory bandwidth;
- long context can reduce model accuracy even when the required fact is retrievable;
- prompt compression can improve latency and sometimes quality;
- information position within long context matters;
- KV-cache compression changes memory and speed trade-offs; and
- Apple-Silicon inference runtimes differ materially.

The working references below identify the sources used to frame these mechanisms; none independently verifies this project's measurements.

The contribution of this project is the integrated case:

- production telemetry led to a reasoning-token hypothesis;
- a real financial question rejected the obvious speed switch;
- native timings corrected a mixed metric;
- nonce controls exposed prefix-cache contamination;
- long generation connected input context and output growth;
- cache precision overturned an overly broad M1 causal explanation, then M6 replication bounded the correction itself;
- runtime residency caught ineffective configuration changes;
- deterministic parsing changed the capacity distribution;
- frozen evidence separated data drift from prompt treatment;
- complete output-contract auditing exposed an unanswerable fixture and a fact dropped by compaction;
- bounded failures distinguished a reasoning loop from a context-capacity failure;
- model-size screening showed that full residency and more parameters did not guarantee usable latency or better checked facts;
- separate internal and wrapper clocks exposed a host-sleep explanation, recorded in an exact-window check whose filtered output was not retained; and
- every optimization remained subordinate to decision quality.

### Working references

- Behnam, P., Fu, Y., Zhao, R., Tsai, P.-A., Yu, Z., and Tumanov, A. (2025). [*RocketKV: Accelerating Long-Context LLM Inference via Two-Stage KV Cache Compression*](https://arxiv.org/abs/2502.14051). ICML 2025.
- Du, Y., Tian, M., Ronanki, S., Rongali, S. B., Bodapati, S. B., Galstyan, A., Wells, A., Schwartz, R., Huerta, E. A., and Peng, H. (2025). [*Context Length Alone Hurts LLM Performance Despite Perfect Retrieval*](https://aclanthology.org/2025.findings-emnlp.1264/). Findings of EMNLP 2025.
- Fioravanti, I. (2026). [*LLM Context Benchmarks*](https://github.com/ivanfioravanti/llm_context_benchmarks). Open-source benchmark harness.
- Jiang, H., Wu, Q., Luo, X., Li, D., Lin, C.-Y., Yang, Y., and Qiu, L. (2023). [*LongLLMLingua: Accelerating and Enhancing LLMs in Long Context Scenarios via Prompt Compression*](https://arxiv.org/abs/2310.06839).
- Kummer, C., Jurkschat, L., Farber, M., and Vahdati, S. (2026). [*Prompt Compression in the Wild: Measuring Latency, Rate Adherence, and Quality for Faster LLM Inference*](https://arxiv.org/abs/2604.02985). ECIR 2026.
- Liu, N. F., Lin, K., Hewitt, J., Paranjape, A., Bevilacqua, M., Petroni, F., and Liang, P. (2024). [*Lost in the Middle: How Language Models Use Long Contexts*](https://arxiv.org/abs/2307.03172). TACL.
- Qwen Team. (2025). [*Qwen3: Think Deeper, Act Faster*](https://qwenlm.github.io/blog/qwen3/).
- Qwen Team. (2026). [*Qwen3.6-35B-A3B model card*](https://huggingface.co/Qwen/Qwen3.6-35B-A3B).
- OpenAI. (2025). [*Introducing gpt-oss*](https://openai.com/index/introducing-gpt-oss/).
- Apple. (2026). [*Mac mini (M6 or M5 Pro) technical specifications*](https://support.apple.com/en-us/128108).
- Rajesh, V., Jodhpurkar, O., Anbuselvan, P., Singh, M., Jallepali, A., Godbole, S., Sharma, P. K., and Shrivastava, H. (2025). [*Production-Grade Local LLM Inference on Apple Silicon: A Comparative Study of MLX, MLC-LLM, Ollama, llama.cpp, and PyTorch MPS*](https://arxiv.org/abs/2511.05502).
- oMLX community. (2026). [*Qwen3.5-4B on M1 Max: performance by context length*](https://omlx.ai/benchmarks/performance/ex0p90rs).
- jyxjjj and Gerganov, G. (2026). Later comments in llama.cpp discussion #12985: [*Qwen3.6-27B Q8_0 long-context Metal benchmark*](https://github.com/ggml-org/llama.cpp/discussions/12985#discussioncomment-18063341); [*maintainer warning about quantized KV cache on Metal*](https://github.com/ggml-org/llama.cpp/discussions/12985#discussioncomment-18063447); and [*partial F16-cache follow-up*](https://github.com/ggml-org/llama.cpp/discussions/12985#discussioncomment-18063832).

## 17. Open research questions

The new-M6 primary-source review, evidence-packet selection, `num_batch`, and higher-precision weights are follow-up research, not prerequisites for this bounded account. The frozen NZ/ASX and US batches have now completed and their sanitised per-pass results are in [E07](evidence-and-provenance.md#e07--frozen-nzasx-and-us-batches-sanitised-extract). Different US tool calls chosen by the arms are reported as part of the end-to-end outcome, not forced into identical call scripts. The first publication can state a bounded memory/speed result. General answer-quality equivalence remains unresolved; the live currency feeder has been corrected, but the original frozen input remains part of the historical evidence. Whether MLX applies Q8_0 KV precision and flash attention under the tested server configuration is also unverified.

## 18. Limitations

- This is one application across one M1 and one M6 host, with runtime versions changing between hardware generations.
- Several experiments began as operational debugging rather than pre-registered research.
- The first three context sweeps predated complete environment capture.
- The replicated M1 F16 evidence and the M6 non-replication used different hardware/runtime generations; each is valid for its treatment, but neither isolates which implementation change explains the difference.
- Rounded residency observations were not all retained as primary artifacts.
- The production-shaped KPG comparison is unpaired and historically controlled.
- Financial quality cases are deliberately failure-mode-weighted and are not a general benchmark.
- The internal audit trail is not a command-by-command laboratory notebook.
- Generated reasoning is assessed through runtime counters and exposed native fields; invisible internal computation cannot be independently audited in full.
- The one-task compaction result failed its gate and cannot establish how another task, ticker, model or prompt treatment would behave.
- The dense-model KPG elimination screen used one run per model; the later 4B/9B and Qwen3.6 panels widened ticker coverage but still used one stochastic report per model/ticker.
- The 27B outer-wrapper interval exceeds the internal ticker timer by approximately 260 unexplained minutes.
- Qwen3.6's production evidence is still young. The two new three-pass frozen panels widen coverage but were scored by Claude, an AI reviewer, without independent human re-scoring; they do not establish a general error rate or answer-quality equivalence between serving builds.

## Provisional conclusion

The biggest performance lever was not obvious from visible report length, and the fastest configuration was not automatically the safest. Reasoning tokens dominated generation in important calls, but reasoning mode did not have one universal effect: disabling it broke a short QFIN financial comparison, while forcing it on a dense KPG decision contract produced thousands of hidden tokens and no answer. Long sequences reduced decode throughput under the deployed M1 Q8_0 cache. Replicated M1 evidence showed F16 could greatly flatten that decline on one runtime path, but the completed M6 llama.cpp A/B detected no material F16 advantage and restored Q8_0 as the memory-efficient choice for that path. The current MLX runner's applied KV precision remains unverified. The direct compaction gate rejected another plausible shortcut: a 26.5% smaller input produced only a 6.3% wall-time saving and failed the quality threshold.

The M6 then exposed a second performance boundary. All three dense candidates fit fully on GPU at 64K, yet 4B, 9B and 27B decoded at 43.8, 27.9 and 9.1 tokens/s. The one-ticker screen found no checked quality advantage for 27B, and the wider dense panel established 9B as the better dense baseline. Qwen3.6-35B-A3B then changed the trade-off rather than merely moving along it: 35B total capacity, approximately 3B active parameters, 72.3 tokens/s at 64K, clean tool calls, three correct frozen financial checks and roughly one quarter of 9B's panel time. After an 80K residency confirmation, it became the production model.

The first production diff also reinforced the central warning. The old model preserved KPG's AFFO percentage and dollar value but inverted “increased” into “contracts”; the new model matched the primary source. A nearby number is not necessarily a correct fact. The durable result is therefore not “the larger model is now trustworthy.” It is that runtime, architecture, prompt contract and semantic factual checks must remain one evidence-gated system even after a successful model upgrade.

The later Ollama refresh made that lesson operational. Under an unchanged model name, both the weights and runner changed. Two live NZ/ASX panels first suggested an MLX speed benefit with reloads; the completed frozen A/B then showed that unloading between tickers removes a substantial memory/time cliff under that same build. The US F/G/H repeats showed that a long tool-using ticker may need a reset between agents as well. Oversized US tool responses required a separate upstream cap. MLX still has operational liabilities: memory accumulation without resets, non-repeatability under the tested seed, and a different response to overlong prompts. The repeated quality scores and report outcomes vary even on controlled inputs, while a newly found cross-currency Python calculation supplied every US arm a wrong fact. Serving identity, data validity, memory behaviour and semantic output remain separate gates; these panels do not establish a general quality winner.

The durable result is methodological:

> Measure the engine precisely, freeze the evidence, test the decision, and preserve every correction that changes the claim.

That is the difference between optimizing a benchmark and improving the system that people actually rely on.

<!--
PAIRED REPORT MAINTENANCE CHECK
Before treating a change to this report as complete, review
from-faster-answers-to-better-decisions.md for the same:
- headline measurements and corrected measurements;
- measured/calculated/corroborative/planned status;
- production configuration and current decision;
- quality-gate outcome;
- limitations and open work; and
- model-comparison and missing-evidence framing.
Update both Last paired review dates together.
-->
