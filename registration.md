# Silicon Sample Benchmark — method registration form


## 0 · Approach identity and output
- **0.1 Team ★** — team_6; Maximilian Kreutner, Markus Strohmaier; University Mannheim; corresponding contact: maximilian.kreutner@uni-mannheim.de
- **0.2 Plain-language summary ★** — We ask one LLM to directly forecast mean survey answers for groups defined by a single demographic attribute and level. For every intervention and control condition, the model sees the exact stimulus, the full scored questionnaire, and one of 27 age, education, gender, income, party, or race categories. Its item-level mean forecasts are converted to the benchmark outcomes, completed deterministically where responses are missing, and aggregated into overall and moderator-level Tier 2 predictions.
- **0.3 Submission tier & approach family ★** — Tier 2; direct group forecasting, single model, zero-shot.
- **0.4 Pipeline diagram** — Verbatim condition text + one demographic level + full questionnaire → QSTN battery prompt → Qwen direct mean forecasts → JSON parsing and repair → item-to-outcome construction → deterministic missing-cell fallback → moderator grid → equal-weight overall cell aggregation → Tier 2 CSVs.
- **0.5 Coverage ★** — Full coverage: 221 main cells (17 conditions × 13 outcomes) and 5,967 moderator cells (17 conditions × 27 levels across six moderators × 13 outcomes), covering control and all 16 interventions.

## A · Scope of LLM use
- **A.1 Purpose** — Qwen is used to directly forecast the item-level mean answers of a representative sample for every condition and demographic-level cell. All later parsing, outcome construction, completion, and aggregation steps are deterministic code.
- **A.2 Degree of automation ★** — Fully automated at prediction time; no human is involved in generation, parsing, repair, fallback completion, or aggregation.

## B · Model / system details (once per model)
- **B.1 Model name(s)** — Qwen/Qwen3.6-27B (Qwen, 27B), run locally; source: https://huggingface.co/Qwen/Qwen3.6-27B. The checkpoint used by production run 6501 was loaded with revision=None on 2026-07-23; the model was publicly released on 2026-04-22.
- **B.2 Access & context mode** — Local vLLM inference with stateless chat-style prompts. Production run 6501 ran on 2026-07-23 from 12:00:09 to 12:30:44 UTC.
- **B.3 Configuration** — QSTN passes vLLM SamplingParams with temperature=1.0, min_p=0.0, presence_penalty=0.0, frequency_penalty=0.0, repetition_penalty=1.0, no user-supplied stop strings, ignore_eos=false, and min_tokens=0. The model generation configuration supplies top_p=0.95 and top_k=20. bfloat16; tensor parallel size 2; GPU-memory utilization 0.95; maximum concurrent sequences 100; eager execution; custom all-reduce disabled; thinking enabled with <think>/</think> boundaries; maximum model length and maximum generation tokens 10,000. vLLM engine seed 0; QSTN default base seed 42 generates one deterministic request seed per prompt. One completion is requested for each of 459 prompts.
- **B.4 Customization** — N/A: no fine-tuning, RAG, prompt optimization against benchmark outcomes, tool use, web search, or agentic scaffolding.
- **B.5 Persistent memory** — N/A; prompts are stateless and no information persists across demographic-condition cells.
- **B.6 Inference stack** — vLLM 0.25.1, local two-GPU tensor-parallel inference, bfloat16, no quantization, Triton attention and kernels. Hardware: two NVIDIA H100 PCIe GPUs; Driver 580.126.20; CUDA 13.0.
- **B.7 Ensembles** — N/A; the submitted predictions use only Qwen/Qwen3.6-27B.

## C · Prompts
- **C.1 Exact prompts** — The source code repository contains the system and user templates in src/group_level/qstn_setup.py, the complete rendered example in prompt/group.txt, and the questionnaire in qstn_data/questionnaire.csv. Complete rendered prompts and responses will be included in the separate raw-output Zenodo deposit. The final prompt structure was fixed before production run 6501.
- **C.2 System-wide instructions** — The system message tells the model that it will receive one demographic and a set of questions, asks it to predict the mean answer for every question, and requires a JSON-only response containing every question ID.
- **C.3 Prompt-design rationale** — We use one battery prompt per cell so the model sees the same stimulus and questionnaire context together. Asking for the mean of a representative sample of 5,000 people makes the target explicitly group-level. Each prompt contains only one moderator and level to obtain the complete subgroup grid without inventing joint demographic profiles.

## D · Persona / profile construction (Tiers 1–2)
- **D.1 Profile source** — No individual personas are constructed. The approach uses all 27 published levels of the six benchmark moderators: age band (4), education (6), gender (3), income (5), party (4), and race (5).
- **D.2 Profile verbalization** — Each prompt states one moderator name and one level in a fixed template, for example age_band: 18-29. No generated narrative is used.
- **D.3 Assignment & weighting** — Every one of the 27 moderator levels is crossed with every control/intervention condition, yielding 459 prompts. No profiles are reused because the units are direct group forecasts. The submitted moderator file retains every level; the main file is the unweighted arithmetic mean across the 27 moderator-level forecasts for each condition and outcome.

