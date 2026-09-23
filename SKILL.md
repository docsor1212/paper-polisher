---
name: paper-polisher
version: 3.6.0
author: DoctorQ Lab
description: >
  AI writing detection (AI-rate self-check for authors), academic polishing
  guidance (style, terminology, translation-smell),
  metaphor audit, quality report, AIGC compliance label check (China 2025-09
  labeling rules), paragraph-level attribution, journal precheck. Bilingual
  CN/EN, 100% local, zero upload, zero credentials. v3: 11-layer recalibrated
  rule engine + token-spectrum layer + length-routed fusion + optional
  supervised Qwen3-0.6B ONNX layer (AUROC 1.0 on held-out test) + LLM
  fingerprint attribution (GLM / DeepSeek / Qwen / Kimi / MiniMax / GPT /
  Claude / Gemini) + freshness pipeline. Base-engine numbers reproduce from the
  bundled held-out evaluation; supervised-layer columns are author-side held-out
  measurements (the model itself is not bundled).
tags: [ai-detection, deai, academic-writing, paraphrase, paper-polish]
---

# Paper Polisher Pro v3

AI writing detection (AI-rate self-check for authors) · academic polishing guidance · terminology standardization · translation-smell check · quality report · AIGC compliance label check · paragraph-level attribution · journal precheck.
100% local, zero upload, zero credentials, pure standard library (optional onnxruntime enhancement layer).

> ## ⛔ Iron laws
> 1. **Only reproducible numbers.** Every metric comes from the held-out (test split) evaluation in `eval/run_eval.py`; unsupported claims like "100% detection rate / F1 98.3%" from older docs have been removed.
> 2. **No verdict on short text.** Texts under 100 characters get `risk=unknown` (community lesson: short-text false positives are uncontrollable).
> 3. **Fingerprints attribute, never score.** (Measured 2026-08-15: injecting fingerprints into the detector doubled human false positives.)
> 4. **Calibration/evaluation separation.** Spectrum, weights and thresholds are built on the calib half only; the test half is reserved for final evaluation (an in-sample AUROC of 0.9972 collapsed to a real 0.9187 once split).

## Academic integrity

This tool is for **authors self-reviewing and improving their own writing quality** — clearer sentences, consistent terminology, natural style. It is not designed to evade institutional AI-detection systems, and it must not be used to misrepresent AI-generated work as human-written. Follow your institution's AI-use and disclosure policies; the bundled `aigc_label_check.py` exists to help you **comply** with disclosure and labeling rules (e.g., China's 2025-09 labeling measures) — to declare AI assistance properly, not to hide it. Every report carries an explicit `integrity_notice` to this effect.

## Measured performance (C-ReD + DetectRL-ZH, held-out test half, n=5,251)

| Metric | v2.0 baseline | v3.0 rules+spectrum | v3.1 +supervised | **v3.4 supervised + edit-regression v2** |
|---|---|---|---|---|
| AUROC (test half) | 0.7046 | 0.9187 | 0.9997 | **1.0** |
| TPR@FPR5% | 30.4% | 49.0% | 99.95% | **100%** |
| TPR@FPR1% | 16.7% | 24.9% | 99.88% | **100%** |
| Human FPR @calibrated p99 | not measured | not measured | 3.56% (30/844) | **0.71% (6/844)** |
| Paraphrase/mixed-attack AUROC | 0.64 | 0.89 | 1.0 (in-corpus) | **1.0** |
| Attack "AI-assisted" recall | — | — | 71.1% | **86.6%** |
| OOD plain-narrative/film recall | — | — | 1/6 | **5/6 supervised-only · 6/6 local fusion** |

**Which column applies to you?** The base package runs the **v3.0 rules+spectrum engine** (0.9187 AUROC column, measured on the full held-out corpus; the bundled small-corpus regression measures 0.9022 — see `eval/results/v350_release.json`). The two right-hand columns require the optional local supervised model (see below). The engine tells you honestly which mode you are in: every report carries `degraded_mode` / `degraded_notice` when the supervised layer is absent or skipped.

### Capability boundary matrix (read before trusting any detector)

