# Silicon Sample Benchmark — method registration form

**Entry covered by this form:** Tier 3, GPT-4o + RAG pipeline (`tier3_pipeline_parallel_fulltext.py`).

> Scope note: this covers this one entry only. GPT-5 / Claude fulltext runs, if deposited as `secondary-k` entries, each need their own copy of this form.

---

## 0 · Approach identity and output

- **0.1 Team ★**: RAG team silicon sample study. Members/creators: Farah Adeeba (University of Konstanz); Jing Ma (University of Konstanz); Marcia Ferreira Goncalves (Graphwise); Max Pellert (Barcelona Supercomputing Center). Corresponding contact: max.pellert@bsc.es (same as the Tier 1 entry).

- **0.2 Plain-language summary ★**: For each of the 16 interventions and 13 outcomes, GPT-4o predicts the treatment effect directly — no simulated respondents. First it ranks all 16 interventions relative to each other for that outcome, then independently estimates the actual effect size for each intervention, grounded in evidence retrieved from a curated corpus of 30+ published science-communication studies. Raw predictions are then scaled by a fixed factor, empirically fit against real human data, correcting for LLMs' known tendency to overestimate effect sizes.

- **0.3 Submission tier & approach family ★**: Tier 3. Approach family: direct forecast (no persona/respondent simulation); single model (GPT-4o); literature-conditioned via retrieval-augmented generation — not zero-shot; no fine-tuning.

- **0.4 Pipeline diagram**:
  1. Load the 16 official intervention texts + control, and the 44 raw survey items (which roll up into the 13 official outcomes).
  2. For each of the 44 raw items:
     a. Retrieve the top-5 literature chunks relevant to that outcome (ChromaDB + `text-embedding-3-small`).
     b. **Step 1 (relative scoring):** GPT-4o scores all 16 interventions at once, −3..+3, for this outcome; 3 independent attempts, averaged per intervention.
     c. For each of the 16 interventions, retrieve up to 6 merged literature chunks (intervention text + outcome query).
     d. **Step 2 (effect estimation):** GPT-4o predicts the actual ATE for this intervention/outcome, given the full intervention text, retrieved evidence, and the Step-1 score as soft supporting context (explicitly instructed not to mechanically rescale it); 3 independent samples, averaged.
  3. Average the 44 raw item-level ATEs into the 13 official outcomes; sign-flip `funding_perceptions` (see G.3 — its raw item is worded in the opposite direction from the official outcome).
  4. Save the raw (uncalibrated) 208-row submission file.
  5. Multiply every value by the calibration constant (0.35) → save the calibrated 208-row submission file. **This is the file submitted.**

- **0.5 Coverage ★**: All 16 interventions and all 13 outcomes covered. 208 rows (16 × 13), `condition,outcome,ate` format. Confirmed against the submitted file.

## A · Scope of LLM use

- **A.1 Purpose**: GPT-4o is used at exactly two pipeline stages: Step-1 relative-effectiveness ranking, and Step-2 effect-size magnitude prediction. A separate embedding model (`text-embedding-3-small`) is used only for RAG retrieval, not for any prediction. No LLM simulates individual respondents — this tier has no persona step (see D).

- **A.2 Degree of automation ★**: Fully automated at prediction time; no human edited any intermediate output. The code also has a `--manual` copy-paste mode for running without an API key — not used for this submission; the actual run used the API mode.

## B · Model / system details

