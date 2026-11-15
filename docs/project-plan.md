---
title: KVTP Source SLM Engineering Plan and Human RL Roadmap
description: Four-milestone technical plan covering dataset curation, human preference alignment with KTO and DPO, RAFT grounding, and deterministic verification
author: Hrishik Sancheti
ms.date: 2026-11-15
ms.topic: reference
keywords: [slm, dpo, kto, rlhf, raft, gujarati, evaluation, shrimad-rajchandra]
estimated_reading_time: 12
---

## Architectural Principles

Building a spiritual question-answering system where original texts serve as the strict source of truth requires separating knowledge storage from linguistic reasoning:

1. Retrieval stores the truth by fetching verbatim Gujarati, Sanskrit, and Prakrit passages indexed by work, *Patrank* (letter number), paragraph, or *Gatha* (verse number).
2. Fine-tuning shapes comprehension so the Small Language Model (SLM) translates 19th-century literary Gujarati accurately, preserves non-negotiable philosophical terms, cites retrieved verses, and abstains when retrieved passages do not answer the user's question.
3. Human preference alignment refines subtle philosophical nuance where two grammatically valid English translations carry very different spiritual implications.
4. Deterministic verification enforces zero fabricated quotes by checking every emitted Gujarati citation character-for-character against the retrieved source record.

---

## Why Base Model Tokenization Matters

English-centric small models fragment Gujarati Unicode characters into multiple byte-level tokens, inflating sequence length, increasing latency, and degrading translation quality.

* `Gemma 3 4B` and `Sarvam-Translate` (built on `Gemma 3 4B` for 22 Indian languages) provide high-fertility multilingual tokenization across Gujarati, Devanagari, and Latin scripts.
* At 4-bit quantization (`QLoRA`), a 4B parameter model requires approximately 2.5 to 3.5 GB of VRAM for inference and 8 to 12 GB of VRAM for training with `Unsloth`, enabling reproducible runs on a single consumer GPU or free cloud notebook.
* Before training in Milestone 1, run a tokenizer fertility benchmark measuring average tokens per verse across candidate models (`Gemma 3 4B`, `Sarvam-Translate`, and `Qwen3.5-4B`) on all 142 *Gathas* of *Atmasiddhi Shastra*.

---

## Milestone 1: Atmasiddhi Baseline, Glossary, and Human Preference RL

### Scope and Objective

Start with a bounded, verifiable corpus: the 142 *Gathas* of *Atmasiddhi Shastra* alongside a 50-term canonical glossary. Prove that a 4B open-weight model aligned with human domain feedback outperforms zero-shot frontier models on terminology preservation and philosophical fidelity.

### Why Offline Human RL Fits Milestone 1 Better Than Online PPO

Training a separate reward model with Proximal Policy Optimization (`PPO`) requires thousands of labeled comparisons; on a specialized 142-verse corpus, a neural reward model quickly overfits surface length or formatting rather than doctrinal accuracy.

Offline preference optimization methods solve this directly from small, high-signal human batches:

* Kahneman-Tversky Optimization (`KTO`) learns directly from binary human thumbs-up / thumbs-down ratings on individual model answers, making every single review by a knowledgeable friend immediately usable even before paired rewrites exist.
* Direct Preference Optimization (`DPO`) learns from paired `(prompt, chosen, rejected)` responses, pushing probability mass away from subtle mistranslations (such as translating *Swabhav* as "personality" instead of "intrinsic soul-nature") without instantiating a separate reward model.

### Step-by-Step Milestone 1 Workflow

1. Digitize and normalize the 142 public-domain Gujarati *Gathas* of *Atmasiddhi Shastra* with verse numbers, meter metadata, and transliteration.
2. Author 50 canonical glossary entries conforming to [glossary_entry.schema.json](../data/schemas/glossary_entry.schema.json), explicitly listing approved English explanations and forbidden flattening synonyms.
3. Construct a 60-question "Mistranslation Trap" evaluation set covering six core themes of *Atmasiddhi* (existence of soul, permanence, agency of karma, liberation, *Punya* / *Paap*, and *Sadguru* / *Samkit*), plus 15 out-of-scope questions that require abstention.
4. Generate `k = 3` candidate responses per prompt from the base `Gemma 3 4B` / `Sarvam-Translate` model at temperatures `0.2`, `0.6`, and `0.9` with retrieved *Gathas* and glossary context.
5. Collect human feedback through a lightweight Gradio or Argilla review card shared with 3 to 5 knowledgeable practitioners:
   * Reviewers mark individual candidates as faithful or distorted (`KTO` signal).
   * Reviewers select the better of two candidates or lightly edit the winning explanation to create gold `(chosen, rejected)` pairs conforming to [preference_pair.schema.json](../data/schemas/preference_pair.schema.json).
   * Reviewers tag each rejected output with a failure taxonomy (`conceptual_flattening`, `tradition_drift`, `archaic_gujarati_misread`, `unsupported_inference`, or `missing_abstention`).
6. Train a `QLoRA` adapter in two lightweight stages using Hugging Face `TRL` and `Unsloth`:
   * Stage 1A runs short Supervised Fine-Tuning (`SFT`) on ~150 curated verse translations and Q&A examples to lock in the output structure.
   * Stage 1B runs `KTO` (if unpaired ratings dominate) or `DPO` with `beta = 0.1` to `0.2` on 150 to 300 human preference records to sharpen doctrinal precision.

