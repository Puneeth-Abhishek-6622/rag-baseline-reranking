# RAG Retrieval: Baseline vs. Cross-Encoder Re-Ranking

This project compares dense vector retrieval with a two-stage retrieval pipeline that re-ranks candidates using a cross-encoder. The experiment uses **D&D Basic Rules, Version 1.0 (2018)** as a single-document corpus.

> **Assignment deliverables:** submit the GitHub repository and a separate concise report. This README documents how to run the project; it is not a replacement for the report or for the answer-quality review requested in the assignment.

## Project overview

The goal is to test whether cross-encoder re-ranking improves the relevance of the three passages given to a grounded answer-generation model, and to measure the extra retrieval latency.

### Pipeline A — dense baseline

1. Extract text from the PDF while retaining 1-based PDF page numbers.
2. Split page text into overlapping chunks (1,000-character chunk size and 180-character overlap).
3. Encode chunks using `sentence-transformers/all-MiniLM-L6-v2` with normalized embeddings.
4. Build a FAISS inner-product index; with normalized vectors, inner product corresponds to cosine similarity.
5. Retrieve the top 10 candidates for each question and pass the top 3 directly to the answer-generation prompt.

### Pipeline B — dense retrieval plus re-ranking

1. Retrieve the same top 10 candidates from the dense index.
2. Score each query–chunk pair using `cross-encoder/ms-marco-MiniLM-L-6-v2`.
3. Sort the 10 candidates by cross-encoder score and select the top 3.
4. Use the same prompt structure as the baseline, changing only the selected evidence passages.


## Repository contents

A typical project layout is:

```text
.
├── README.md
├── rag_baseline_vs_reranking.ipynb   # the final notebook
├── requirements.txt
├── .env.example                      # optional Gemini configuration template
├── source.pdf                        # local source document; do not commit unless permitted
└── outputs/                          # created by the notebook
```

The uploaded notebook may have a version suffix in its filename (for example, `rag_baseline_vs_reranking(3).ipynb`). For GitHub, it is clearer to rename the final, tested copy to `rag_baseline_vs_reranking.ipynb` and update this README if you choose another name.

## Requirements

- Python **3.10–3.12**
- CPU or compatible CUDA-enabled PyTorch setup
- The project dependencies listed in `requirements.txt`
- The D&D Basic Rules PDF used by the notebook, saved locally as `source.pdf`

The notebook looks for `source.pdf` beside itself or in a `data/` subdirectory. Make sure the file is the same PDF version used for the page-level evaluation labels.

## Setup and run

### 1. Clone the repository and create an environment

```bash
git clone <YOUR_GITHUB_REPOSITORY_URL>
cd <YOUR_REPOSITORY_DIRECTORY>
python -m venv .venv
```

Activate the environment:

**Linux/macOS**

```bash
source .venv/bin/activate
```

**Windows PowerShell**

```powershell
.venv\Scripts\Activate.ps1
```

### 2. Install the dependencies

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

If you want GPU acceleration, install a PyTorch build compatible with your CUDA environment using the official PyTorch installation instructions. CPU execution is also supported, though model loading and inference may be slower.

### 3. Add the source PDF

Place your authorized copy of the assignment's source PDF in either location:

```text
source.pdf
```

or

```text
data/source.pdf
```

The PDF is not assumed to be included in the Git repository. Only distribute it if you have permission to do so.

### 4. Run the notebook

Start Jupyter:

```bash
python -m jupyter notebook
```

Open the final notebook and select **Kernel → Restart Kernel and Run All**. Let the embedding model and cross-encoder finish downloading/loading on the first run. Review the output tables and plots after execution; do not rely on numbers copied from an older run.

### 5. (Optional) Enable Gemini answer generation

The notebook can generate Answer 1 and Answer 2 for the selected failure queries. To enable it, create a local `.env` file in the project directory:

```text
GEMINI_API_KEY=your_api_key_here
GEMINI_MODEL=your_available_gemini_model_id
```

Use a model ID supported by your Gemini API account. Install `google-genai` from `requirements.txt` if needed, then restart the notebook kernel and run all cells. Without a valid API key, the retrieval evaluation still runs, but the LLM answer comparison is skipped.

**Never commit `.env`, API keys, or other credentials to GitHub.** Keep `.env` in `.gitignore`.

## Evaluation

The notebook evaluates six answerable questions (Q1–Q6) and one out-of-document question (Q7). Gold supporting pages are manually annotated against the selected PDF. Evaluation includes:

- **Recall@k** for k = 1, 3, 5, and 10, based on the annotated supporting pages
- **MRR@10** (mean reciprocal rank of the first retrieved supporting page)
- **Hit@3**
- Chunk rank changes before and after re-ranking
- Dense retrieval, re-ranking, and combined retrieval-stage latency
- An out-of-document / abstention check

Two intended failure-analysis examples are Q5 (melee/ranged ability modifiers and the finesse exception) and Q6 (when Hit Dice may be spent and the related short-rest procedure). The notebook computes the actual rank changes from the current run; inspect the outputs before quoting any rank shift in the report.

Latency figures measure retrieval and re-ranking only. They exclude LLM answer generation and may vary with hardware, model warm-up, and system load.

## Generated artifacts

The notebook creates an `outputs/` directory with reproducibility and analysis files, including:

- `faiss.index` — dense vector index
- `chunks.jsonl` — chunk text, IDs, source, and PDF page metadata
- `chunk_inventory.csv` — chunk inventory
- `evaluation_questions.json` — evaluation questions and annotated pages
- `top10_rankings.jsonl` — ranked passages and scores for each pipeline
- `rank_movements.csv` — dense and re-ranked positions
- `query_level_metrics.csv` and `aggregate_metrics.csv` — retrieval metrics
- `latency_profile.csv`, `pipeline_latency.csv`, and `pipeline_latency_summary.csv` — timing results
- `failure_case_analysis.csv` — evidence and rank shifts for the selected failure cases
- `generated_answer_comparison.csv` — generated answers, when Gemini is enabled
- `answer_review_sheet.csv` — template to document manual answer/citation review
- `run_summary.json` — compact summary of settings and results

## How to interpret results

Re-ranking can improve the ordering of passages already found by dense retrieval, but it cannot recover a relevant passage that was not included in the original top 10. A cross-encoder jointly scores the query and a passage, which can improve fine-grained relevance judgments, but evaluating many query–passage pairs costs more than comparing a query embedding with precomputed document embeddings. This is why the experiment re-ranks only a small candidate set.

Do not interpret cosine similarity or cross-encoder scores as calibrated probabilities, and do not compare their numeric values directly. Discuss retrieval metrics, answer correctness, citation support, and latency as separate outcomes.

## Reproducibility and limitations

- The experiment uses one PDF and a small set of six answerable questions; results are not a broad retrieval benchmark.
- Page-level relevance labels are only a proxy for chunk-level relevance.
- The cross-encoder only reorders the initial top 10 candidates.
- PDF text extraction can flatten tables, stat blocks, and multi-column layouts.
- Generated answers can vary by model and API version. Read Answer 1 and Answer 2, check every cited chunk/page against the retrieved evidence, and complete `outputs/answer_review_sheet.csv` before drawing conclusions.
- Re-run the complete notebook and use that run's metrics in the final report.

## Suggested report outline

1. Bi-encoder vs. cross-encoder architecture
2. Rank-shift case studies for Q5 and Q6, with chunk IDs and PDF pages
3. Qualitative comparison of Answer 1 and Answer 2, including omissions, unsupported claims, and citation quality
4. Retrieval/re-ranking latency table
5. Why cross-encoders are applied to a small candidate set rather than the full document collection
6. Limitations of the dataset, labels, and measurements
