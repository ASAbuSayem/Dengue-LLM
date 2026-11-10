# Evaluating Small Open-Weight Language Models on the National Dengue Guidelines of Bangladesh and India

**Bangla–English benchmark and offline evaluation code**

A bilingual (Bangla–English) benchmark and evaluation code for testing whether language models follow the national dengue clinical-management guidelines of Bangladesh and India. Everything runs offline on a single NVIDIA T4 GPU (16 GB VRAM), and the benchmark, notebooks and results can be reused for any model or study.

---

## What this repository contains

A multiple-choice benchmark built from the national dengue clinical-management guidelines of Bangladesh and India, asked in English and Bangla, and the notebooks that evaluate small open-weight LLMs on it, running offline on a single NVIDIA T4 GPU (16 GB VRAM).

| Research question | What is measured |
|---|---|
| RQ1 | Accuracy on Bangladesh-guideline vs India-guideline questions |
| RQ2 | Accuracy drop when the same question is asked in Bangla instead of English |
| RQ3 | On *conflict* items (the two guidelines disagree), which country's rule the model follows: Guideline-Mismatch Rate (GMR) |
| RQ4 | Zero-shot vs generic RAG (both guidelines) vs country-specific RAG |
| Deployability | GPU memory (load and peak), load time, seconds per item, output tokens/s, Bangla token inflation |

## Repository layout

```
dengue-llm-guidelines/
├── notebooks/
│   ├── NB1_Dengue_Corpus_and_Benchmark.ipynb   # guideline PDFs → page text → RAG chunks; benchmark drafting tools
│   └── NB2_LLM_Evaluation.ipynb                # retrieval, model runs, statistics, tables, figures (Kaggle run, with outputs)
├── data/
│   ├── benchmark.csv                           # final benchmark (128 items, English + Bangla)
│   └── benchmark_removed_items.csv             # items removed during checking, with reasons
├── results/                                    # final run: raw outputs, scored responses, tables, figures
├── requirements.txt
├── LICENSE                                     # code: MIT
└── DATA_LICENSE                                # benchmark: CC BY 4.0
```

The guideline PDFs and the derived text chunks (`corpus_chunks.jsonl`) are **not** redistributed here. NB1 downloads the PDFs from their official sources and rebuilds the chunks.

## Source guidelines

| Country | Document | Source |
|---|---|---|
| Bangladesh | National Guideline for Clinical Management of Dengue Syndrome, 4th ed. 2018 (DGHS) | mirror used by NB1: `https://medbox.org/index.php/dl/5e9d82924b9817284241d683` |
| India | National Guidelines for Clinical Management of Dengue Fever 2023 (NCVBDC) | `https://ncvbdc.mohfw.gov.in/Doc/National%20Guidelines%20for%20Clinical%20Management%20of%20Dengue%20Fever%202023.pdf` |

Page numbers in the benchmark (`page_BD`, `page_IN`) are **PDF page indices** (1-based, as counted by a PDF viewer), not the numbers printed on the pages.

## The benchmark (`data/benchmark.csv`)

| Set | Items | Meaning |
|---|---|---|
| BD | 67 | Answer defined by the Bangladesh guideline |
| IN | 54 | Answer defined by the India guideline |
| CONFLICT | 7 | Same question; the correct answer differs between the two guidelines (`answer_BD ≠ answer_IN`) |

Each CONFLICT item is asked twice, once with the patient in Bangladesh and once with the patient in India, so each language has 135 evaluation units (270 prompts per setting per model, 810 per model).

**Columns:** `qid, set, topic, question_en, A_en … D_en, question_bn, A_bn … D_bn, answer_BD, answer_IN, page_BD, page_IN, evidence_quote, draft_by, draft_model, verified, bn_verified, verified_by, bn_draft_by, notes`

**How the items were made:**
- Candidate items were drafted by LLMs: Gemini 2.5 Flash, Qwen2.5-7B-Instruct, and Claude. The `draft_model` column records which one.
- Each answer was checked against the text of the cited guideline page, and items whose evidence was missing, ambiguous, or contradicted elsewhere in the same guideline were removed (see `benchmark_removed_items.csv`).
- Bangla versions were translated from English; every number, dose and unit was checked against the English item.
- Answer positions were shuffled so that A/B/C/D are balanced.
- Qwen2.5-7B is both a drafter and an evaluated model; NB2 reports accuracy split by drafter (`S1_draft_bias`), and the tables mark it with †.

Only 7 genuine Bangladesh–India conflicts were found: most clinical-management text in the two guidelines is shared. Several *within-guideline* inconsistencies were found as well; they are listed in `benchmark_removed_items.csv`.

## Models evaluated

All run locally in 4-bit NF4 (bitsandbytes), one at a time, with greedy decoding.

| Tag | Hugging Face ID | Licence | Gated |
|---|---|---|---|
| `qwen2.5-7b` | `Qwen/Qwen2.5-7B-Instruct` | Apache-2.0 | no |
| `qwen2.5-3b` | `Qwen/Qwen2.5-3B-Instruct` | Qwen Research | no |
| `llama3.1-8b` | `meta-llama/Llama-3.1-8B-Instruct` | Llama 3.1 Community | yes |
| `gemma2-9b` | `google/gemma-2-9b-it` | Gemma Terms of Use | yes |

