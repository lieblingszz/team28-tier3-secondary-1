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

- **B.1 Model name(s)**: **Submitted entry:** GPT-4o, provided by OpenAI. The requested API model identifier was `gpt-4o`, a rolling alias rather than a dated snapshot. The exact resolved snapshot was not captured for the committed run; the pipeline now records the API response's `.model` value as `resolved_model` in `raw_log_*.jsonl`. GPT-4o is a proprietary hosted model; its parameter count and internal architecture are not publicly specified, and no open-weight software license applies. OpenAI documents a 128,000-token context window, a 16,384-token maximum output, and an October 1, 2023 knowledge cutoff ([official model documentation](https://developers.openai.com/api/docs/models/gpt-4o)).

  The same deposited code also supports two **alternative single-model variants**, which are not part of this GPT-4o entry and require their own registration forms if deposited:

  - **GPT-5.4 (OpenAI):** requested API identifier `gpt-5.4`, a rolling alias. OpenAI's model catalog lists the dated snapshot `gpt-5.4-2026-03-05`, a 1,050,000-token context window, a 128,000-token maximum output, and an August 31, 2025 knowledge cutoff. It is a proprietary hosted reasoning-capable model; parameter count and internal architecture are not publicly specified ([official model documentation](https://developers.openai.com/api/docs/models/gpt-5.4)). The exact resolved model for a run must be taken from that run's raw log.
  - **Claude Sonnet 4.6 (Anthropic):** API identifier `claude-sonnet-4-6`. Anthropic introduced the model on February 17, 2026 and documents a 1-million-token context window in beta. It is a proprietary hosted model; parameter count and internal architecture are not publicly specified ([Anthropic announcement](https://www.anthropic.com/news/claude-sonnet-4-6)). The exact model string returned for a run must be taken from that run's raw log.

- **B.2 Access & context mode**: Hosted API inference, not local/self-hosted inference. GPT-4o and GPT-5.4 use OpenAI's Chat Completions API through the `openai` Python SDK; Claude Sonnet 4.6 uses Anthropic's Messages API through the `anthropic` Python SDK. Calls are stateless and single-turn: every request contains one `user`-role prompt and no `system` message. Step-1 scoring calls and Step-2 effect-estimation calls are independent; only the averaged Step-1 score is inserted into the relevant Step-2 prompt as ordinary supporting context. No conversation history or provider-side memory carries between calls. The code does not impose a separate context-window cap, so the provider/model limit applies. A full run took approximately 45 minutes per model. The committed GPT-4o result filename records a run timestamp of August 25, 2026 at 13:29:46; using the reported runtime, the run ended at approximately 14:14:46. The timezone and call-level timestamps were not retained. Exact dates for the alternative GPT-5.4 and Claude runs are not documented in this entry's committed result files.

- **B.3 Configuration**: Shared sampling structure: 3 independent Step-1 scoring attempts per raw survey item, followed by 3 independent Step-2 samples per intervention×item pair; calls are made separately with one completion per API request and averaged downstream. Default concurrency is at most 5 worker threads. No model variant sets `top_p`/top-k, frequency or presence penalties, stop sequences, or a fixed seed, so exact bit-for-bit reproduction is not expected.

  - **GPT-4o (submitted entry):** temperature `0.5`; maximum output tokens 600 for Step 1, 110 for Step 2, and 15 for the parse-failure fallback. Reasoning effort: N/A in this pipeline.
  - **GPT-5.4 alternative:** no temperature is passed; `reasoning_effort` is explicitly `none`. The code uses `max_completion_tokens` and adds a 2,000-token reasoning-family allowance, yielding limits of 2,600 for Step 1, 2,110 for Step 2, and 2,015 for fallback calls. These are output-budget settings, not context-window limits.
  - **Claude Sonnet 4.6 alternative:** no temperature is passed, so the Anthropic API default applies; maximum output tokens are 600 for Step 1, 110 for Step 2, and 15 for fallback calls. Extended/adaptive thinking is not enabled. Reasoning effort: N/A for this run configuration.

- **B.4 Customization**: No fine-tuning for any model variant. RAG is enabled by default for all three variants: ChromaDB + OpenAI `text-embedding-3-small`, over the manually curated literature corpus described in H.2. Retrieval is outcome-specific in Step 1 and intervention×outcome-specific in Step 2; full intervention text is supplied without truncation. No automated prompt-optimization loop was used; prompts were manually iterated during development (C.1) and then held fixed. No model tools, web search, or general agentic scaffold are used. The fixed two-step forecasting workflow and final global calibration factor of 0.35 are pipeline-level processing, not model customization.

- **B.5 Persistent memory**: None. Every API request is independent, with no multi-turn model conversation and no memory across items, interventions, attempts, samples, or model variants. Passing the averaged Step-1 score into a Step-2 prompt is explicit pipeline-derived context, not persistent model memory.

- **B.6 Inference stack**: Provider-hosted inference. GPT-4o and GPT-5.4 use the OpenAI Python SDK and Chat Completions API; Claude Sonnet 4.6 uses the Anthropic Python SDK and Messages API. Shared retrieval uses ChromaDB with `text-embedding-3-small`. The repository specifies only minimum dependency versions (`openai>=1.0.0`, `anthropic>=0.50.0`, `chromadb>=1.0.0`), so the exact SDK versions used for an earlier unlogged run cannot be recovered from these files. GPU model, numerical precision, quantization, and tensor parallelism are provider-managed and therefore unknown/not applicable.

- **B.7 Ensembles**: N/A for this entry: the submitted predictions use GPT-4o only. The three Step-1 attempts and three Step-2 samples are repeated stochastic draws from the same model followed by averaging, not a cross-model ensemble. GPT-5.4 and Claude Sonnet 4.6 are alternative single-model variants supported by the code; they must be registered as separate entries unless their outputs are explicitly pooled into a distinct ensemble submission.

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

- **F.2 Aggregation rule**: Step 1: mean of the 3 parsed scores per intervention; the operators observed no case in which all three scoring attempts failed, so the all-zero neutral fallback was not used. Step 2: mean of the 3 parsed ATEs per (intervention, item). Item-level ATEs are meaned across each official outcome's constituent raw items, sign-flipped for `funding_perceptions`, then the full 208-row table is multiplied by the calibration constant (0.35) for the submitted file.

## G · Validation & post-processing

- **G.1 Human validation**: N/A; no human reviewed individual model outputs.

- **G.2 Post-processing**: Numeric extraction supports a stricter fallback re-prompt on Step-2 parse failure and a neutral all-zero Step-1 vector if all 3 scoring attempts fail. The operators observed no parsing failures during the submitted run: 0 Step-1 all-zero fallbacks and 0 Step-2 fallback re-prompts. No responses were excluded or dropped. Raw API logs were not retained, so these counts are based on the operators' run observation rather than a retrospective log audit. "Effective N per condition" — N/A, no individual synthetic respondents generated.

- **G.3 Calibration corrections**: Yes. All values multiplied by a global constant, `CALIBRATION_FACTOR = 0.35`, as the last step before saving the calibrated file — corrects for LLMs' documented tendency to overestimate treatment-effect magnitudes. Originally 0.56, following Ashokkumar et al. (2026)'s primary-archive correction figure; empirically re-validated against real human ground truth from Voelkel et al. (2025/2026, *Nature Climate Change*) — 4 outcome composites × 10 conditions, 40 predicted/real pairs, through-origin regression + RMSE comparison. 0.35 sits close to the empirically best-fit value (0.321) on that data and clearly beats 0.56 (pooled RMSE 1.123 vs 1.255). Applied globally, not per-intervention/outcome — no real ground truth exists for the actual 16 target interventions to fit a finer-grained version. Cross-ref H.2, I.2, J.1.

## H · Learning and conditioning components

- **H.1 Fine-tuning data**: N/A.

- **H.2 Context & retrieval corpora**: A curated corpus of 43 published PDF papers on climate/science communication and trust, chunked and indexed in the persistent ChromaDB store at `code/chroma_rag_db/` and embedded with `text-embedding-3-small`. The packaged index contains 1,372 records. The corpus includes Voelkel et al.'s own paper (`voelkel_2026_megastudy.pdf`) — intentional, real literature legitimately informing real predictions (see I.4 for why this isn't circular with the calibration-validation work in G.3).

## I · Data inputs, blinding, and competing interests

- **I.1 Competing interests ★**: No team member has any competing interest to disclose. Farah Adeeba's OpenAI API costs for this Tier 3 entry were paid personally, out of pocket — no institutional or third-party funding for the API usage specifically. (Cross-ref Max's Tier 1 entry, whose compute was an institutional in-kind allocation — different funding path per entry, both disclosed on their own forms.)

