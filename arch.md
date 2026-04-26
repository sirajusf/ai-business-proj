╔══════════════════════════════════════════════════════════════════════════════════════════════╗
║                        IntelliMove — Complete System Architecture                           ║
╚══════════════════════════════════════════════════════════════════════════════════════════════╝

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                              INPUT SIDE A — JD PROCESSING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  RAW JOB DESCRIPTION                                                                        │
│  Unstructured text with 5 sections:                                                         │
│  Title | About | Responsibilities | Requirements | Nice to Have                             │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 1 — SECTION EXTRACTION                                                                │
│  What: Parse the raw JD text into the 5 structured sections.                                │
│  Why:  Remove boilerplate (salary, location) so LLM only sees relevant content.             │
│  How:  Simple text parsing — no LLM needed here.                                            │
│                                                                                             │
│  Output:                                                                                    │
│  {                                                                                          │
│    "title":           "Senior Data Engineer",                                               │
│    "about":           "We are looking for...",                                              │
│    "responsibilities":"Design and build pipelines...",                                      │
│    "requirements":    "Python (must have), Spark (must have)...",                           │
│    "nice_to_have":    "Healthcare domain, dbt..."                                           │
│  }                                                                                          │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 2 — LLM SKILL EXTRACTION + WEIGHTING          🤖 AWS Bedrock (Claude 3 Haiku)         │
│  What: LLM reads all 5 sections + injected rubric.                                          │
│        Identifies every skill, assigns a raw weight, tags section, detects role type.       │
│  Why:  Extracts structured skill importance from unstructured JD text.                      │
│        Rubric ensures consistency — same reasoning rules applied to every JD.               │
│                                                                                             │
│  Rubric injected into prompt:                                                               │
│  - Frequency: mentioned more than once = higher weight                                      │
│  - Position:  listed earlier = more important                                               │
│  - Role type: IC → technical > soft | Managerial → leadership > technical                  │
│  - Explicitness: "must have" language = higher weight                                       │
│  - Constraints: no skill > 0.40 | soft cap: IC=0.25, Mgr=0.35 | sum = 1.0                 │
│                                                                                             │
│  LLM Output:                                                                                │
│  {                                                                                          │
│    "role_type": "individual_contributor",                                                   │
│    "skills": [                                                                              │
│      {"skill": "Python",           "weight": 0.28, "section": "required"     },            │
│      {"skill": "Apache Spark",     "weight": 0.22, "section": "required"     },            │
│      {"skill": "SQL",              "weight": 0.20, "section": "required"     },            │
│      {"skill": "Data Pipelines",   "weight": 0.15, "section": "required"     },            │
│      {"skill": "Leadership",       "weight": 0.10, "section": "required"     },            │
│      {"skill": "Healthcare Domain","weight": 0.03, "section": "nice_to_have" },            │
│      {"skill": "dbt",              "weight": 0.02, "section": "nice_to_have" }             │
│    ]                                                                                        │
│  }                                                                                          │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 3 — SECTION MULTIPLIER                                                                │
│  What: Apply weight multiplier based on which section the skill came from.                  │
│  Why:  Required skills must outweigh nice-to-have skills consistently.                      │
│        Enforced in code — not by LLM — so it is always consistent and auditable.           │
│                                                                                             │
│  Rules:                                                                                     │
│  required     × 1.0  →  weight unchanged                                                   │
│  nice_to_have × 0.5  →  weight halved                                                      │
│                                                                                             │
│  Output:                                                                                    │
│  {"skill": "Python",            "weight": 0.280, "section": "required"     }               │
│  {"skill": "Apache Spark",      "weight": 0.220, "section": "required"     }               │
│  {"skill": "SQL",               "weight": 0.200, "section": "required"     }               │
│  {"skill": "Data Pipelines",    "weight": 0.150, "section": "required"     }               │
│  {"skill": "Leadership",        "weight": 0.100, "section": "required"     }               │
│  {"skill": "Healthcare Domain", "weight": 0.015, "section": "nice_to_have" } ← halved     │
│  {"skill": "dbt",               "weight": 0.010, "section": "nice_to_have" } ← halved     │
│  NOTE: weights no longer sum to 1.0 — renormalization happens after dedup                  │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 4 — SKILL NORMALIZATION (JD SIDE)      🤖 LLM fallback only — dict first             │
│  What: Map every raw skill name to a canonical name + assign to a skill group.              │
│  Why:  "Apache Spark" and "Spark" must resolve to the same skill before matching.           │
│        Skill groups enable partial credit matching (PyTorch ≈ TensorFlow).                  │
│                                                                                             │
│  Flow per skill:                                                                            │
│  raw name → check SKILLS_DICT → found: return canonical (fast ⚡)                          │
│                               → not found: call LLM 🤖 → cache result → return canonical   │
│                                                                                             │
│  LLM Output (per skill, only on cache miss):                                                │
│  {"canonical_skill_name": "Apache Spark", "group": "Data Processing"}                      │
│                                                                                             │
│  Output after normalization:                                                                │
│  {"skill": "Python",           "weight": 0.280, "section": "required",                     │
│   "group": "Programming Languages"                                    }                     │
│  {"skill": "Apache Spark",     "weight": 0.220, "section": "required",                     │
│   "group": "Data Processing"                                          }                     │
│  {"skill": "SQL",              "weight": 0.200, "section": "required",                     │
│   "group": "Programming Languages"                                    }                     │
│  {"skill": "Data Pipelines",   "weight": 0.150, "section": "required",                     │
│   "group": "Data Engineering"                                         }                     │
│  {"skill": "Leadership",       "weight": 0.100, "section": "required",                     │
│   "group": "Soft Skills"                                              }                     │
│  {"skill": "Healthcare Domain","weight": 0.015, "section": "nice_to_have",                 │
│   "group": "Healthcare Standards"                                     }                     │
│  {"skill": "dbt",              "weight": 0.010, "section": "nice_to_have",                 │
│   "group": "Data Engineering"                                         }                     │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 5 — DEDUPLICATION                                                                     │
│  What: After normalization, multiple raw skills may resolve to the same canonical name.     │
│        Merge them into one entry.                                                           │
│  Why:  "Python programming" and "Python scripting" both → "Python" after normalization.     │
│        Without dedup, Python would appear twice with split weights.                         │
│                                                                                             │
│  Rules:                                                                                     │
│  merged_weight  = min(sum of duplicate weights, 0.40)  ← cap at 0.40                      │
│  merged_section = "required" if ANY duplicate was required                                  │
│                                                                                             │
│  Output: (same structure, duplicates resolved into single entries)                          │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 6 — RE-NORMALIZATION                                                                  │
│  What: Divide each weight by the new total so all weights sum back to 1.0.                  │
│  Why:  Multiplier + dedup both change the total. Scoring formula requires sum = 1.0.        │
│                                                                                             │
│  Final JD Output:                                                                           │
│  {                                                                                          │
│    "jd_id":     "JD001",                                                                    │
│    "title":     "Senior Data Engineer",                                                     │
│    "role_type": "individual_contributor",                                                   │
│    "soft_cap":  0.25,                                                                       │
│    "skills": [                                                                              │
│      {"skill": "Python",           "weight": 0.2871, "section": "required"    },            │
│      {"skill": "Apache Spark",     "weight": 0.2261, "section": "required"    },            │
│      {"skill": "SQL",              "weight": 0.2051, "section": "required"    },            │
│      {"skill": "Data Pipelines",   "weight": 0.1534, "section": "required"    },            │
│      {"skill": "Leadership",       "weight": 0.1022, "section": "required"    },            │
│      {"skill": "Healthcare Domain","weight": 0.0154, "section": "nice_to_have"},            │
│      {"skill": "dbt",              "weight": 0.0107, "section": "nice_to_have"}             │
│    ]                                                                                        │
│  }                                                                                          │
│  Weights sum: 1.0000 ✅                                                                     │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                          ┌────────────────────┘
                          │  JD side complete
                          │  feeds into MATCHING ENGINE ──────────────────────────────────────┐
                          │                                                                    │


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                         INPUT SIDE B — EMPLOYEE PROFILE ENGINE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  EMPLOYEE DATA SOURCES                                                                      │
│  Resume | Bio | Manager Reviews | Peer Reviews | Certifications | Project Records           │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 7 — LLM SKILL EXTRACTION (per document)   🤖 AWS Bedrock (Claude 3 Haiku)            │
│  What: For each document, LLM extracts every skill with type, recency, and sentiment.       │
│  Why:  Multi-source extraction builds a richer, more reliable profile than resume alone.    │
│        Sentiment captures negative signals that resumes never contain.                      │
│                                                                                             │
│  Run once per document — results merged in Step 9.                                          │
│                                                                                             │
│  LLM Output (per document):                                                                 │
│  {                                                                                          │
│    "skills": [                                                                              │
│      {                                                                                      │
│        "skill":            "Python",                                                        │
│        "type":             "technical",                                                     │
│        "last_used":        2024,                                                            │
│        "sentiment":        "positive",                                                      │
│        "sentiment_reason": "excellent Python skills in production pipelines"                │
│      },                                                                                     │
│      {                                                                                      │
│        "skill":            "Leadership",                                                    │
│        "type":             "soft",                                                          │
│        "last_used":        2023,                                                            │
│        "sentiment":        "negative",                                                      │
│        "sentiment_reason": "still developing, needs to improve mentoring"                  │
│      }                                                                                      │
│    ]                                                                                        │
│  }                                                                                          │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 8 — SKILL NORMALIZATION (EMPLOYEE SIDE)  🤖 LLM fallback only — dict first           │
│  What: Same normalizer as Step 4. Maps raw extracted skill names to canonical names.        │
│  Why:  Employee resume may say "Python programming" while JD says "Python".                 │
│        Without normalization they would never match.                                        │
│        Same SKILLS_DICT and SKILL_GROUPS used — shared between both sides.                  │
│                                                                                             │
│  Flow per skill: (identical to Step 4)                                                      │
│  raw name → SKILLS_DICT → found: return canonical ⚡                                        │
│                         → not found: LLM 🤖 → cache → return canonical                     │
│                                                                                             │
│  LLM Output (per skill, only on cache miss):                                                │
│  {"canonical_skill_name": "Python", "group": "Programming Languages"}                      │
│                                                                                             │
│  Output: same structure with canonical skill names replacing raw names                      │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 9 — PROFILE MERGE                                                                     │
│  What: Combine extracted skills from all documents into one unified profile per employee.   │
│  Why:  Python may appear in the resume AND a manager review.                                │
│        We need one consolidated entry with all sources and sentiments combined.             │
│                                                                                             │
│  Merge rules per skill:                                                                     │
│  New skill     → add fresh entry                                                            │
│  Existing skill → update last_used if more recent                                           │
│                → add source to sources list (if not already there)                          │
│                → append sentiment to sentiments list                                        │
│                → update sentiment_reason if new mention is negative                         │
│                                                                                             │
│  Output (merged, pre-confidence):                                                           │
│  "Python": {                                                                                │
│    "type":             "technical",                                                         │
│    "last_used":        2024,                                                                │
│    "sources":          ["resume", "manager_review"],                                        │
│    "sentiments":       ["neutral", "positive"],                                             │
│    "sentiment_reason": "excellent Python skills in production pipelines"                    │
│  }                                                                                          │
│  "Leadership": {                                                                            │
│    "type":             "soft",                                                              │
│    "last_used":        2023,                                                                │
│    "sources":          ["manager_review"],                                                  │
│    "sentiments":       ["negative"],                                                        │
│    "sentiment_reason": "still developing, needs to improve mentoring"                      │
│  }                                                                                          │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 10 — CONFIDENCE SCORING                                                               │
│  What: Compute a final confidence score per skill based on source count + sentiment.        │
│  Why:  A skill confirmed by 3 sources is more reliable than one self-reported in a resume.  │
│        A negatively reviewed skill should carry less weight than a praised one.             │
│                                                                                             │
│  Formula:                                                                                   │
│  base       = 0.70 (1 source) | 0.85 (2 sources) | 1.00 (3+ sources)                      │
│  modifiers  = +0.15 per positive | 0.00 per neutral | -0.10 per negative                   │
│  confidence = max(0.10, min(1.0, base + sum(modifiers)))                                    │
│                                                                                             │
│  Examples:                                                                                  │
│  Python (resume neutral + manager positive):                                                │
│    base=0.85  +0.00 +0.15  → confidence = 1.00                                             │
│  Leadership (manager negative):                                                             │
│    base=0.70  -0.10        → confidence = 0.60                                             │
│                                                                                             │
│  Final Employee Profile Output:                                                             │
│  {                                                                                          │
│    "emp_id": "E001",                                                                        │
│    "name":   "Alex Rivera",                                                                 │
│    "skills": {                                                                              │
│      "Python": {                                                                            │
│        "type":             "technical",                                                     │
│        "last_used":        2024,                                                            │
│        "sources":          ["resume", "manager_review"],                                    │
│        "sentiments":       ["neutral", "positive"],                                         │
│        "sentiment_reason": "excellent Python skills in production pipelines",               │
│        "confidence":       1.00                                                             │
│      },                                                                                     │
│      "Apache Spark": {                                                                      │
│        "type":             "technical",                                                     │
│        "last_used":        2022,                                                            │
│        "sources":          ["resume"],                                                      │
│        "sentiments":       ["neutral"],                                                     │
│        "sentiment_reason": "not specified",                                                 │
│        "confidence":       0.70                                                             │
│      },                                                                                     │
│      "Leadership": {                                                                        │
│        "type":             "soft",                                                          │
│        "last_used":        2023,                                                            │
│        "sources":          ["manager_review"],                                              │
│        "sentiments":       ["negative"],                                                    │
│        "sentiment_reason": "still developing, needs to improve mentoring",                  │
│        "confidence":       0.60                                                             │
│      }                                                                                      │
│    }                                                                                        │
│  }                                                                                          │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                          ┌────────────────────┘
                          │  Employee side complete
                          │  feeds into MATCHING ENGINE ──────────────────────────────────────┐
                          │                                                                    │


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                    MATCHING ENGINE  (JD side + Employee side converge here)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

     JD Skills + Weights                    Employee Skills + Confidence
     (from Step 6)                          (from Step 10)
            │                                        │
            └──────────────────┬─────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 11 — DECAY SCORE                                                                      │
