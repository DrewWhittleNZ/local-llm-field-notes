# Changes to the field notes

This lists substantive changes to the published account, not every spelling or layout edit. The reports are an ongoing field study; new evidence can narrow an earlier claim without making the earlier measurement disappear.

## 17 October 2026 — LFM trial and structured-output probe

- Added a frozen three-company LFM2.5-versus-production comparison and a corrected, single-call schema probe. The schema filled every required field in that probe but changed one explicitly supplied, portfolio-specific figure; the public extract uses proportionally scaled numbers. See the [plain-language case](from-faster-answers-to-better-decisions.md#a-form-with-every-box-filled-can-still-change-your-number), [technical account](from-tokens-per-second-to-decision-quality.md#1313-a-faster-reasoning-moe-failed-the-final-output-contract-and-a-schema-exposed-a-different-error), and [E10 evidence limits](evidence-and-provenance.md#e10--a-complete-schema-still-changed-a-supplied-figure).
- Recorded that the first schema probe omitted upstream reports and was discarded. Neither the corrected single call nor the candidate's faster per-token decode establishes an end-to-end quality or speed win. The production model and report interface were not changed.

## 16 October 2026 — Ollama 0.40.4 US retest

- Added the fresh-fixture, three-pass G/H comparison on Ollama 0.40.4. Between-agent unloading still reduced memory and elapsed time within that fixture, so the workaround remains. The upgrade's standalone effect cannot be quantified against the older fixture. See the [plain-language account](from-faster-answers-to-better-decisions.md#the-newer-ollama-still-needed-the-memory-workaround), [technical analysis](from-tokens-per-second-to-decision-quality.md#1312-ollama-0404-reduced-memory-under-unloading-but-did-not-eliminate-the-cliff), and [E09 measurements](evidence-and-provenance.md#e09--ollama-0404-us-unload-retest).
- Recorded a third leaked-thinking rejection, the first observed on 0.40.4. The guard preserved the previous good report; the generation problem remains open. This batch was not semantically scored, so publication counts are not an answer-quality comparison.

## 15 October 2026 — Vault section-retrieval experiment

- Added a frozen, three-pass comparison of keyword excerpts and per-agent BM25 section retrieval, using the same 20K-character budget and saved company inputs. See the [plain-language explanation](from-faster-answers-to-better-decisions.md#giving-the-ai-more-room-does-not-guarantee-it-sees-the-right-page), [technical analysis](from-tokens-per-second-to-decision-quality.md#1311-a-better-vault-coverage-proxy-did-not-improve-the-checked-reports), and [E08 measurements](evidence-and-provenance.md#e08--frozen-vault-section-retrieval-ab-extract).
- Narrowed the retrieval claim: BM25 improved an offline passage-coverage proxy, but did not improve the checked reports in this small equal-budget test. It remains off in production. A lower-cap memory test is a separate open question.

## 15 October 2026 — First public version

- Published the plain-language report, technical report, and sanitised evidence appendix together. They distinguish directly reproduced numbers from internal summaries and preserve known negative results, including the failed context-compaction gate.
- The analyzer code, private financial records, full prompts, raw logs, and working history were not published with these notes.