- **I.2 External human data †**: (a) Ashokkumar et al. (2026) — source of the original 0.56 calibration heuristic. (b) Voelkel et al. (2025/2026, *Nature Climate Change*) — used to empirically re-validate and refit the calibration constant to 0.35 (G.3, J.1); not used for training/fine-tuning, only to fit one global scalar. (c) The RAG corpus (H.2) — published papers reporting human experimental findings, retrieved into context at inference time (in-context, not training); this is the approach's intended mechanism, disclosed here for completeness.

- **I.3 Blinding attestation ★** — **mandatory**: I, Farah Adeeba, attest that I did not access, solicit, or was shown any human outcome data from this study, including pilots, before the prediction lock. Real human data referenced elsewhere in this form (Voelkel et al. 2025/2026; Ashokkumar et al. 2026) are independent, already-published studies unrelated to this benchmark's own target study, used only as disclosed in I.2/G.3/J.1.

- **I.4 Contamination note †**: GPT-4o has a training-data/knowledge cutoff of October 2023 according to OpenAI's model documentation. Voelkel et al.'s paper was published in 2026 and therefore postdates that cutoff, so it could not have entered GPT-4o's training data through that publication. The team did not independently establish the benchmark target materials' first public-release date from an archived source, so no stronger claim is made about whether those materials predate or postdate the cutoff. The blinding attestation in I.3 remains the team's direct statement that no target-study human outcome data, including pilots, were accessed before the prediction lock. The RAG corpus intentionally includes Voelkel's paper as retrieved-at-inference-time context (H.2) — in-context exposure, not training-data contamination. For the separate calibration validation against Voelkel's human ground truth, Voelkel citations were excluded from retrieval to avoid circularity.

