# 🎯 Who Wants to Be a PoliMillionaire?

> NLP Project — Politecnico di Milano, A.Y. 2025/2026

**Team WeNeedMoreGPU**

| Name | Email |
|------|-------|
| Luca Bordin | luca1.bordin@mail.polimi.it |
| Mattia Menegale | mattia.menegale@mail.polimi.it |
| Marco Minaudo | marco.minaudo@mail.polimi.it |

---

## 📌 Overview

This project develops a chatbot able to autonomously compete in the **PoliMillionaire** online quiz. The quiz spans **6 categories** and can be played in two modes:

| Mode | Description |
|------|-------------|
| **Text** | Questions and options are provided as plain text |
| **Speech** | Questions and options are received as audio files, transcribed via Whisper ASR, then processed by the same text pipeline |

The system interacts with the official PoliMillionaire API to retrieve questions, submit answers, and manage game sessions.

---

## 🗂️ Project Structure

```
NLP_PROJECT_WE_NEED_MORE_GPU.ipynb   # Main notebook
millionaire_client.py                # Official quiz API client
logs_naive/                          # S1 — naive baseline logs
logs_naive_prompt_aware/             # S2 — category-aware logs
logs_speech_naive_openai_whisper_small/   # S3.1 — Whisper small logs
logs_speech_naive_normalized_answers_openai_whisper_large/  # S3.2 — Whisper large logs
logs_rag_new_thresold_per_category/  # S4 — RAG logs
logs_rag_new_thresold_per_category_test_set/  # S4 — RAG test set logs
logs_s51/ ... logs_s56/              # S5 — Math strategy logs
logs_s6_final_competition_setup/     # S6 — Final setup logs
hf_models/                           # Cached HuggingFace model weights
Heatmaps_Naive_Approach_With_Prompt/ # Saved evaluation plots
analysis/math_fewshot_candidates.json  # Math few-shot examples
```

---

## 🏗️ Pipeline Overview

| Section | Name | Description |
|---------|------|-------------|
| **S0** | Environment Setup | Dependencies, imports, Drive mounting, constants, utility functions, model loading |
| **S1** | Naive Baseline | Multi-model evaluation with a generic prompt — establishes the performance baseline |
| **S2** | Prompt Engineering | Category-specific prompts — tests whether targeted instructions improve reliability |
| **S3** | Speech Mode | ASR pipeline with Whisper; refined from `whisper-small` to `whisper-large-v3-turbo` |
| **S4** | RAG | Selective retrieval from Wikipedia and news APIs for knowledge-intensive categories |
| **S5** | Math | Six dedicated strategies for the Math competition |
| **S6** | Final Setup | Combination of the best strategy per category for competition |

---

## 🤖 Models

### LLM Backbone

| Model | HuggingFace ID | Role |
|-------|---------------|------|
| Qwen2.5-3B-Instruct | `Qwen/Qwen2.5-3B-Instruct` | Baseline evaluation |
| Qwen2.5-7B-Instruct | `Qwen/Qwen2.5-7B-Instruct` | Baseline evaluation |
| Llama-3.2-3B-Instruct | `meta-llama/Llama-3.2-3B-Instruct` | Baseline evaluation |
| Llama-3.1-8B-Instruct | `meta-llama/Llama-3.1-8B-Instruct` | Baseline evaluation |
| Gemma-2-9B-it | `google/gemma-2-9b-it` | **Final factual model** |
| AceMath-7B-Instruct | `nvidia/AceMath-7B-Instruct` | **Final Math model** |
| DeepSeek-R1-Distill-Qwen-7B | `deepseek-ai/DeepSeek-R1-Distill-Qwen-7B` | Math evaluation |
| Phi-4-mini-reasoning | `microsoft/Phi-4-mini-reasoning` | Math evaluation |

All models are loaded with **4-bit quantization** (NF4 via BitsAndBytes) to fit within GPU memory constraints.

### ASR

| Model | Role |
|-------|------|
| `openai/whisper-small` | Initial speech baseline (S3.1) |
| `openai/whisper-large-v3-turbo` | Final refined ASR setup (S3.2) |

---

## 🔍 RAG Pipeline

RAG is applied selectively to knowledge-intensive categories (**Entertainment, Ancient History, Science & Nature, Philosophy & Psychology, News**). Math is excluded since it requires computation, not factual lookup.

