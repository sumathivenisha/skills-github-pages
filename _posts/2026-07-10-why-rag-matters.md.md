---
layout: post
title: "Why RAG Matters: Beyond Simple Vector Search"
date: 2025-07-10 10:00:00 +0400
categories: [AI, RAG, LLMs]
tags: [rag, vector-search, production-systems]
author: Sumathi Malikarjun
excerpt: Moving from toy RAG demos to production systems requires thinking beyond embeddings. Here's what I learned building a real-world candidate-matching RAG pipeline.
---

## The RAG Mirage

When I started exploring Retrieval-Augmented Generation (RAG) last year, the demos looked magical. Throw an embedding model at your documents, spin up a vector database, and suddenly your LLM has context. Ship it to production... and watch it fail catastrophically on edge cases.

<!--more-->

### The Naive Approach

Most RAG tutorials follow this flow:
1. Chunk documents
2. Generate embeddings
3. Vector search on query
4. Send top-k to LLM

Simple. Clean. **Incomplete.**

### Real-World Complexity

Building the **HR Recruitment RAG System** taught me what actually matters:

#### 1. **Hybrid Scoring Beats Pure Vector Search**

Semantic similarity alone misses keyword-heavy matches. I engineered a hybrid engine:
- **70% LLM-based rubric scoring** — 7 criteria, 100-point scale
- **30% BM25 keyword matching** — catches exact skill matches
- **Result:** Better candidate ranking, fewer false positives

```python
# Pseudo-code: Hybrid Scoring
hybrid_score = (0.7 * llm_rubric_score) + (0.3 * bm25_score)
ranked_candidates = sorted(candidates, key=hybrid_score, reverse=True)
```

#### 2. **Deterministic Pre-Filtering Saves Token Costs**

Before running expensive LLM scoring, filter deterministically:
- Years of experience (threshold)
- Required skills (exact match)
- Location (if critical)
- Education level (boolean check)

Then vector search only the filtered set. **Reduced token spend by 65%.**

#### 3. **Semantic Chunking > Fixed-Size Chunks**

Splitting resumes by fixed 512-token windows shatters context. Instead:
- Identify natural boundaries (experience sections, skills)
- Preserve semantic meaning
- Embed complete, coherent chunks
- Easier for LLM to reason over

#### 4. **Error Handling & Observability Are Non-Negotiable**

Production RAG fails silently:
- What if embeddings timeout?
- What if vector search returns 0 results?
- What if LLM fails mid-evaluation?

Built an automated error-handling pipeline with:
- Retry logic with exponential backoff
- Fallback to keyword search if semantic fails
- Email alerts + logging dashboard
- Weekly audits of failed queries

### Lessons Learned

✅ **Hybrid > Pure Vector Search** — Combine semantic + lexical  
✅ **Deterministic Pre-Filter** — Save costs, improve quality  
✅ **Semantic Chunking** — Preserve meaning, not just tokens  
✅ **Observability First** — Know when and why it fails  
✅ **Evaluate Ruthlessly** — A/B test scoring weights with real data  

### The Stack I Used

- **N8N** — Orchestration & workflow automation
- **Azure OpenAI GPT-4.1** — Embeddings & LLM rubric
- **Supabase pgvector** — Vector storage & hybrid search
- **Python** — Custom scoring & chunking logic

### What's Next

I'm exploring:
- Adaptive chunking based on query complexity
- Multi-modal RAG (CVs with images, charts)
- Real-time embedding updates
- Cost-optimized re-ranking strategies

---

**Have you built RAG systems that went sideways?** I'd love to hear your edge cases and workarounds. Drop a comment or reach out on [LinkedIn](https://linkedin.com/in/sumathi-m-5198271a0).

---

*This post is based on my experience building the HR Recruitment RAG System (2025-2026). Code snippets available on GitHub soon.*