## J · Internal selection procedure

- **J.1 Design-space search †**: Configurations tried during development: RAG-grounded prompting was the core design from the start (not tested on/off against an accuracy criterion — it's the approach's defining feature, not a tuned choice); full untruncated intervention text replaced an earlier truncated version specifically to remove a known truncation-bias risk (not selected via a quantitative comparison either — a correctness fix, not a tuning choice); at least 3 named prompt variants ("optionA/B/C") were tried earlier in development. The one criterion that *was* quantitatively validated is the calibration constant (G.3/I.2): correlation and RMSE against Voelkel et al. (2025/2026) real human data, used to choose 0.35 over the original 0.56 and over the empirically best-fit 0.321. No target-study human data was consulted at any point (I.3).

## K · Reproducibility & frozen artifacts

- **K.1 Code & materials**: The reproducibility package is the repository's `code/` directory. It contains `tier3_pipeline.py`, `tier3_pipeline_parallel_fulltext.py`, `rag_vector_db.py`, the prebuilt `chroma_rag_db/`, all 16 intervention texts, `survey_items.json`, `requirements.txt`, and timestamped raw and calibrated result CSVs. The official submitted CSV is in `predictions/`. No secrets are included; API keys are read from environment variables only. `metadata.json` records the GitHub repository as `code_repository`. No separate code-only DOI was created, so `code_doi` remains null; the repository code will be included in the submission's Zenodo release snapshot.

- **K.2 Raw output logs †**: Complete raw API-response logs were not retained for the submitted run. Consequently, the exact per-call prompts/responses, resolved model snapshot, token usage, and call-level timestamps cannot be reconstructed retrospectively. The deposited code now supports JSONL raw-response logging for future runs, but that functionality was added after the submitted predictions were generated. The team is disclosing this limitation rather than rerunning a stochastic pipeline and replacing the original predictions.

- **K.3 Computational resources**: Formula: `n_items × (step1_attempts + n_conditions × n_samples)` = 44 × (3 + 16×3) = **2,244 completion calls** per full model run, plus any retry calls. RAG performs 44 outcome-level retrieval-query embeddings and 44 × 16 × 2 = 1,408 intervention/outcome retrieval-query embeddings, for approximately **1,452 embedding queries** per full run. Wall-clock time was approximately 45 minutes per model. Reported API cost was approximately **USD 20–30 per complete model run** for Claude and similarly for the GPT variants. Exact token counts and dashboard-billed amounts were not retained.

## L · Disclosure class

The final disclosure classification is **A · Open**, as recorded in `metadata.json`. The absence of historical raw API logs and exact token accounting is disclosed in K.2–K.3; no information is withheld under seal or escrow.