## E · Stimulus and survey administration
- **E.1 Stimulus presentation** — QSTN inserts each condition text verbatim from qstn_data/conditions.json. The one text associated with the target condition is shown in each prompt.
- **E.2 Survey walk-through** — One full 44-item scored battery is administered in one completion per condition-demographic cell. The prompt preserves the questionnaire item and option order and displays the native scale anchors. There is no context carry-over, item randomization, or added attention/comprehension item.
- **E.3 Response elicitation** — Free-generation JSON object with one numeric group-mean prediction per item. Token log-probabilities and constrained decoding are not used.

## F · Stochasticity and aggregation
- **F.1 Runs & seeds** — Exactly one completion per condition-demographic cell, for 459 completions total. vLLM uses engine seed 0 and QSTN default base seed 42 generates a distinct deterministic request seed for each prompt. No repeated generations are averaged.
- **F.2 Aggregation rule** — Parsed item means are converted to the 13 benchmark outcomes: six composites are arithmetic means of their required components, funding_perceptions is reverse-coded as 100 minus the model response, and the remaining outcomes use their direct item values. Newsletter forecasts produced on the legacy 1=Yes, 2=No mean scale are converted to a signup proportion as 2 − x; the single response expressed directly as 35 percent is converted to 0.35. Duplicate cell values, if present, are averaged. The moderator file contains the completed condition × moderator level × outcome grid. The main mean for each condition and outcome is the unweighted arithmetic mean of its 27 moderator-level values.

## G · Validation & post-processing
- **G.1 Human validation** — N/A.
- **G.2 Post-processing** — Direct JSON parsing is attempted first, followed by JSON repair. Of 459 responses, five required repair and two remained unparseable, leaving 457 parsed response objects. Numeric strings are accepted by extracting their first numeric value, and a composite is emitted only when all its component predictions are present. Under the generation-time parser this yielded 5,928 observed moderator cells; 39 of 5,967 cells were completed deterministically using, in order: the mean for the same condition/outcome/moderator, the same condition/outcome, the same outcome/moderator level across conditions, the same outcome overall, then the native-scale midpoint. To harmonize the newsletter output with the required 0–1 proportion, values on the prompted legacy 1=Yes, 2=No mean scale were recoded as 2 − x, the single percentage-form value 35 was divided by 100, and values already in [0,1] were retained. The 17 main newsletter means were then recomputed from the corrected moderator cells. No generated prompt-cell record was manually excluded.
- **G.3 Calibration corrections** — No empirical or outcome-fitted calibration is applied. The newsletter conversion described in G.2 is deterministic scale harmonization only.

## H · Learning and conditioning components
- **H.1 Fine-tuning data** — N/A.
- **H.2 Context & retrieval corpora** — N/A.

## I · Data inputs, blinding, and competing interests
- **I.1 Competing interests ★** — N/A.
- **I.2 External human data †** — N/A; no external human dataset is used for fine-tuning, retrieval, in-context examples, calibration, or post-processing.
- **I.3 Blinding attestation ★** — The signed attestation is available in declaration.pdf in the source code repository. No team member accessed, solicited, or was shown human outcome data from this study, including pilots, before the prediction lock.
- **I.4 Contamination note †** — There is no known exposure or contamination. Qwen does not publish a precise training-data cutoff for this checkpoint. Qwen/Qwen3.6-27B was publicly released on 2026-04-22, while the benchmark condition, questionnaire, and survey materials were first released on 2026-07-21; the released checkpoint therefore predates the public benchmark materials.

## J · Internal selection procedure
- **J.1 Design-space search †** — Multiple local single-model direct-forecast runs were produced during development. Qwen/Qwen3.6-27B was selected as the latest dense Qwen production run for this Tier 2 entry. Human benchmark outcomes were unavailable, so no pipeline or model was selected against benchmark performance.

## K · Reproducibility & frozen artifacts
- **K.1 Code & materials** — https://github.com/dess-mannheim/silicon_sampling_benchmark; relevant files include the group prompt builder, questionnaire and condition inputs, launcher, JSON parser, outcome construction, fallback completion code, and the source 27B prediction files. This deposit contains predictions/team_6_T2_secondary-3_v1_cells_main.csv and predictions/team_6_T2_secondary-3_v1_cells_moderator.csv. DOI: [TBD before submission].
- **K.2 Raw output logs †** — The complete unprocessed Qwen/Qwen3.6-27B run-6501 record and its SHA-256 checksum are uploaded to https://zenodo.org/records/22124448 with DOI: 10.5281/zenodo.22124447.
- **K.3 Computational resources** — 459 local model completions and no API calls or API cost. Run 6501 used two NVIDIA H100 PCIe GPUs. The battery completed in 27 m 54 s; end-to-end time from launch through writing all artifacts was 30 m 35 s on 2026-07-23. The logs retain aggregate throughput but not exact total token counts.

## L · Disclosure class

- **A · Open** — all items public. External resources are all publicly available.
