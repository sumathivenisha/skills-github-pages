---
layout: post
title: "Hybrid Scoring Mastery: BM25 + Vector Search + LLM Rubrics"
date: 2025-07-20 10:30:00 +0400
categories: [AI, RAG, Ranking]
tags: [hybrid-search, bm25, vector-ranking, scoring-engine]
author: Sumathi Malikarjun
excerpt: Vector search alone misses exact matches. Keyword search alone misses semantic similarity. The real power is in combining them intelligently. Here's how.
---

## The Single-Signal Problem

Most RAG systems commit to one ranking strategy:

**Team A says:** "Use vector search, it's semantic!"  
**Team B says:** "Use keyword search, it's reliable!"  
**Reality says:** "You need both."

<!--more-->

## Why Each Fails Alone

### Vector Search Alone

```
Query: "Python AWS certification"

Results:
1. Resume: "AWS Certified Solutions Architect. 5 years infrastructure."
   → High semantic similarity (AWS + architecture = infrastructure)
   
2. Resume: "Python developer, no AWS experience, certified in Java"
   → Low semantic similarity (Python mentioned, but no AWS)
   
Problem: The second candidate has Python + Certificate, but ranks last.
```

### Keyword Search Alone

```
Query: "Python AWS certification"

Results:
1. Resume: "AWS certified. Python. Certified."
   → Exact keyword matches, ranks first
   
2. Resume: "Infrastructure-as-Code enthusiast. Built 50+ AWS systems in Python"
   → No exact "certified" keyword, ranks lower
   
Problem: Keyword spamming ranks above genuine expertise.
```

### Hybrid Search

```
Combined signal:
1. Resume with "AWS Certified" + strong infrastructure background
   Vector: 0.85, Keyword: 0.95 → Hybrid: 0.88 (First)

2. Resume with "Python + AWS certified" but weak AWS background
   Vector: 0.72, Keyword: 0.85 → Hybrid: 0.76 (Second)

3. Resume with "Infrastructure-as-Code Python/AWS" but not certified
   Vector: 0.88, Keyword: 0.60 → Hybrid: 0.78 (Close third)
```

**Hybrid respects both semantic quality and exact requirements.**

---

## Building a Production Hybrid Scoring Engine

### Architecture Overview

```
┌─ Query Input
│
├─ Vector Search Path
│  ├─ Embed query
│  ├─ Search vectors
│  └─ Get similarity scores (0-1)
│
├─ Keyword Search Path
│  ├─ Parse query keywords
│  ├─ BM25 scoring
│  └─ Get keyword scores (0-1)
│
├─ Deterministic Filtering
│  └─ Boolean requirements (pass/fail)
│
├─ Hybrid Combination
│  ├─ Normalize scores (both 0-1)
│  ├─ Apply weights
│  └─ Calculate final score
│
├─ LLM-Based Reranking (Optional)
│  ├─ Apply business logic to top-50
│  └─ Get business-context scores
│
└─ Output: Ranked Results
```

---

## Step 1: Vector Search Scoring

```python
class VectorSearchScorer:
    """
    Semantic similarity scoring using embeddings.
    """
    
    def __init__(self, embedding_model, vector_db):
        self.embedding_model = embedding_model  # text-embedding-3-small
        self.vector_db = vector_db  # Supabase pgvector
    
    async def score_candidates(self, query, candidates, top_k=200):
        """
        Get semantic similarity scores for all candidates.
        """
        
        # Embed the query
        query_embedding = await self.embedding_model.embed(query)
        
        # Vector search (fast, approximate)
        results = await self.vector_db.search(
            query_embedding=query_embedding,
            candidate_ids=[c.id for c in candidates],
            distance_metric="cosine",
            limit=top_k
        )
        
        # Normalize to 0-1 (cosine returns -1 to 1)
        scores = {}
        for result in results:
            # cosine distance: -1 (opposite) to 1 (identical)
            # Normalize to 0-1
            normalized_score = (result.distance + 1) / 2
            scores[result.candidate_id] = normalized_score
        
        return scores
```

