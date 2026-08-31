# Silicon Sample Benchmark — Tier 3 Submission (clean, final)

**Team:** Farah Adeeba · Jing Ma · Max Pellert · Márcia Ferreira
**Team ID:** team_28
**Deadline:** 31 August 2026

This folder is the minimal, verified set of files needed to reproduce and
submit the Tier 3 result. Nothing here was changed from your working
files — this is a clean copy, not an edit.

## What's in here and why

| File / folder | Why it's needed |
|---|---|
| `tier3_pipeline_parallel_fulltext.py` |  Parallel + full-text + RAG, no truncation. 
| `tier3_pipeline.py` | Not legacy — the script above imports its core logic directly (`MODEL_CONFIG`, `CALIBRATION_FACTOR`, prompt helpers, calibration, aggregation, file-saving). Required. |
| `rag_vector_db.py` | `VectorRAG` class — loads and queries the vector database. |
| `chroma_rag_db/` | The already-built vector index (1,372 records from 43 PDF papers). Required for the full RAG configuration at runtime; without an API key and access to the persistent index, `VectorRAG` uses its 24-record hardcoded TF-IDF fallback corpus instead of the full literature corpus. Don't skip this folder. |
| `LLMmegastudy-main/simulation/stimuli/` | The 16 intervention text files. |
| `LLMmegastudy-main/simulation/data/survey_items.json` | The 13 outcome item definitions. |
| `requirements.txt` | `pip install -r requirements.txt` |
| `results/tier3_submission_calibrated_..._132946.csv` |  GPT-4o + RAG + full text, calibrated with b=0.35. 208 rows, `condition,outcome,ate`. |
| `results/tier3_submission_raw_..._132946.csv` | Same run, before calibration — kept alongside for provenance/audit only. |

Verified: `python tier3_pipeline_parallel_fulltext.py --dry_run --quick` runs end-to-end
on exactly this file set with no missing dependencies.

## To reproduce the submission file from scratch

```bash
pip install -r requirements.txt
export OPENAI_API_KEY="your-key"     
python tier3_pipeline_parallel_fulltext.py --model gpt4o --n_samples 3
```

Output lands in `results/`. Use the `tier3_submission_calibrated_*` file.

## Calibration

Raw LLM effect-size predictions are multiplied by **b = 0.35** before saving
the calibrated file (`CALIBRATION_FACTOR` in `tier3_pipeline.py`). This value
was empirically fit against real Voelkel et al. (2025) ground truth.