│  What: Reduce each employee skill score based on how long ago it was last used.             │
│  Why:  A skill used 8 years ago is less reliable than one used last month.                  │
│        Linear decay is simple and explainable — every year costs 10 points.                 │
│                                                                                             │
│  Formula:                                                                                   │
│  decay_score = max(0.20,  1.0 − (years_since_used × 0.10))                                 │
│                                                                                             │
│  Examples:                                                                                  │
│  Python     last_used=2024  2yr ago  → max(0.20, 1.0−0.2) = 0.80                          │
│  Apache Spark last_used=2022  4yr ago  → max(0.20, 1.0−0.4) = 0.60                        │
│  Leadership  last_used=2023  3yr ago  → max(0.20, 1.0−0.3) = 0.70                         │
│                                                                                             │
│  Output: decay_score added to each skill in employee profile                                │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 12 — EFFECTIVE SKILL SCORE                                                            │
│  What: Combine decay and confidence into one number representing the employee's             │
│        true strength for that skill right now.                                              │
│  Why:  A skill can be old AND poorly reviewed — both factors should reduce the score.       │
│                                                                                             │
│  Formula:                                                                                   │
│  effective_score = decay_score × confidence                                                 │
│                                                                                             │
│  Examples:                                                                                  │
│  Python:      0.80 × 1.00 = 0.800   (current + multi-source positive)                      │
│  Apache Spark: 0.60 × 0.70 = 0.420  (4yr old + single source)                             │
│  Leadership:  0.70 × 0.60 = 0.420   (3yr old + negative review)                            │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 13 — PER-SKILL CONTRIBUTION                                                           │
│  What: For each JD skill, compute how much it contributes to the final score.              │
│  Why:  A skill that is both important (high JD weight) AND mastered (high effective)        │
│        should contribute more than an important skill that is weak or missing.              │
│                                                                                             │
│  Formula:                                                                                   │
│  if skill in employee profile:                                                              │
│      contribution = JD_weight × effective_score                                             │
│  if skill NOT in employee profile:                                                          │
│      check SKILL_GROUPS — does employee have a skill in the same group?                     │
│          YES → partial credit: contribution = JD_weight × effective_score × 0.60           │
│                status = "partial_group_match"                                               │
│          NO  → contribution = 0.00  (full gap)                                             │
│                status = "missing"                                                           │
│                                                                                             │
│  Output:                                                                                    │
│  {"skill": "Python",           "jd_weight":0.2871, "effective":0.800,                      │
│   "contribution":0.2297, "status":"matched"                          }                     │
│  {"skill": "Apache Spark",     "jd_weight":0.2261, "effective":0.420,                      │
│   "contribution":0.0950, "status":"matched"                          }                     │
│  {"skill": "Leadership",       "jd_weight":0.1022, "effective":0.420,                      │
│   "contribution":0.0429, "status":"matched"                          }                     │
│  {"skill": "Data Pipelines",   "jd_weight":0.1534, "effective":0.000,                      │
│   "contribution":0.0000, "status":"missing"                          }                     │
│  {"skill": "TensorFlow",       "jd_weight":0.1782, "effective":0.800,                      │
│   "contribution":0.0855, "status":"partial_group_match" ← PyTorch found}                   │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 14 — BASE SCORE                                                                       │
│  What: Sum all per-skill contributions into one base match score.                           │
│  Why:  Aggregates individual skill scores into an overall match percentage.                 │
│        Naturally bounded 0→1 because JD weights sum to 1.0 and effective ≤ 1.0.            │
│                                                                                             │
│  Formula:                                                                                   │
│  base_score = Σ contribution(skill)  for all skills in JD                                  │
│                                                                                             │
│  Example: 0.2297 + 0.0950 + 0.0429 + 0.0000 + ... = 0.513                                 │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 15 — SOFT SKILL BONUS                                                                 │
│  What: Add a small bonus for each soft skill the employee has in their profile.             │
│  Why:  Universal soft skills add value to any role regardless of JD requirements.           │
│        Rewards well-rounded employees without distorting the technical match score.         │
│                                                                                             │
│  Formula:                                                                                   │
│  for each skill in employee profile where type == "soft":                                   │
│      soft_bonus += 0.02                                                                     │
│  soft_bonus = min(soft_bonus, 0.10)  ← hard cap at 10%                                     │
│                                                                                             │
│  Example:                                                                                   │
│  Team Player + Communication + Fast Learner = 0.06  (under cap)                            │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 16 — FINAL SCORE                                                                      │
│  What: Add base score and soft bonus, cap at 1.0.                                           │
│  Why:  Produces the single match percentage shown to HR and employee.                       │
│                                                                                             │
│  Formula:                                                                                   │
│  final_score = min(base_score + soft_bonus, 1.0)                                           │
│                                                                                             │
│  Example: min(0.513 + 0.060, 1.0) = 0.573 → 57%                                           │
└──────────────────────────────────────────────┬──────────────────────────────────────────────┘
                                               │
                                               ▼

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                                         OUTPUT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  STEP 17 — EXPLAINABILITY REPORT                                                            │
│  What: Generate a full human-readable match report from the scoring output.                 │
│  Why:  Every number must be traceable — HR needs to justify decisions,                      │
│        employees need to understand their gaps and act on them.                             │
│  How:  Pure Python — no LLM needed. All information comes from Steps 11-16.                │
│                                                                                             │
│  Output:                                                                                    │
│  ─────────────────────────────────────────────────────────────────────                      │
│  MATCH REPORT:  Alex Rivera  →  Senior Data Engineer                                        │
│  ─────────────────────────────────────────────────────────────────────                      │
│  ✅ Python         current, 2 src, positive        23.0% / 28.7%                           │
│  ⚠️  Apache Spark   4yr old, 1 src                  9.5% / 22.6%                           │
│  ⚠️  Leadership     negative review                  4.3% / 10.2%                           │
│  ❌  Data Pipelines missing                           0.0% / 15.3%                          │
│  ─────────────────────────────────────────────────────────────────────                      │
│  Base Score:                                         51.3%                                  │
│  Soft Skills: Team Player, Communication, Fast Learner  +6.0%                              │
│  ─────────────────────────────────────────────────────────────────────                      │
│  FINAL SCORE:                                         57%                                   │
│  ─────────────────────────────────────────────────────────────────────                      │
│  TOP GAPS TO CLOSE:                                                                         │
│  1. Data Pipelines  →  acquire skill      potential gain: +15.3%                           │
│  2. Spark recency   →  refresh skill      potential gain: +13.1%                           │
│  3. Leadership      →  address review     potential gain: +5.9%                            │
│  ─────────────────────────────────────────────────────────────────────                      │
└──────────────────────────────┬──────────────────────────────────────────────────────────────┘
                               │
            ┌──────────────────┼──────────────────┐
            │                  │                  │
            ▼                  ▼                  ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────────────────┐