**Key insight:** Vector search is *approximate*. It's fast, but not perfect. Use it to reduce search space, not as final ranking.

---

## Step 2: BM25 Keyword Scoring

BM25 is the industry standard for keyword ranking (used by Elasticsearch, Solr, etc).

```python
from rank_bm25 import BM25Okapi

class BM25Scorer:
    """
    Keyword relevance scoring using BM25.
    Best-of-breed for keyword ranking.
    """
    
    def __init__(self):
        self.k1 = 1.5  # Term frequency saturation (tuning parameter)
        self.b = 0.75  # Document length normalization
    
    async def score_candidates(self, query, candidates):
        """
        Score candidates using BM25.
        """
        
        # Tokenize query and documents
        query_tokens = self.tokenize(query)
        candidate_docs = [
            self.tokenize(self.candidate_to_text(c))
            for c in candidates
        ]
        
        # Initialize BM25
        bm25 = BM25Okapi(candidate_docs, k1=self.k1, b=self.b)
        
        # Score each candidate
        scores = {}
        bm25_scores = bm25.get_scores(query_tokens)
        
        # Normalize to 0-1
        max_score = max(bm25_scores) if bm25_scores else 1.0
        min_score = min(bm25_scores) if bm25_scores else 0.0
        range_score = max_score - min_score if max_score != min_score else 1.0
        
        for i, candidate in enumerate(candidates):
            normalized = (bm25_scores[i] - min_score) / range_score if range_score > 0 else 0.5
            scores[candidate.id] = normalized
        
        return scores
    
    def tokenize(self, text):
        """
        Simple tokenization: lowercase, split, remove stopwords.
        """
        import re
        stopwords = {"the", "a", "an", "and", "or", "is", "in", "of"}
        
        tokens = re.findall(r'\b\w+\b', text.lower())
        return [t for t in tokens if t not in stopwords]
    
    def candidate_to_text(self, candidate):
        """
        Convert candidate object to searchable text.
        """
        return f"{candidate.name} {candidate.skills} {candidate.experience} {candidate.summary}"
```

**Key insight:** BM25 favors documents with query terms. It's deterministic and explainable.

---

## Step 3: Intelligent Weighting

Don't hardcode weights. Learn them from your data:

```python
class AdaptiveWeighting:
    """
    Different queries benefit from different weights.
    """
    
    def determine_weights(self, query):
        """
        Route to appropriate weighting strategy based on query type.
        """
        
        query_lower = query.lower()
        
        # Strategy 1: Exact match queries
        if any(keyword in query_lower for keyword in ["certified", "expert", "proven"]):
            # Keyword-heavy query → boost keyword scoring
            return {"vector": 0.5, "keyword": 0.5}
        
        # Strategy 2: Semantic/conceptual queries
        if any(keyword in query_lower for keyword in ["passionate about", "interested in", "strong in"]):
            # Semantic-heavy query → boost vector scoring
            return {"vector": 0.8, "keyword": 0.2}
        
        # Strategy 3: Mixed queries (default)
        return {"vector": 0.7, "keyword": 0.3}

class HybridScorer:
    """
    Combine vector + keyword scores.
    """
    
    def __init__(self):
        self.weighting = AdaptiveWeighting()
    
    async def hybrid_score(self, query, candidates):
        """
        Combine vector and keyword scores intelligently.
        """
        
        # Get both scores
        vector_scores = await self.vector_scorer.score_candidates(
            query, candidates
        )
        keyword_scores = await self.bm25_scorer.score_candidates(
            query, candidates
        )
        
        # Determine weights
        weights = self.weighting.determine_weights(query)
        
        # Combine
        hybrid_scores = {}
        for candidate in candidates:
            vector = vector_scores.get(candidate.id, 0.0)
            keyword = keyword_scores.get(candidate.id, 0.0)
            
            hybrid = (
                weights["vector"] * vector +
                weights["keyword"] * keyword
            )
            
            hybrid_scores[candidate.id] = {
                "vector": vector,
                "keyword": keyword,
                "hybrid": hybrid,
                "weights": weights
            }
        
        return hybrid_scores
```