| Scenario | Behavior |
|---|---|
| Chinese academic prose, full stack | Best case (AUROC 1.0 held-out, human FPR 0.71%) |
| Base package without model | Rules+spectrum (0.9187); **medical register over-scored** (rules-only human FPR @medium: ~59% medical vs ~2% general) → trust only @high verdicts on medical text |
| English text | Language gating skips the Chinese-trained supervised layer by design; rules-only English skeleton, advisory only |
| Mixed human+AI documents (document-level) | AUROC 0.38 — a principled limitation of document-level averaging; use `paragraph_report.py` attribution instead |
| Edit-extent regression head | ρ=0.540 — reported as metadata, never used in verdicts |
| Colloquial / oral-register text | The style layer is calibrated on academic prose; treat style scores as advisory outside that register |

## What's new in v3.6.0

- **Academic-integrity guardrails**: every report now carries an explicit `integrity_notice` field/line; new "Academic integrity" section; positioning stated plainly — author self-review and writing quality, compliance with disclosure rules, not detector evasion.
- **Docs hardened for platform policy**: evasion-flavored phrasing replaced with quality-framed language in the English documentation (detection and revision guidance stay; no detector-evasion framing). Chinese documentation keeps the SkillHub-approved wording.

## What's new in v3.5.0

- **Degraded-mode disclosure**: `ai_detector.py` now reports `degraded_mode` + `degraded_notice` (JSON and text) whenever the supervised layer is absent, disabled (`PP_NO_SUP=1`), or skipped by language gating — including the medical-register over-score warning with the actual held-out numbers.
- **Iron law #2 enforced**: texts under 100 characters now return `risk=unknown` with an explicit no-verdict notice (previously documented but not implemented; short texts also show as "cannot judge" in `quality_report.py` instead of a misleading green).
- **`scripts/pp_doctor.py`**: one-command environment self-check — data integrity, script compilation, optional deps, model presence, supervised-layer loadability, short-text/long-text/determinism probes, deai_gate guard. Exit 0 = green.
- **`deai_gate.py` usage guard + closed fallback loop**: `--help` / missing file no longer run the gate on a bogus filename; layer timeouts are caught (neutral 50); a failed smell layer now falls back to a neutral 50 instead of 0, and a failed terminology layer no longer dumps tracebacks into notes.
- **Chinese-Windows encoding hardening**: every entry point forces UTF-8 stdout/stderr and tolerates non-UTF-8 (e.g. GBK) input files — no more crashes on default zh-CN consoles (found by adversarial multi-expert testing).
- Docs rebuilt in honest dual-language form (this file + SKILL_ZH.md); trigger words expanded (AI率 / 查AI率 / AIGC 检测 …).

## Architecture (v3)

```
ai_detector.py            Main engine: 8 rule layers (125 recalibrated patterns, markdown caps,
                          EN openers, paragraph-level language) + length-routed fusion
 + layers_surface.py      L9 surface stats L10 token-spectrum (9,955-token delta spectrum)
                          L11 chain-of-thought features
 + fusion_config.json     Weights & thresholds (calib-half grid search + human p95/p99)
 + model_fingerprints.json v4 fingerprint registry (11 families incl. GLM-5.3 self-sampled; attribution only)
 + layers_lm.py           Optional supervised layer (local ONNX + pure-Python Qwen tokenizer;
                          PP_NO_SUP=1 falls back to rules)
paragraph_report.py       Paragraph-level attribution HTML (pattern×spectrum 50/50 fusion)
aigc_label_check.py       AIGC compliance labels (China labeling rules 2025-09: metadata/C2PA/explicit)
fingerprint_miner.py      Fingerprint mining (new model drop → sample → mine → register)
pattern_recalibrator.py   Data-driven pattern recalibration (human-hit filtering)
build_spectrum.py / calibrate_v3.py   Spectrum build / weight calibration
freshness_cron.py         Monthly freshness pipeline (sample → rebuild → calibrate → regression)
pp_doctor.py              Environment self-check (v3.5)
eval/                     corpus_builder / attack_gen / run_eval (AUROC, TPR@FPR, per-model, attack decay)
```

## Quick start