Retrieval uses `BAAI/bge-m3` (multilingual), top-4 chunks of ~300 words.

## How to run

### Requirements
- A GPU with at least 16 GB (free Colab or Kaggle T4 is enough).
- A Hugging Face access token, saved as the secret `HF_TOKEN`. Accept the licence on the model pages of Llama 3.1 and Gemma 2 first.

### Kaggle (recommended for the full run)
1. Create a Kaggle Dataset with `benchmark.csv` and `corpus_chunks.jsonl` (the latter from NB1), and add it as input.
2. Import `NB2_LLM_Evaluation.ipynb`. Settings: Accelerator **GPU T4 x2**, Internet **On**; add `HF_TOKEN` under Add-ons → Secrets.
3. Run once with `PIPELINE_TEST = True` (fake model, about 2 minutes) to check the pipeline.
4. Set `PIPELINE_TEST = False` and `RUN_MODELS = ['qwen2.5-7b']`, then **Save Version → Save & Run All**.
5. For the next model, add the previous version's output as input (results resume from it), change `RUN_MODELS`, and save a new version.
6. After all four models, the last run writes the tables and figures, zipped as `dengue_eval_results.zip`.

### Google Colab
NB1 and NB2 both mount Google Drive and keep everything under `MyDrive/DengueLLM/`. Run NB1 up to cell 3b to build the chunks, put `benchmark.csv` in `DengueLLM/data/`, then run NB2 as above.

## Outputs (NB2)

| File | Content |
|---|---|
| `raw/raw_<model>.csv` | Every prompt and raw response, with tokens, latency and GPU |
| `raw/deploy_<model>.csv` | VRAM, load time and session information |
| `tables/T1–T6, S1–S2` (CSV, LaTeX, Excel) | Accuracy, Bangla–English gap, GMR, setting effects, format failures, deployability, drafter split, retrieval quality |
| `figures/fig1–fig4, figS1` (PDF, PNG) | Accuracy by setting, Bangla gap, conflict behaviour, accuracy vs cost, retrieval recall |

Statistics: 95% confidence intervals from a cluster bootstrap over questions (2,000 resamples); paired comparisons use McNemar's test with Holm correction.

## Prompts and retrieval settings
Every prompt states the country of treatment. English user prompt (Bangla is a full translation, see NB2 cell 5):

```
The patient is being treated in <Bangladesh|India>. Follow the national dengue clinical guideline of <Bangladesh|India>.

[RAG settings only] Use the following guideline excerpts:
[Passage k | <guideline>, p. <start>-<end>]
<chunk text>

Question: <question>
A. ... B. ... C. ... D. ...

Reply with only the letter of the correct option (A, B, C or D) in the form "Answer: X".
```

- System message: answer according to the national dengue clinical guideline of the country of treatment.
- Retrieval: `BAAI/bge-m3` dense embeddings (max 512 tokens, L2-normalised, cosine similarity); query = question + four options; top-4 chunks of ~300 words with 50-word overlap.
- Generic RAG searches all 163 chunks; country RAG searches only the chunks of the country of treatment.
- Models: `Qwen/Qwen2.5-7B-Instruct`, `Qwen/Qwen2.5-3B-Instruct`, `meta-llama/Llama-3.1-8B-Instruct`, `google/gemma-2-9b-it`; 4-bit NF4 (bitsandbytes), float16 compute (float32 for Gemma-2), greedy, `max_new_tokens=24`, batch 1, seed 42.

## Reproducibility
- Seed 42 everywhere; greedy decoding (`do_sample=False`); `max_new_tokens=24`; batch size 1.
- Responses are saved after every item; re-running a cell skips finished items.
- Answers are parsed again at analysis time, so a parser change needs no re-run.
- The GPU name is stored with every response; compare timing only across runs on the same GPU type.

## Main results (country-specific RAG, Kaggle T4)

| Model | Accuracy EN | Accuracy BN | Peak VRAM (GB) |
|---|---|---|---|
| Qwen2.5-7B | 94.1 | 84.4 | 8.63 |
| Qwen2.5-3B | 93.3 | 73.3 | 3.86 |
| Llama-3.1-8B | 96.3 | 87.4 | 8.90 |
| Gemma-2-9B | 94.1 | 92.6 | 11.36 |

Without retrieval, accuracy is 55.6–63.0% (EN) and 48.1–60.0% (BN). Full tables are in `results/tables/`.

## Licence
- Code: MIT (see `LICENSE`).
- Benchmark (`data/`): CC BY 4.0 (see `DATA_LICENSE`). Short evidence quotes from the guidelines are included only to document each answer.
- The guidelines themselves remain the property of their issuing government bodies.

## Disclaimer
This benchmark is a research artefact. It is not clinical guidance; always follow the current official guideline.

## Using this repository
- **Evaluate another model:** add it to `HF_MODELS` in NB2 (cell 2), copy one of the model-run cells with the new tag, and put the tag in `RUN_MODELS`; new results are appended to the existing ones.
- **Use only the benchmark:** `data/benchmark.csv` is self-contained (question, options, answer per country, guideline page and evidence sentence, in English and Bangla).
- **Re-analyse without a GPU:** the statistics and figure cells in NB2 read `results/scored_responses.csv` and run in seconds.