---

## Step 4: Deterministic Pre-Filtering

Before ranking, filter aggressively:

```python
class DeterministicFilter:
    """
    Boolean requirements that candidates must meet.
    Dramatically reduces ranking scope.
    """
    
    async def filter(self, candidates, job_requirements):
        """
        Pass/fail filtering based on absolute requirements.
        """
        
        filtered = []
        
        for candidate in candidates:
            # Check 1: Minimum experience
            if candidate.years_of_experience < job_requirements.min_years:
                candidate.filter_reason = f"Only {candidate.years_of_experience} years, need {job_requirements.min_years}"
                continue
            
            # Check 2: Required skills (at least one)
            if not self.has_any_required_skill(
                candidate.skills,
                job_requirements.required_skills
            ):
                candidate.filter_reason = "Missing all required skills"
                continue
            
            # Check 3: Forbidden skills (conflict)
            if self.has_forbidden_skill(
                candidate.skills,
                job_requirements.forbidden_skills
            ):
                candidate.filter_reason = "Has conflicting technology"
                continue
            
            # Check 4: Location (if strict)
            if job_requirements.location_strict and \
               candidate.location not in job_requirements.acceptable_locations:
                candidate.filter_reason = f"Location {candidate.location} not acceptable"
                continue
            
            # Passed all filters
            filtered.append(candidate)
        
        return filtered
```

**Impact:** Search scope reduced by 70-90%, costs drop, speed improves.

---

## Step 5: LLM-Based Reranking (Optional but Powerful)

Vector + Keyword scores are mathematical. Business logic is semantic.

```python
class LLMReranker:
    """
    Use Claude to apply business-specific scoring.
    Only applied to top-50 (token efficiency).
    """
    
    async def rerank_top_candidates(self, query, top_candidates, job_requirements):
        """
        Have Claude score based on business criteria.
        """
        
        prompt = f"""
You are an expert recruiter evaluating candidates.

Job Requirements:
{job_requirements}

Query:
{query}

Candidates (top vector+keyword matches):
{json.dumps([c.to_dict() for c in top_candidates[:50]])}

Score each candidate 0-100 using this rubric:
- Technical alignment (0-30): Does their skill set match?
- Experience relevance (0-25): Is their background relevant?
- Growth potential (0-20): Will they grow in this role?
- Cultural fit (0-15): Do values align?
- Leadership (0-10): Can they lead or mentor?

Return JSON with {{
    "candidate_id": "...",
    "score": 0-100,
    "reasoning": "...",
    "red_flags": [...],
    "recommendation": "strong_hire|hire|maybe|pass"
}}
"""
        
        response = await claude.messages.create(
            model="claude-3-5-sonnet",
            max_tokens=4000,
            messages=[{"role": "user", "content": prompt}]
        )
        
        llm_scores = json.loads(response.content[0].text)
        
        return {
            "query": query,
            "top_candidates": top_candidates,
            "llm_rankings": llm_scores
        }
```

---

## Step 6: Final Ranking

Combine all signals:

```python
class FinalRanking:
    """
    Combine vector + keyword + LLM scores for final ranking.
    """
    
    async def rank_all_signals(self, query, candidates):
        """
        End-to-end ranking combining all scoring methods.
        """
        
        # Step 1: Deterministic filtering
        filtered = await self.deterministic_filter.filter(
            candidates,
            query.requirements
        )
        
        # Step 2: Hybrid scoring (vector + keyword)
        hybrid_scores = await self.hybrid_scorer.hybrid_score(
            query.text,
            filtered
        )
        
        # Step 3: Sort by hybrid score
        hybrid_ranked = sorted(
            [
                (c, hybrid_scores[c.id]["hybrid"])
                for c in filtered
            ],
            key=lambda x: x[1],
            reverse=True
        )
        
        top_50 = [c for c, _ in hybrid_ranked[:50]]
        
        # Step 4: LLM reranking (expensive, only top-50)
        llm_rankings = await self.llm_reranker.rerank_top_candidates(
            query.text,
            top_50,
            query.requirements
        )
        
        # Step 5: Final results combining all signals
        final_results = []
        for candidate in hybrid_ranked:
            candidate_id = candidate[0].id
            hybrid_score = candidate[1]
            
            # Find LLM score if available
            llm_score = None
            for llm_result in llm_rankings:
                if llm_result["candidate_id"] == candidate_id:
                    llm_score = llm_result["score"] / 100  # Normalize to 0-1
                    break
            
            # Combine: 60% hybrid, 40% LLM (if available)
            if llm_score:
                final_score = 0.6 * hybrid_score + 0.4 * llm_score
            else:
                final_score = hybrid_score
            
            final_results.append({
                "candidate": candidate[0],
                "hybrid_score": hybrid_score,
                "llm_score": llm_score,
                "final_score": final_score
            })
        
        return sorted(
            final_results,
            key=lambda x: x["final_score"],
            reverse=True
        )
```

---

## Real Numbers from Production

Using this system on recruitment matching:

```
Baseline (Vector Only):
- Accuracy (vs human evals): 71%
- False Positives: 18%
- False Negatives: 11%

Vector + Keyword (Hybrid):
- Accuracy: 89%
- False Positives: 4%
- False Negatives: 7%

Hybrid + LLM Reranking (Top-50):
- Accuracy: 94%
- False Positives: 2%
- False Negatives: 4%
- Cost increase: $0.08/query (worth it)
```

---

## Tuning Hybrid Weights

Don't guess. Measure:

```python
async def a_b_test_weights(query_samples, candidate_pool):
    """
    Test different weight combinations against real queries.
    """
    
    weight_configs = [
        {"vector": 0.5, "keyword": 0.5},  # Equal
        {"vector": 0.7, "keyword": 0.3},  # Vector-heavy
        {"vector": 0.6, "keyword": 0.4},  # Balanced
        {"vector": 0.8, "keyword": 0.2},  # Very vector-heavy
    ]
    
    results = {}
    
    for config in weight_configs:
        accuracy = 0
        
        for query in query_samples:
            # Rank candidates using this config
            scores = await hybrid_scorer.score_with_weights(
                query,
                candidate_pool,
                weights=config
            )
            
            ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)
            top_candidate = ranked[0][0]
            
            # Check against ground truth (human eval)
            if top_candidate == query.expected_best_match:
                accuracy += 1
        
        results[str(config)] = accuracy / len(query_samples)
    
    return results
```

---

## Key Takeaways

1. **Never use single-signal ranking.** Vector + Keyword catches more matches.
2. **Normalize scores to 0-1.** Only then can you weight fairly.
3. **Deterministic filtering reduces scope massively.** Pre-filter before ranking.
4. **BM25 is the gold standard for keyword scoring.** Use proven algorithms.
5. **Adaptive weighting beats fixed weights.** Different queries need different emphasis.
6. **LLM reranking adds business logic.** Use it on top results only.
7. **Measure and tune weights.** A/B test against real data, not guesses.
8. **Cost-quality trade-offs matter.** LLM reranking is expensive; use it strategically.

---

## What I'm Exploring Next

- **Learning-to-rank models** — Can we train a model to predict good weights?
- **Query-specific weighting** — Dynamic weights based on query analysis?
- **Multi-stage ranking** — Coarse → medium → fine grained filtering?
- **Embedding + Keyword fusion** — Earlier fusion vs. score fusion?

---

**Built hybrid ranking systems?** Curious about your weight tuning approach? [LinkedIn](https://linkedin.com/in/sumathi-m-5198271a0).

---

*Hybrid scoring system engineered over 18 months of production RAG work. Open-sourcing components soon.*
