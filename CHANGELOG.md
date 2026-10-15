# Changes to the field notes

This lists substantive changes to the published account, not every spelling or layout edit. The reports are an ongoing field study; new evidence can narrow an earlier claim without making the earlier measurement disappear.

## 15 October 2026 — Vault section-retrieval experiment

- Added a frozen, three-pass comparison of keyword excerpts and per-agent BM25 section retrieval, using the same 20K-character budget and saved company inputs. See the [plain-language explanation](from-faster-answers-to-better-decisions.md#giving-the-ai-more-room-does-not-guarantee-it-sees-the-right-page), [technical analysis](from-tokens-per-second-to-decision-quality.md#1311-a-better-vault-coverage-proxy-did-not-improve-the-checked-reports), and [E08 measurements](evidence-and-provenance.md#e08--frozen-vault-section-retrieval-ab-extract).
- Narrowed the retrieval claim: BM25 improved an offline passage-coverage proxy, but did not improve the checked reports in this small equal-budget test. It remains off in production. A lower-cap memory test is a separate open question.

## 15 October 2026 — First public version

- Published the plain-language report, technical report, and sanitised evidence appendix together. They distinguish directly reproduced numbers from internal summaries and preserve known negative results, including the failed context-compaction gate.
- The analyzer code, private financial records, full prompts, raw logs, and working history were not published with these notes.