│  HR DASHBOARD    │ │ EMPLOYEE PORTAL  │ │  NOTIFICATIONS               │
│                  │ │                  │ │                              │
│ Ranked candidate │ │ Role browser     │ │ Score changed >5% → alert   │
│ list per role    │ │ with scores      │ │ Shortlisted → alert          │
│                  │ │                  │ │ New role posted → alert      │
│ Match report     │ │ My match report  │ │                              │
│ per candidate    │ │                  │ │                              │
│                  │ │ Career growth    │ │                              │
│ Side-by-side     │ │ plan + gaps      │ │                              │
│ comparison       │ │                  │ │                              │
│                  │ │ Aspiration mode  │ │                              │
│ Post new role    │ │ (closed roles)   │ │                              │
└──────────────────┘ └──────────────────┘ └──────────────────────────────┘


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                              SHARED UTILITIES (used throughout)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────┐  ┌─────────────────────────────┐  ┌──────────────────────────┐
│  SKILLS_DICT (JSON)         │  │  SKILL_GROUPS (JSON)        │  │  AWS Bedrock             │
│                             │  │                             │  │  Claude 3 Haiku          │
│  Maps raw → canonical       │  │  Maps canonical → group     │  │                          │
│  "spark" → "Apache Spark"   │  │  "TensorFlow" → "ML         │  │  Used in:                │
│  "TF"    → "TensorFlow"     │  │               Frameworks"   │  │  Step 2  (JD weights)    │
│                             │  │  "PyTorch"   → "ML          │  │  Step 4  (JD normalize)  │
│  Self-growing:              │  │               Frameworks"   │  │  Step 7  (emp extract)   │
│  dict first ⚡              │  │                             │  │  Step 8  (emp normalize) │
│  LLM fallback 🤖            │  │  Enables partial credit     │  │                          │
│  cache back                 │  │  matching in Step 13        │  │  max_tokens per call:    │
│                             │  │                             │  │  normalize   → 100       │
│  Shared by Steps 4 + 8      │  │  Shared by Steps 4 + 8 + 13 │  │  JD extract  → 1000      │
└─────────────────────────────┘  └─────────────────────────────┘  │  emp extract → 1500      │
                                                                   └──────────────────────────┘
