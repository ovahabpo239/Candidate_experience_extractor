# Candidate Experience Extractor: Project Summary (for CV writing)

> Context for an AI assistant: this is a factual summary of a Python project, taken from its source code and git history. Use it to write CV bullet points. Do not invent metrics. The repo has no recorded accuracy or F1 numbers and no production-scale counts.

## One-line description
A Python NLP pipeline that pulls candidate resumes (PDF/TXT) from Azure Blob Storage, extracts text (with OCR fallback), and **anonymizes personal data with a fine-tuned XLM-RoBERTa NER model**. A GPT-4o-mini "auditor" then checks for leaked PII. The pipeline also extracts **hard skills and years of experience** per candidate.

## Timeline & team
- Active development: Dec 2025 – Jan 2026 (about 30 commits).
- 2 contributors, using a feature-branch workflow (`dev`, feature branches, merges).
  - **Daniil**: anonymization pipeline, XLM-RoBERTa extractor, GPT dataset labeling, error-mining loop, logging integration, Ollama model manager, main pipeline.
  - **BellamyCeleman**: experience-duration extractor, skills-extraction integration, repo restructuring.
  - *(Keep only the parts that were yours.)*

## Tech stack
Python 3.12 · PyTorch · Hugging Face Transformers (XLM-RoBERTa, Trainer API, token classification) · Hugging Face Datasets · Azure OpenAI (GPT-4o-mini) · Ollama (local LLM, Phi-4) · Azure Blob Storage SDK · Azure Application Insights (OpenCensus) · pdfplumber · PyMuPDF · Tesseract OCR · pandas · CoNLL/BIO tagging · python-dotenv · Cursor (AI-assisted development)

## What was built (by component)

### 1. End-to-end resume anonymization pipeline (`Main.py`)
- Streams resumes page by page from an **Azure Blob container**, with a configurable page size and start/end page.
- Extracts text, anonymizes it with the NER model (entities become `[REDACTED]`), then can **verify the output with GPT-4o-mini**. It reports PASSED/FAILED counts and a pass rate per batch.
- Has a CLI built on a small fluent argument-builder wrapper (`ConsoleArgs`).

### 2. Custom NER / anonymization model (XLM-RoBERTa)
- **Fine-tuned `xlm-roberta-base`** for token classification on multilingual resumes (English, Russian, Ukrainian). Entity types: `FULL_NAME`, `DATES`, `COMPANIES`, `HARD_SKILLS`, `LOCATIONS` (BIO scheme).
- Training pipeline uses the HF `Trainer` with fp16 on GPU, early stopping, best-model selection by F1 (seqeval-style metrics), and subword label alignment.
- Has **3 training modes**:
  - `new`: train from base.
  - `continue`: resume fine-tuning.
  - `extend`: add new entity classes to an existing model **and transfer classifier weights for the labels it already has**, so it doesn't need full retraining.
- Inference module returns entities with **character offsets** (via offset mapping), exports to JSON, and anonymizes text by span replacement.

### 3. LLM-powered automatic dataset labeling
- Built a **GPT-4o-mini (Azure OpenAI) labeler** that turns raw resume text into a **CoNLL/BIO training dataset**. This replaces manual annotation.
- Locates every occurrence of each entity in the text and converts it to token-level tags.
- Handles the robustness problems: retries, rate-limit and content-filter errors, cleanup of malformed JSON and markdown fences, and **checkpointing and resume** for long labeling runs.
- Automated collection of the raw dataset from Azure Blob Storage.

### 4. Error-mining loop to improve model accuracy
- A pipeline that runs the NER anonymizer, then uses GPT-4o-mini as a **second-opinion PII detector** on the anonymized output. It collects cases where the model **missed entities** and turns them into new training data. This is an active-learning-style feedback loop.
- Added **prompt-injection protection**: regex sanitization of instructions like "ignore previous instructions" found inside resumes, and wrapping untrusted text in `<text>` tags.

### 5. Robust document text extraction
- A PDF-to-text converter with a **3-level fallback**: pdfplumber → PyMuPDF → **Tesseract OCR at 300 DPI** for scanned resumes.
- Detects the Tesseract install automatically across Linux, macOS and Windows.

### 6. Skills & experience extraction
- **Trie (prefix-tree) skill matcher**: finds every known skill in a single pass over the text, with word-boundary checks so "SQL" doesn't match inside "MySQL".
- A skills dictionary (about 240 tech skills) scraped from a GitHub repo via the GitHub API.
- **Experience-duration calculator** that uses a **local LLM (Phi-4 via Ollama)** with few-shot prompting. It turns free-form date ranges ("Jan 2023 – Present", "01/2020 – 03/2020") into years and months. Output is a CSV of skill / years / months.
- An Ollama model manager that checks whether the model is installed locally and **pulls it if it's missing**.

### 7. Infrastructure / engineering quality
- A **thread-safe singleton logger factory** (double-checked locking) with pluggable handlers: console, file, and **Azure Application Insights**. Configured through environment variables, and integrated across all modules.
- Modular package architecture, config via `.env`, and secrets kept out of code.

## Suggested CV bullets (edit to match your role)
- Built a multilingual **resume anonymization pipeline** in Python. It ingests CVs from **Azure Blob Storage**, extracts text with an OCR fallback, and redacts PII using a **fine-tuned XLM-RoBERTa NER model**.
- Cut manual annotation by building a **GPT-4o-mini auto-labeling pipeline** that produces CoNLL/BIO training data, with retry logic, checkpointing and JSON validation.
- Designed an **LLM-as-judge feedback loop**: GPT-4o-mini audits the anonymized output, and the entities the model missed become new training data that improves recall.
- Implemented **incremental model extension**, adding new entity classes to a trained transformer while keeping the existing classifier weights.
- Built skill and experience extraction: a **Trie-based skill matcher** over a 240-skill dictionary, and a **local LLM (Phi-4/Ollama)** date-range-to-duration calculator.
- Built a thread-safe, multi-sink logging system with **Azure Application Insights** integration, used across the whole codebase.

## Honest caveats (so the CV stays accurate)
- No evaluation results (F1, accuracy, number of resumes processed) are committed to the repo. Add real numbers only if you have them from your training runs.
- It's a prototype/R&D-stage project. There are no automated tests, no CI and no deployment config.
