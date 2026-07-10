---
layout: post
title: "Prompt Engineering for Production: Techniques That Actually Work"
date: 2025-07-16 11:00:00 +0400
categories: [AI, LLMs, Prompt-Engineering]
tags: [claude, prompting, production, optimization]
author: Sumathi Malikarjun
excerpt: Most prompt engineering advice is theater. Here are the techniques I use in production systems to get reliable, repeatable LLM outputs at scale.
---

## The Prompt Engineering Paradox

Everyone agrees prompting matters. But we treat it like an art, not engineering.

You get Claude to write perfect code with a thoughtful prompt. Tomorrow, it hallucinates. You tweak the prompt. It works again. Rinse, repeat.

This is **not** production-grade. Production needs:
- **Repeatable results** (same input → same output structure)
- **Measurable quality** (you can test it)
- **Cost optimization** (fewer tokens for same output)
- **Failure detection** (when it's wrong, you know)

<!--more-->

## Technique 1: The System + User Pattern

Most people treat prompts as a single text block. Structure them instead:

```python
class LLMRequest:
    def __init__(self, task, user_input):
        self.messages = [
            {
                "role": "system",
                "content": self.system_prompt(task)
            },
            {
                "role": "user",
                "content": self.user_prompt(user_input)
            }
        ]
    
    def system_prompt(self, task):
        """Stable, reusable instructions for this task."""
        return f"""
You are an expert recruiter evaluating candidates for technical roles.

Your job: Score candidates on a 0-100 scale using the rubric below.
Be harsh but fair. Prefer specificity over generosity.

SCORING RUBRIC:
- Technical Skills (0-20): Does their experience match job requirements?
- Leadership (0-15): Have they managed people or projects?
- Communication (0-15): Can they explain complex ideas clearly?
- Problem-Solving (0-20): Do they solve novel problems?
- Culture Fit (0-15): Values alignment?
- Overall Potential (0-15): Will they grow in this role?

CONSTRAINTS:
- Scores must be integers 0-100
- Always explain your reasoning
- Flag any red flags or concerns
- Return valid JSON only, no markdown
"""
    
    def user_prompt(self, user_input):
        """Variable input, always the same format."""
        return f"""
Candidate Resume:
{user_input['resume']}

Job Description:
{user_input['job_description']}

Required Skills (must-haves):
{', '.join(user_input['required_skills'])}

Score this candidate. Return JSON:
{{
    "score": <0-100>,
    "technical_skills_score": <0-20>,
    "leadership_score": <0-15>,
    "communication_score": <0-15>,
    "problem_solving_score": <0-20>,
    "culture_fit_score": <0-15>,
    "potential_score": <0-15>,
    "reasoning": "<explain your scoring>",
    "red_flags": [<list any concerns>],
    "recommendation": "hire|maybe|reject"
}}
"""
```

**Why this works:**
- System prompt is stable (don't change it every day)
- User prompt is parametrized (change only the variables)
- Claude learns the pattern, executes consistently
- You can test/benchmark against system prompts

---

## Technique 2: Chain-of-Thought Forcing

Don't ask Claude for the answer directly. Make it show its work:

### Bad Prompt
```
Evaluate if this candidate is qualified for the role.
Response: Yes/No
```

Claude will often guess wrong without showing reasoning.

### Good Prompt
```
Evaluate if this candidate qualifies.

Work through this step-by-step:
1. List the required skills (from job description)
2. For each required skill, find evidence in the resume
3. Rate each skill match: High/Medium/Low
4. Check for red flags (employment gaps, location, etc.)
5. Make a final recommendation based on steps 1-4

Format your response as JSON:
{
    "required_skills": [...],
    "skill_matches": {
        "skill1": {"found": true, "evidence": "...", "rating": "High"}
    },
    "red_flags": [...],
    "overall_fit": "High/Medium/Low",
    "recommendation": "hire|maybe|reject"
}
```

**Why:** Claude thinks better when you structure the thinking process. You also get visibility into its reasoning.

---

## Technique 3: Few-Shot Prompting for Consistency

Don't tell Claude what to do. **Show** Claude what to do:

```python
def create_few_shot_prompt(task, examples):
    """
    Provide 2-3 good examples of input → output.
    Claude learns the pattern and replicates it.
    """
    prompt = f"""
You are a technical interviewer evaluating candidates.

Here are examples of how to evaluate:

EXAMPLE 1:
Resume: "5 years Python, built microservices at Google..."
Job: "Senior Python Engineer, 7+ years required"
Expected Output:
{{
    "experience_years": 5,
    "matches_requirement": false,
    "gap": 2,
    "note": "Strong technical skills but below experience threshold"
}}

EXAMPLE 2:
Resume: "10 years C++, 2 years Python, shipping production AI systems..."
Job: "Senior Python Engineer, 7+ years required"
Expected Output:
{{
    "experience_years": 2,
    "equivalent_experience": 8,
    "matches_requirement": true,
    "note": "C++ experience transfers well, exceeds requirement"
}}

Now evaluate this:

Resume: "{task['resume']}"
Job: "{task['job']}"

Return JSON in the same format:
"""
    return prompt
```

**Results:** Few-shot prompting reduces hallucinations by ~40% in my testing.

---

## Technique 4: Structured Output with JSON Schema

Enforce the output format. Don't hope Claude returns valid JSON:

```python
async def call_claude_with_schema(prompt, json_schema):
    """
    Tell Claude: 'Respond ONLY with JSON matching this schema.'
    Validate the response against the schema.
    """
    
    validation_instruction = f"""
Your response MUST be valid JSON matching this schema:
{json.dumps(json_schema, indent=2)}

Return ONLY the JSON. No markdown, no explanation.
"""
    
    response = await claude.messages.create(
        model="claude-3-5-sonnet",
        max_tokens=1000,
        messages=[
            {"role": "user", "content": prompt + "\n\n" + validation_instruction}
        ]
    )
    
    # Parse and validate
    try:
        json_text = response.content[0].text.strip()
        # Remove markdown fences if present
        json_text = json_text.replace("```json", "").replace("```", "").strip()
        
        output = json.loads(json_text)
        
        # Validate against schema
        jsonschema.validate(instance=output, schema=json_schema)
        return output
    
    except json.JSONDecodeError:
        raise ValueError(f"Invalid JSON from Claude: {response.content[0].text}")
    except jsonschema.ValidationError as e:
        raise ValueError(f"Output doesn't match schema: {e}")
```

**Why:** You avoid parsing errors, silent field omissions, and data inconsistencies.

---

## Technique 5: Temperature & Top-P Tuning

Most people leave these at defaults. Don't.

```python
# For deterministic tasks (scoring, classification)
response = await claude.messages.create(
    model="claude-3-5-sonnet",
    max_tokens=1000,
    temperature=0,  # 0 = deterministic, always same output
    messages=messages
)

# For creative tasks (brainstorming, content)
response = await claude.messages.create(
    model="claude-3-5-sonnet",
    max_tokens=1000,
    temperature=0.8,  # 0.8 = creative but still coherent
    messages=messages
)

# For production scoring: use temperature=0
# For content generation: use temperature=0.7-0.8
```

**Rule of thumb:**
- **Classification/Scoring** → `temperature=0` (always consistent)
- **Content Generation** → `temperature=0.7` (creative, not random)
- **Never use** → `temperature > 1.0` (too chaotic)

---

## Technique 6: Prompt Versioning & A/B Testing

Treat prompts like code. Version them. Test them.

```python
class PromptRegistry:
    """
    Keep a versioned registry of prompts.
    Test different versions against real queries.
    """
    
    PROMPTS = {
        "candidate_scoring_v1": {
            "system": "You are an expert recruiter...",
            "created_date": "2025-07-01",
            "metrics": {"avg_score": 75, "std_dev": 12, "errors": 2}
        },
        "candidate_scoring_v2": {
            "system": "You are a technical hiring manager...",  # Different tone
            "created_date": "2025-07-10",
            "metrics": {"avg_score": 78, "std_dev": 8, "errors": 0}  # Better!
        }
    }
    
    async def ab_test(self, prompt_v1, prompt_v2, test_candidates):
        """
        Run both prompts on the same candidates.
        Compare: consistency, error rate, time to eval.
        """
        results_v1 = await self.evaluate_batch(prompt_v1, test_candidates)
        results_v2 = await self.evaluate_batch(prompt_v2, test_candidates)
        
        comparison = {
            "v1_consistency": self.measure_consistency(results_v1),
            "v2_consistency": self.measure_consistency(results_v2),
            "v1_error_rate": self.measure_errors(results_v1),
            "v2_error_rate": self.measure_errors(results_v2),
            "winner": "v2" if results_v2["error_rate"] < results_v1["error_rate"] else "v1"
        }
        
        return comparison
```

**In production:** I maintain 3-5 versions of critical prompts and regularly test them.

---

## Technique 7: Error Detection & Fallbacks

Not all LLM outputs are valid. Detect and handle gracefully:

```python
async def robust_llm_call(prompt, schema, max_retries=3):
    """
    Call LLM with fallback logic.
    """
    for attempt in range(max_retries):
        try:
            response = await claude.messages.create(
                model="claude-3-5-sonnet",
                max_tokens=1000,
                temperature=0,
                messages=[{"role": "user", "content": prompt}]
            )
            
            # Parse and validate
            output = json.loads(response.content[0].text)
            jsonschema.validate(instance=output, schema=schema)
            
            return {"success": True, "output": output}
        
        except (json.JSONDecodeError, jsonschema.ValidationError) as e:
            if attempt < max_retries - 1:
                # Retry with explicit instruction
                prompt += "\n\nIMPORTANT: You must return valid JSON."
                continue
            else:
                # Give up, return error
                return {
                    "success": False,
                    "error": str(e),
                    "fallback_action": "escalate_to_human"
                }
        
        except Exception as e:
            return {
                "success": False,
                "error": f"Unexpected error: {str(e)}",
                "fallback_action": "use_default_value"
            }
```

**Rule:** Always have a fallback. Don't let LLM errors crash your system.

---

## Technique 8: Cost Optimization Through Caching

Same question asked twice? Cache the response:

```python
async def cached_llm_call(prompt, cache_duration_hours=24):
    """
    Check cache before calling Claude.
    """
    prompt_hash = hashlib.sha256(prompt.encode()).hexdigest()
    
    # Check cache
    cached = await cache.get(f"llm:{prompt_hash}")
    if cached and not cache.is_expired(cached):
        return {"source": "cache", "output": cached["output"]}
    
    # Not cached → call Claude
    response = await claude.messages.create(
        model="claude-3-5-sonnet",
        max_tokens=1000,
        messages=[{"role": "user", "content": prompt}]
    )
    
    output = response.content[0].text
    
    # Store in cache
    await cache.set(f"llm:{prompt_hash}", {
        "output": output,
        "created_at": datetime.now(),
        "expires_at": datetime.now() + timedelta(hours=cache_duration_hours)
    })
    
    return {"source": "claude", "output": output}
```

**Impact:** Reduced LLM calls by 60% for recruitment scoring (many repeat candidates).

---

## Production Prompt Template

Here's my battle-tested template:

```
# System Prompt (stable, don't change often)
You are a [ROLE]. Your task is to [SPECIFIC OBJECTIVE].

## Context
[Background info about the domain]

## Instructions
1. [Step 1]
2. [Step 2]
3. [Step 3]

## Constraints
- [Hard constraint 1]
- [Hard constraint 2]
- [You MUST return JSON only, no markdown]

## Output Format
{
    "field1": "type1",
    "field2": "type2"
}

---

# User Prompt (variable input, consistent format)
[Specific task for this input]

Input:
[User data here]

Constraints:
[Any specific constraints for this input]

Return JSON.
```

---

## Results From Production Use

**Before:** Prompt engineering was ad-hoc, inconsistent  
**After:** Structured prompting with versioning

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| **Output Validity** | 94% | 99.8% | ↑ 5.8% |
| **Consistency (σ)** | High | Low | Better |
| **Avg Tokens/Query** | 1,200 | 420 | 65% cheaper |
| **Time to New Prompt** | 2 days | 2 hours | 24x faster |
| **A/B Test Wins** | N/A | 8/10 | Measurable |

---

## Key Takeaways

1. **Structure your prompts.** System + User pattern is more maintainable.
2. **Show, don't tell.** Few-shot examples beat verbose instructions.
3. **Make it think.** Chain-of-thought forces better reasoning.
4. **Enforce output.** JSON schema validation prevents silent failures.
5. **Temperature matters.** Deterministic for scoring, creative for content.
6. **Version & test.** Treat prompts like code with A/B testing.
7. **Handle errors gracefully.** Fallbacks are non-negotiable in production.
8. **Cache aggressively.** Same queries shouldn't hit Claude twice.

---

## What I'm Learning Next

- **Prompt distillation** — Can smaller models replicate Claude's outputs?
- **Dynamic temperature** — Adjust based on task uncertainty?
- **Instruction hierarchy** — When to embed vs. inject constraints?
- **Multi-turn reasoning** — Better results from conversation than single request?

---

**Spent weeks optimizing a prompt that still breaks?** You're not alone. Hit me up on [LinkedIn](https://linkedin.com/in/sumathi-m-5198271a0) with your edge cases.

---

*Techniques from 18+ months of production LLM work scaling Claude and Azure OpenAI deployments.*