- **B.1 Model name(s)**: Provider: OpenAI. Model identifier requested: `gpt-4o` (rolling alias, not a dated snapshot). **TODO**: exact resolved snapshot — the pipeline now captures this automatically (`resolved_model` field, from the API response's own `.model`) once the logged rerun below is done; paste it here from the first line of `raw_log_*.jsonl`.

- **B.2 Access & context mode**: OpenAI Chat Completions API (`openai` Python SDK), stateless — every call is an independent request, no thread/session state carried between calls. **TODO**: exact call date range — will be captured automatically in every log line's `ts` field once the rerun below is done.

- **B.3 Configuration**: Temperature 0.5 (deliberately not 0, so the repeated sampling in F actually varies). Max tokens: 600 (Step-1 scoring), 110 (Step-2 effect), 15 (Step-2 fallback single-number re-prompt, used only if the main parse fails). No top-p/top-k/penalty/stop-sequence overrides. No `seed` set — not bit-for-bit reproducible under identical settings. Reasoning effort: N/A (`gpt-4o` is not a reasoning model in this codebase's sense). Completions per item: 3 Step-1 attempts + 3 Step-2 samples per intervention, both averaged.

- **B.4 Customization**: No fine-tuning. RAG: yes — ChromaDB + `text-embedding-3-small`, over a manually curated corpus (see H.2). No automated prompt-optimization loop; prompts were manually iterated during development (see C.1), then held fixed for this run. No tool use, no web search, no agentic scaffold beyond the fixed two-step structure in 0.4.

- **B.5 Persistent memory**: None across calls.

- **B.6 Inference stack**: N/A — hosted API model, not self-hosted.

- **B.7 Ensembles**: N/A for this entry — single model. (Repeated same-model sampling that gets averaged is documented under F, not here.)

## C · Prompts

- **C.1 Exact prompts**: Verbatim text deposited alongside the code (see K.1); `build_scoring_prompt_fulltext()` and `build_effect_prompt_fulltext()` in `tier3_pipeline_parallel_fulltext.py`. Iteratively refined during development (earlier truncated-text and prompt-wording variants tested, see J.1); held fixed for this run, pre-specified rather than adjusted in response to this run's own outputs.

- **C.2 System-wide instructions**: None — no `system`-role message is used; every call sends a single `user`-role message. The "You are an expert in behavioral science..." framing is part of that user-role text itself, not a separate system instruction.

- **C.3 Prompt-design rationale**: The two-step structure (rank, then independently estimate magnitude) forces differentiation between interventions while keeping the magnitude estimate reasoned rather than a mechanical rescaling of the rank — the prompt explicitly tells the model not to just multiply the Step-1 score. Full untruncated intervention text replaced an earlier truncated version once truncation was identified as a bias risk. RAG grounding anchors magnitude estimates to retrieved findings instead of the model's unaided prior. Temperature 0.5 was chosen specifically so the repeated-sampling average in F has real variance to average over.

## D · Persona / profile construction (Tiers 1–2)

N/A — Tier 3, direct forecast, no simulated respondents.

## E · Stimulus and survey administration

- **E.1 Stimulus presentation**: Verbatim, full intervention text, no paraphrasing, no truncation. No state-contingent stimulus content in these 16 interventions.

- **E.2 Survey walk-through**: Not a whole-survey simulation — each of the 44 raw items handled independently, no context carried over between items or calls. **Item and condition order is fixed, not randomized** — both follow the order interventions/items are listed in `survey_items.json`, on every call. This is a genuine limitation relative to survey-randomization best practice (unlike the Tier 1 entry, which randomizes block/item order per call): a fixed ranking-list order is a plausible, if likely small, source of position bias in Step 1 specifically. Scale display: each item's own scale-label text, inserted verbatim from `survey_items.json`. No attention/comprehension checks — not applicable, no simulated respondents.

- **E.3 Response elicitation**: Step 1: structured JSON object, intervention name → score, parsed with `parse_scores()`. Step 2: constrained free text (≤15 words reasoning) ending in a required `ANSWER: <number>` line, parsed with `parse_number()`; a stricter single-number-only fallback prompt is sent if parsing fails. Not logprob-based.

## F · Stochasticity and aggregation

- **F.1 Runs & seeds**: Step 1: 3 independent scoring attempts per item, run concurrently, each scoring all 16 interventions at once. Step 2: 3 independent samples per (intervention, item) pair, run concurrently. No `seed` set; temperature 0.5 — not reproducible bit-for-bit under identical settings, only in distribution across repeated draws.

- **F.2 Aggregation rule**: Step 1: mean of the 3 parsed scores per intervention (all-zero neutral fallback if every attempt fails to parse — **TODO**: confirm from the rerun's log whether this fallback ever actually fired, see G.2). Step 2: mean of the 3 parsed ATEs per (intervention, item). Item-level ATEs are meaned across each official outcome's constituent raw items, sign-flipped for `funding_perceptions`, then the full 208-row table is multiplied by the calibration constant (0.35) for the submitted file.

## G · Validation & post-processing

- **G.1 Human validation**: N/A; no human reviewed individual model outputs.

- **G.2 Post-processing**: Numeric extraction with a stricter fallback re-prompt on Step-2 parse failure; Step-1 items falling back to a neutral all-zero score vector if all 3 attempts fail to parse. No responses excluded/dropped outright — every failure either recovers via fallback or degrades to neutral, never silently drops a cell. **TODO**: exact parse-failure/fallback count for this entry — the new logged rerun (see K.2) makes this countable directly (`grep step2_fallback` in `raw_log_*.jsonl`, plus any Step-1 all-zero fallbacks the console output flags); not available for the earlier, unlogged run. "Effective N per condition" — N/A, no individual synthetic respondents generated.

- **G.3 Calibration corrections**: Yes. All values multiplied by a global constant, `CALIBRATION_FACTOR = 0.35`, as the last step before saving the calibrated file — corrects for LLMs' documented tendency to overestimate treatment-effect magnitudes. Originally 0.56, following Ashokkumar et al. (2026)'s primary-archive correction figure; empirically re-validated against real human ground truth from Voelkel et al. (2025/2026, *Nature Climate Change*) — 4 outcome composites × 10 conditions, 40 predicted/real pairs, through-origin regression + RMSE comparison. 0.35 sits close to the empirically best-fit value (0.321) on that data and clearly beats 0.56 (pooled RMSE 1.123 vs 1.255). Applied globally, not per-intervention/outcome — no real ground truth exists for the actual 16 target interventions to fit a finer-grained version. Cross-ref H.2, I.2, J.1.

## H · Learning and conditioning components

- **H.1 Fine-tuning data**: N/A.

- **H.2 Context & retrieval corpora**: A curated corpus of published papers on climate/science communication and trust, chunked and indexed in a persistent ChromaDB store (`chroma_rag_db/`), embedded with `text-embedding-3-small` — 795 chunks at last count. **TODO**: reconcile "30+ papers" against the `corpus/` folder's current 43 PDFs before depositing — confirm `chroma_rag_db/` reflects all 43, or note the true indexed count. The corpus includes Voelkel et al.'s own paper (`voelkel_2026_megastudy.pdf`) — intentional, real literature legitimately informing real predictions (see I.4 for why this isn't circular with the calibration-validation work in G.3).

## I · Data inputs, blinding, and competing interests

- **I.1 Competing interests ★**: No team member has any competing interest to disclose. Farah Adeeba's OpenAI API costs for this Tier 3 entry were paid personally, out of pocket — no institutional or third-party funding for the API usage specifically. (Cross-ref Max's Tier 1 entry, whose compute was an institutional in-kind allocation — different funding path per entry, both disclosed on their own forms.)

- **I.2 External human data †**: (a) Ashokkumar et al. (2026) — source of the original 0.56 calibration heuristic. (b) Voelkel et al. (2025/2026, *Nature Climate Change*) — used to empirically re-validate and refit the calibration constant to 0.35 (G.3, J.1); not used for training/fine-tuning, only to fit one global scalar. (c) The RAG corpus (H.2) — published papers reporting human experimental findings, retrieved into context at inference time (in-context, not training); this is the approach's intended mechanism, disclosed here for completeness.

- **I.3 Blinding attestation ★** — **mandatory**: I, Farah Adeeba, attest that I did not access, solicit, or was shown any human outcome data from this study, including pilots, before the prediction lock. Real human data referenced elsewhere in this form (Voelkel et al. 2025/2026; Ashokkumar et al. 2026) are independent, already-published studies unrelated to this benchmark's own target study, used only as disclosed in I.2/G.3/J.1.

- **I.4 Contamination note †**: GPT-4o's training data cutoff is approximately October 2023 (per OpenAI's public model documentation). Voelkel et al.'s paper was published in 2026 — postdates GPT-4o's cutoff, so no contamination risk from that publication into GPT-4o's training data. **TODO**: confirm the benchmark's own target-study materials' first public-release date, to confirm GPT-4o's cutoff also predates those (the organizers' main contamination question, separate from the Voelkel point). Note: the RAG corpus intentionally includes Voelkel's paper as retrieved-at-inference-time context (H.2) — in-context exposure, not training-data contamination, and distinct from the calibration-validation script, which explicitly filters Voelkel citations out of its own retrieval before scoring against Voelkel's ground truth (avoiding circularity — see `validate_voelkel2025.py`'s `filtered_retrieve()`).

## J · Internal selection procedure

- **J.1 Design-space search †**: Configurations tried during development: RAG-grounded prompting was the core design from the start (not tested on/off against an accuracy criterion — it's the approach's defining feature, not a tuned choice); full untruncated intervention text replaced an earlier truncated version specifically to remove a known truncation-bias risk (not selected via a quantitative comparison either — a correctness fix, not a tuning choice); at least 3 named prompt variants ("optionA/B/C") were tried earlier in development. The one criterion that *was* quantitatively validated is the calibration constant (G.3/I.2): correlation and RMSE against Voelkel et al. (2025/2026) real human data, used to choose 0.35 over the original 0.56 and over the empirically best-fit 0.321. No target-study human data was consulted at any point (I.3).

## K · Reproducibility & frozen artifacts

- **K.1 Code & materials**: A clean, minimal, verified copy of exactly the code and data needed to reproduce this submission is at `submission_final/` in the project folder — `tier3_pipeline.py`, `tier3_pipeline_parallel_fulltext.py`, `rag_vector_db.py`, the prebuilt `chroma_rag_db/`, the 16 stimulus texts + `survey_items.json`, `requirements.txt`, and the submitted CSV (raw + calibrated). No secrets included (API keys are read from environment variables only). **TODO**: zip and deposit to Zenodo, then record the link/DOI here and in `metadata.json` → `code_repository`/`code_doi`.

- **K.2 Raw output logs †**: **TODO — action required before depositing.** The pipeline has been updated to log the complete, unprocessed text of every API call (one JSON line per call: timestamp, resolved model snapshot, exact prompt, exact raw response, and which step/item/condition/sample it was) to `results/raw_log_<label>_<timestamp>.jsonl`, verified working (mock-API test, no real calls, before this touched your machine). Run the rerun command below; the script prints the log file's path, line count, and SHA-256 hash at the end — paste those three values here, and this also fills in the B.1/B.2 TODOs above from the log's own `resolved_model`/`ts` fields.

- **K.3 Computational resources**: Formula: `n_items × (step1_attempts + n_conditions × n_samples)` = 44 × (3 + 16×3) = **2,244 completion calls**, plus ~34 cheap embedding/retrieval calls (not completions). **TODO**: exact token count and cost from the OpenAI usage dashboard for the rerun's date range, and wall-clock time (the script doesn't currently time itself — note start/end time from your terminal).

## L · Disclosure class

Given the answers above, nothing here currently requires **C · Sealed** or even escrowing — every † item (I.2, I.4, J.1, K.2) has a disclosable answer once the TODOs are filled, and I.1/I.3 (★, must be public) are already clean. **A · Open** looks achievable. Record the final choice in `metadata.json` → `disclosure_class`.

---

## Next step: the rerun that fills in the remaining TODOs

```bash
cd ~/Desktop/challenge/challenge
export OPENAI_API_KEY="your-key"
python3 tier3_pipeline_parallel_fulltext.py --model gpt4o --n_samples 3
```

This produces a **new** timestamped submission file (temperature 0.5, no seed — values will shift slightly from the current one, per F.1) plus `results/raw_log_..._<new-timestamp>.jsonl` with the SHA-256/line-count printed at the end. Once it's done: tell me the new timestamp and I'll (a) swap `submission_final/results/` to the new, fully-logged run, and (b) fold the printed hash/line-count/resolved-model into this form's remaining TODOs.