```bash
# AI writing detection (probability + layered evidence + fingerprint attribution)
python scripts/ai_detector.py draft.txt --format json
# Journal precheck (suspected-AIGC ratio vs the 20-25% reference line, non-interchangeable disclaimer)
python scripts/ai_detector.py draft.txt --profile journal
# Paragraph-level attribution (locate human/AI collaboration)
python scripts/paragraph_report.py draft.txt --output report.html
# AIGC compliance label check (docx/pdf/png/txt)
python scripts/aigc_label_check.py manuscript.docx figures/*.png
# Terminology / translation smell / 4-layer gate (same as v2)
python scripts/term_check.py draft.txt --auto-fix
python scripts/translation_smell_check.py draft.txt
python scripts/deai_gate.py draft.txt
# Environment self-check
python scripts/pp_doctor.py
# Held-out regression (mandatory after any engine change)
python eval/run_eval.py --split test --tag mytag
```

### Optional supervised layer (recommended, v3.2+)

```bash
pip install onnxruntime regex          # the two optional dependencies
# Place the two model files exactly as shipped by the authors:
#   ~/.cache/paper-polisher/qwen3-detector/model.int8.onnx
#   ~/.cache/paper-polisher/qwen3-detector/tokenizer.json
python scripts/layers_lm.py            # self-test: supervised_available: true
# ai_detector.py fuses automatically afterwards (0.9*supervised + 0.1*rules);
# PP_NO_SUP=1 temporarily falls back to rules-only.
# ⚠️ Do not substitute other exports or quantizations — measured probability drift; use exactly these files.
```

## Trigger words (Chinese)

`润色论文` `查AI率` `论文AI率` `AIGC检测` `AIGC率` `GPT检测` `查AI写作` `论文润色` `改写论文` `AI论文检测` `学术写作助手` `AI写作检测` `毕业论文润色` `学位论文降重` `SCI论文编辑` `手稿润色` `AI写作评分` `AI改写检测` `文风对标顶刊` `这篇文章像不像AI`

## Related skills (Paper Toolbox family)

- **cn-med-oa** — free Chinese medical literature (OA) download & citation metadata
- **pubmed-verifier** — verify PMID/DOI references before submission
- **cite-holmes** — deep research with machine-verified citations
- **academic-figures** — publication-ready scientific figures in one command
- **doc-holmes** — layout-preserving PDF translation

Writing a paper? The family covers the full loop: literature → verified citations → de-AI polishing → figures.

## Fingerprint freshness (against "detectors lag one generation")

On a new-model release day: `python scripts/fingerprint_miner.py --corpus <new_samples.jsonl> --model <family> --apply`
Monthly full pass: `python scripts/freshness_cron.py` (crontab `0 3 1 * *`). Compare adjacent `eval/results/freshness_*.json`; investigate if AUROC drops by more than 3 percentage points.

## Version history (condensed)

- **v3.5.0 (2026-09-20)** — degraded-mode disclosure (engine mode + medical-register warning with held-out numbers); iron law #2 enforced (<100 chars → risk=unknown, quality_report shows "cannot judge" instead of misleading green); `pp_doctor.py` self-check; `deai_gate.py` usage guard; honest dual-language docs rebuild.
- v3.4.3 — markdown table-separator rows filtered from paragraph scoring (6/8 flagged rows in real MD manuscripts were false positives).
- v3.4.2 — fixed CJK double-count in language detection (Chinese journal PDFs misrouted to EN rules); degenerate PDF hard-line-break paragraph rebuilding (747→19 segments); paragraph-level language routing dead code fixed.
- v3.4.1 — language gating: English text skips the Chinese-trained supervised layer (measured EN OOD p_ai=0.9996 → EN false positives 91.7→17.2). Rule-editor experiment: negative result, honestly abandoned.
- v3.4.0 — edit-extent regression v2 (1,620 pairs, token-level distance, two-stage training): human FPR@p99 1.66%→0.71%; attack "AI-assisted" recall 86.6%.
- v3.2/v3.3 — supervised layer v3.2 (4-dim head, local ONNX fp16, pure-Python Qwen tokenizer); OOD blind spots honestly recorded then closed (film-register recall 1/6→5/6, GLM-5.3 probe 9/9).
- v3.1 — Qwen3-0.6B LoRA supervised layer (AUROC 0.9997 held-out).
- v3.0 — eval-driven rebuild: recalibrated pattern library (693→125 patterns, 568 dead/inverted signals removed), token-spectrum layer, length-routed fusion, calib/test leak-proof split, fingerprint registry v4, paragraph attribution, AIGC label check, journal precheck, freshness pipeline, honest docs. AUROC 0.7046→0.9187.
- v2.0.x — 9-layer rule engine + terminology library (baseline column above; non-reproducible claims removed).