### Milestone 1 Exit Metrics

| Metric                          | Evaluation Method                                                       | Target Threshold                  |
|---------------------------------|-------------------------------------------------------------------------|-----------------------------------|
| Glossary Term Fidelity          | Deterministic regex and AST check against forbidden flattening synonyms | >= 95% compliance on held-out set |
| Citation Integrity              | Exact substring match of quoted Gujarati *Gatha* against source JSON    | 100% (hard post-generation gate)  |
| Blind Human Preference Win-Rate | Side-by-side blind review by friends: aligned 4B vs zero-shot base SLM  | >= 70% win-rate for aligned 4B    |
| Abstention Precision            | Out-of-scope prompt suite where *Atmasiddhi* has no direct statement    | >= 90% explicit abstention        |

---

## Milestone 2: Structured Output Enforcement and Iterative Alignment

### Scope and Objective

Expand the corpus to shorter poetic compositions (*Apoorva Avsar* and selected *padas*), expand the glossary to 150 terms, and harden the runtime pipeline so malformed citations or schema violations cannot occur.

### Engineering Deliverables

1. Integrate grammar-constrained decoding (`Outlines` or `XGrammar`) so every SLM output conforms to a strict response schema:
   * `cited_ids`: array of retrieved verse identifiers
   * `verbatim_gujarati`: array of exact source lines
   * `word_breakdown`: key Gujarati / Sanskrit terms with glossary-aligned meanings
   * `faithful_translation`: English or Hindi rendering of the cited passage
   * `grounded_explanation`: direct response to the seeker's question
   * `abstention_flag`: boolean indicating insufficient textual support
2. Build a `pytest` regression suite wired into GitHub Actions that runs deterministic schema, citation, and glossary checks on sample outputs whenever prompts or adapter configs change.
3. Run Round-2 iterative `DPO` on hard failure cases surfaced during Milestone 1 friend testing.

---

## Milestone 3: Scaling to Vachanamrut with RAFT and Verifiable Rewards

### Scope and Objective

Scale from structured poetry to prose letters and discourses in *Shrimad Rajchandra Vachanamrut* (hundreds of *Patranks* written to different seekers across varying contexts).

### Retrieval-Augmented Fine-Tuning Design

Standard RAG struggles when 5 retrieved letters share overlapping vocabulary (*vairagya*, *agnya*, *kashaya*) while addressing different seeker situations.

1. Chunk *Vachanamrut* along natural structural boundaries (*Patrank* metadata, recipient context when public-domain, date, and paragraph) rather than arbitrary token counts.
2. Build a hybrid retriever combining lexical search (`BM25` over Gujarati script and transliterated terms) with multilingual dense embeddings (`BAAI/bge-m3` or `IndicBERT` / `Gemma` embeddings) plus cross-encoder reranking.
3. Train the model using Retrieval-Augmented Fine-Tuning (`RAFT`):
   * Each training prompt includes 1 to 2 gold *Patrank* passages mixed with 3 to 4 hard-negative distractor passages retrieved by `BM25`.
   * Include 20% "oracle-missing" examples containing only distractors so the model learns to abstain rather than force an answer from tangentially related letters.
4. Optional RL extension with Group Relative Policy Optimization (`GRPO`): use deterministic reward functions (exact quote verification, valid *Patrank* citation ID, glossary term compliance, and format validity) as automated verifiable rewards alongside human `DPO` pairs.

---

## Milestone 4: Quantized Serving and Study-Group Companion

### Scope and Objective

Package the aligned SLM and verified retrieval pipeline into a fast, low-cost application usable by friends during personal reflection or *swadhyay* (group study).

### Engineering Deliverables

1. Export 4-bit quantized weights (`AWQ` for `vLLM` server deployment and `GGUF` for local laptop / CPU execution via `llama.cpp`).
2. Build a clean mobile-friendly Progressive Web App (PWA) presenting three transparent layers on every answer:
   * Original Gujarati *Patrank* or *Gatha* text with deep link to the full letter or verse context
   * Key term breakdown highlighting preserved philosophical terms
   * Grounded explanation in English, Hindi, or Gujarati
3. Embed a continuous human feedback loop directly in the UI: every thumbs-down or "flag translation nuance" submission writes a new candidate record to a staging queue for periodic human review and `DPO` / `KTO` adapter refreshes.
4. Instrument request latency, token throughput, cache hit rates, and abstention rates via OpenTelemetry.

---

## Risks and Mitigations

| Risk                                                      | Impact                                              | Mitigation                                                                            |
|-----------------------------------------------------------|-----------------------------------------------------|---------------------------------------------------------------------------------------|
| Subtle doctrinal drift during synthetic data generation   | Teaches the SLM plausible but inaccurate philosophy | Restrict frontier teacher usage to formatting; require human sign-off on meanings     |
| OCR noise in scanned 19th-century Gujarati print editions | Corrupts retrieval and verbatim quote verification  | Normalize Unicode (`NFC`) and proofread *Atmasiddhi* + top *Patranks* before SFT      |
| Catastrophic forgetting of general English fluency        | Degrades explanation readability after QLoRA tuning | Use low-rank adapters (`r = 16` or `32`), low learning rates, and `DPO` `beta >= 0.1` |
| Copyright contamination from modern ashram publications   | Violates licensing and open-source distribution     | Audit every ingested file source; store provenance metadata per corpus document       |