```
Question
   │
   ├─ Entity extraction (GLiNER + regex)
   │
   ├─ Query building (per-option + neutral queries)
   │
   ├─ Wikipedia / News retrieval
   │   ├─ Wikipedia MediaWiki API        (categories 0, 1, 2, 4)
   │   ├─ The Guardian Content API       (category 5)
   │   └─ GNews API                      (category 5, fallback)
   │
   ├─ Chunking + First-stage ranking (BM25 + dense embeddings, RRF fusion)
   │   └─ Embedder: BAAI/bge-base-en-v1.5
   │
   ├─ Reranking (BGE cross-encoder)
   │   └─ Reranker: BAAI/bge-reranker-v2-m3
   │
   ├─ Confidence gating (rerank score + option spread thresholds)
   │   ├─ Strong evidence → RAG prompt
   │   └─ Weak evidence  → Naive prompt fallback
   │
   └─ FINAL_ANSWER: <id>
```

---

## ➗ Math Strategies (S5)

| Sub-section | Strategy | Fallback |
|-------------|----------|----------|
| **S5.1** | Simple math prompt | — (global fallback) |
| **S5.2** | Tool calling (7 predefined Python tools) | S5.1 |
| **S5.3** | Free-form code generation + sandbox execution | S5.1 |
| **S5.4** | Code generation + self-review stage | S5.1 |
| **S5.5** | Function registry + dynamic code generation (28 helpers) | S5.1 |
| **S5.6** | **Few-shot CoT + category-aware routing** ✅ | S5.1 |

S5.6 classifies the math question into one of 22 sub-categories (e.g., `permutation_group_order`, `statistics_sampling_design`, `calculus_integrals_differential`) and selects the most similar few-shot examples using Jaccard similarity. It was the best-performing strategy overall.

---

## 🏆 Final Competition Setup

| Competition | Category | Strategy | Model |
|-------------|----------|----------|-------|
| 0 | Entertainment | RAG (Wikipedia) | `google/gemma-2-9b-it` |
| 1 | Ancient History | RAG (Wikipedia) | `google/gemma-2-9b-it` |
| 2 | Science & Nature | RAG (Wikipedia) | `google/gemma-2-9b-it` |
| 3 | Math | Few-Shot CoT + Routing | `nvidia/AceMath-7B-Instruct` |
| 4 | Philosophy & Psychology | Naive baseline | `google/gemma-2-9b-it` |
| 5 | News | RAG (Guardian + GNews) | `google/gemma-2-9b-it` |

Both **text mode** and **speech mode** are supported. In speech mode, the ASR step (Whisper large-v3-turbo) runs before the answering pipeline, with transcript normalization to reduce hallucinated prefixes, repetitions, and noise.

---

## ⚙️ Setup

### Requirements

The notebook runs on **Google Colab** with a GPU runtime (A100 recommended).

```bash
pip install transformers==4.51.3 accelerate>=1.3.0 bitsandbytes>=0.46.1 \
            safetensors>=0.4.5 huggingface_hub sentencepiece protobuf \
            requests rank_bm25 sentence-transformers FlagEmbedding \
            gliner nltk langchain-core librosa soundfile
```

### Google Drive

The notebook expects a base directory at:
```
/content/drive/MyDrive/NLP/NLP_ASSIGNMENT
```
Override via the `NLP_BASE` environment variable.

### Secrets (Colab `userdata`)

| Key | Purpose |
|-----|---------|
| `username` | PoliMillionaire username |
| `poli-millionaire` | PoliMillionaire password |
| `HF_TOKEN` | HuggingFace access token (for gated models) |
| `GNews_API` | GNews API key |
| `GUARDIAN_API_KEY` | The Guardian Content API key |

---

## 📊 Evaluation Metrics

- **Question-level accuracy** — fraction of correctly answered questions per model/category
- **Mean/Median stopping level** — average game level reached before a wrong answer ends the game
- **RAG ON vs OFF accuracy** — diagnostic of the retrieval gating strategy
- **Latency** — per-question elapsed time (LLM inference + ASR transcription in speech mode)

Results are saved as JSON logs and visualized as heatmaps (RdYlGn / YlGnBu colormaps).

---

## 🛠️ Development Tools

- **Codex** — implementation support and debugging
- **Claude Code** — code refactoring and experiment management
