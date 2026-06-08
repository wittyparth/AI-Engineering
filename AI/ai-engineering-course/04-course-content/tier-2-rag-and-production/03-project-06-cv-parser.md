# Project 6: Structured Data Extraction Pipeline

- **Tier:** 2 — First Real Clients
- **Project #:** 6 of 7
- **Tech Stack:** Python, LLM API (GPT-4o-mini), Pydantic v2, batch processing queue, JSON/CSV output, cost tracking
- **Concepts:** Structured outputs at scale, Pydantic validation + retry logic, cost optimization per document, batch failure handling, quality measurement
- **Quality Gate:** ✅ APPROVED when extraction accuracy > 95% on a validation set, cost < $0.10 per CV, and you can show the failure cases where it struggles

---

## Phase 1: Brief (Priya)

> *Priya: "This one's a volume play. No RAG. No chat. Just extraction. Lots of it."*

**Client:** **TalentScout** — a recruiting firm that handles placements for 40+ companies. They process 1,000+ CVs/resumes every week.

**Their problem:** Every CV comes in a different format — PDF, DOCX, plain text. Some are 2 pages, some are 12. Every format has different sections in different orders. Their current process? A human reads each CV and manually types the data into their ATS (Applicant Tracking System). It takes 15 minutes per CV. They spend 250 hours/week on data entry.

**What we're building:** A pipeline that takes raw CVs (PDF, DOCX, text) and extracts structured data:
- Candidate name, contact info, location
- Work experience (company, title, dates, responsibilities)
- Education (degree, institution, year)
- Skills (technical + soft)
- Certifications

**Non-negotiable:**
- Must handle PDF, DOCX, and plain text
- Must validate output (no missing required fields, no hallucinated experience)
- Must be consistent — same CV twice produces the same result 95%+ of the time
- Must cost less than $0.10 per CV (that's $100/week for 1,000 CVs — cheaper than the human)
- Must detect and flag CVs it can't parse reliably (don't silently produce garbage)

**Deadline:** 6 days. They're running a hiring drive next week and need the pipeline ready.

**Definition of done:** Drop 10 CVs into a folder → run the pipeline → get a CSV file with all 10 CVs parsed into identical structure. Cost < $1.00 for 10 CVs. Accuracy > 95%.

---

## Phase 2: Learning Path (Maya)

> *Maya: "No RAG this time. This is about structured extraction — getting LLMs to produce perfect, consistent structured data at scale. It's harder than it sounds."*

### Why This Is Hard

"Structured extraction sounds simple: 'dump the CV into the LLM and ask for JSON.' But here's what goes wrong:

1. **CVs are LONG.** A 12-page CV with detailed publication history can be 20,000+ tokens. That's expensive to process.
2. **CVs are FORMAT-WILD.** Some use tables for skills. Some write experience in paragraph form. Some don't have a separate education section. The LLM needs to handle every layout.
3. **Consistency is hard.** Run the same CV twice and the LLM might structure experience differently. The second run might include a detail the first missed.
4. **Validation at scale.** You can't manually check 1,000 outputs. You need automated validation that catches bad extractions.

### Learning Order (Scratch-First)

**Step 1: Build the Pydantic schema.**
Before you write any extraction code, define exactly what "structured data" looks like. Every field, every type, every validation rule.

> *"Schema-First Development. The output model IS the spec. If the schema is wrong, the pipeline is wrong. Get this right first."*

**Step 2: Single CV extraction.**
One CV in, structured JSON out. Use Structured Outputs with Pydantic + `strict: true`.

**Step 3: Handle long CVs.**
A CV that exceeds the model's context window (or your cost budget) needs a strategy. Do you truncate? Summarize sections? Process section by section and merge?

**Step 4: Validation + retry.**
Build validators that catch bad extractions (missing required fields, implausible values, inconsistent dates). When validation fails, retry with feedback — 'you missed the education section, please include it.'

**Step 5: Batch processing + cost tracking.**
Process 10+ CVs. Track per-CV cost. Log failures. Generate a batch summary report.

**Step 6: Quality measurement.**
Build a validation set of 50 CVs. Measure extraction accuracy per field. Identify systematic weaknesses (e.g., skills extraction is 98% accurate but date ranges are only 85%).

### Memory Triage

**Memorize cold:**
- The `client.beta.chat.completions.parse()` pattern with Pydantic models
- The `additionalProperties: false` requirement in strict schemas
- The validation-with-retry pattern: `try → validate → if fails: retry with error feedback`
- The `tiktoken` encoding pattern for token counting + truncation

**Look up when needed:**
- Python-DOCX library API (`python-docx` for Word files)
- PyMuPDF specific parameters for different PDF layouts
- `pandas` to_csv/write methods for CSV output

**Understand deeply:**
- Why zero temperature matters for extraction — *"temperature != 0 means different output every time. For extraction you want DETERMINISTIC output. Temperature must be 0."*
- Why validation with retry is more effective than asking nicely — *"the first extraction might miss fields. If you validate and retry with 'you missed X,' the second attempt is much more reliable. This pattern catches 90% of extraction errors."*

### First Concrete Step

> "Build the Pydantic schema. Define CVOutput with nested models for Experience, Education, Skill, Certification. Test it: create an instance manually with fake data, serialize to JSON, verify the output shape."

> *"If the schema doesn't work with fake data, it won't work with real data. Test the schema in isolation before you add the LLM."*

---

## Phase 3: The Build

> *This is a data engineering project that happens to use AI. Most of the work is format handling, validation, and batch management.*

### Milestone 1: Pydantic Schema Design

```python
from pydantic import BaseModel, Field, field_validator
from typing import List, Optional
from datetime import date

class Experience(BaseModel):
    company: str = Field(..., description="Company name")
    title: str = Field(..., description="Job title")
    start_date: str = Field(..., description="Start date (month/year)")
    end_date: Optional[str] = Field(None, description="End date (month/year) or 'Present'")
    responsibilities: List[str] = Field(..., description="Key responsibilities/achievements")
    
    @field_validator('end_date')
    def validate_end_date(cls, v):
        if v and v.lower() not in ('present', 'current'):
            # Basic format check
            assert '/' in v, "end_date must be in month/year format or 'Present'"
        return v

class Education(BaseModel):
    degree: str = Field(..., description="Degree name (e.g., 'B.S. Computer Science')")
    institution: str = Field(..., description="School/university name")
    year: Optional[int] = Field(None, description="Graduation year")
    gpa: Optional[float] = Field(None, ge=0.0, le=4.3, description="GPA if listed")

class Skill(BaseModel):
    name: str = Field(..., description="Skill name")
    category: Optional[str] = Field(None, description="e.g., 'Technical', 'Soft', 'Language'")
    proficiency: Optional[str] = Field(None, description="e.g., 'Expert', 'Intermediate', 'Beginner'")

class CVOutput(BaseModel):
    full_name: str
    email: Optional[str] = None
    phone: Optional[str] = None
    location: Optional[str] = None
    summary: Optional[str] = Field(None, description="Professional summary/profile")
    experience: List[Experience] = []
    education: List[Education] = []
    skills: List[Skill] = []
    certifications: List[str] = []
```

**Expected stuck point:** Pydantic validation catches errors but some are subtle — like date strings that aren't parseable, or GPAs > 4.0 from non-US institutions.

**Maya's Socratic question:**
> *"Your schema rejects a GPA of 4.5. But some Indian universities use a 10-point scale, and a 4.5 GPA is reasonable there. How do you handle different grading systems without rejecting valid data?"*

> They should: make GPA validation more flexible (log a warning instead of raising an error) OR add a `gpa_scale` field. This is a real production problem — schemas need to accommodate cultural variation.

### Milestone 2: Extraction with Retry

The core extraction function with validation retry:

```python
def extract_cv(cv_text: str, max_retries: int = 2) -> CVOutput:
    """Extract structured data from CV text with validation retry."""
    
    # Handle long CVs by truncating to safe token count
    tokens = tiktoken.encoding_for_model("gpt-4o-mini").encode(cv_text)
    if len(tokens) > 12000:  # Leave room for system prompt + output
        cv_text = truncate_to_tokens(cv_text, 12000)
    
    prompt = EXTRACTION_PROMPT  # see below
    
    for attempt in range(max_retries + 1):
        try:
            response = client.beta.chat.completions.parse(
                model="gpt-4o-mini",
                messages=[
                    {"role": "system", "content": prompt},
                    {"role": "user", "content": cv_text}
                ],
                response_format=CVOutput,
                temperature=0.0,  # CRITICAL: deterministic output
            )
            
            result = response.choices[0].message.parsed
            
            # Validate the result
            validation_errors = validate_extraction(result, cv_text)
            if not validation_errors:
                return result
            
            # On last attempt, return with warnings
            if attempt == max_retries:
                result._warnings = validation_errors
                return result
                
        except Exception as e:
            if attempt == max_retries:
                raise
            continue
    
    raise ValueError("Failed to extract CV after all retries")

def validate_extraction(result: CVOutput, original_text: str) -> list[str]:
    """Check for common extraction errors."""
    errors = []
    
    # Check for empty extractions
    if not result.experience and not result.education:
        errors.append("No experience or education extracted")
    
    # Check for missing key fields
    if not result.full_name:
        errors.append("Name not extracted")
    
    # Check for unusually short extractions (maybe truncated)
    if result.experience and len(original_text) > 5000 and len(result.experience) <= 1:
        errors.append("Only 1 experience entry for a long CV — possible truncation")
    
    return errors
```

**Expected stuck point:** The `response_format=CVOutput` with `temperature=0.0` should be deterministic. But OpenAI's Structured Outputs with strict mode occasionally refuses to answer (safety refusal). Your code needs to handle `refusal=True` in the response.

**Maya's Socratic question:**
> *"The LLM refused to extract data from a CV because it detected 'sensitive information.' How do you handle refusals for a legit data extraction task?"*

> They should: check `response.choices[0].message.refusal` explicitly, implement a different system prompt for refused CVs (e.g., "This is a CV for recruitment purposes, not unauthorized data processing.").

### Milestone 3: Batch Pipeline

Build the batch processing system:

```python
import os
import json
import csv
from pathlib import Path
from datetime import datetime

class BatchCVProcessor:
    def __init__(self, input_dir: str, output_dir: str, cost_limit: float = 10.0):
        self.input_dir = Path(input_dir)
        self.output_dir = Path(output_dir)
        self.cost_limit = cost_limit
        self.results = []
        self.total_cost = 0.0
        self.failures = []
    
    def estimate_batch_cost(self) -> dict:
        """Count files, estimate tokens, project cost."""
        files = list(self.input_dir.glob("*.pdf")) + \
                list(self.input_dir.glob("*.docx")) + \
                list(self.input_dir.glob("*.txt"))
        return {"file_count": len(files), "estimated_cost": len(files) * 0.05}
    
    def process_all(self) -> dict:
        """Process all CVs in the input directory."""
        os.makedirs(self.output_dir, exist_ok=True)
        
        files = self._get_cv_files()
        print(f"Found {len(files)} CVs to process")
        
        for file_path in files:
            try:
                print(f"Processing: {file_path.name}")
                cv_text = self._read_file(file_path)
                result = extract_cv(cv_text)
                
                # Track cost
                self.total_cost += self._last_call_cost  # from your cost tracker
                
                # Save individual result
                output_path = self.output_dir / f"{file_path.stem}.json"
                with open(output_path, "w") as f:
                    json.dump(result.model_dump(), f, indent=2)
                
                self.results.append({
                    "file": file_path.name,
                    "status": "success",
                    "name": result.full_name,
                    "skills_count": len(result.skills),
                    "cost": self._last_call_cost
                })
                
                # Check budget
                if self.total_cost > self.cost_limit:
                    print(f"Budget limit ${self.cost_limit} reached. Stopping.")
                    break
                    
            except Exception as e:
                self.failures.append({
                    "file": file_path.name,
                    "error": str(e)
                })
                print(f"  FAILED: {e}")
        
        # Generate batch report
        self._generate_report()
        return self._summary()
    
    def _generate_report(self):
        """CSV summary of all results."""
        csv_path = self.output_dir / "batch_results.csv"
        with open(csv_path, "w", newline="") as f:
            writer = csv.DictWriter(f, fieldnames=[
                "file", "status", "name", "skills_count", "cost"
            ])
            writer.writeheader()
            writer.writerows(self.results)
```

**Expected stuck point:** A CV is in DOCX format and `python-docx` extracts text, but the formatting is terrible — tables come out as jumbled text, headers mix with body text.

**Maya's Socratic question:**
> *"30% of your DOCX CVs come out as garbled text because of tables. The LLM can't extract from garbage input. How do you handle CVs that your text extraction pipeline cannot read properly?"*

> They should: (1) detect garbled output (e.g., too many special chars, no sentence structure), (2) flag them for human review, (3) optionally try OCR as fallback. Not every CV is machine-parsable — knowing when to escalate is the skill.

### Milestone 4: Quality Measurement

Build the accuracy evaluation:

```python
def evaluate_extraction_accuracy(
    test_set: list[dict],
    cv_processor
) -> dict:
    """
    test_set: list of {"cv_path": str, "expected": CVOutput}
    """
    field_accuracy = {
        "full_name": {"correct": 0, "total": 0},
        "email": {"correct": 0, "total": 0},
        "experience": {"correct": 0, "total": 0, "partial": 0},
        "skills": {"correct": 0, "total": 0},
    }
    
    for test_case in test_set:
        expected = test_case["expected"]
        
        with open(test_case["cv_path"]) as f:
            cv_text = f.read()
        
        try:
            result = extract_cv(cv_text)
        except:
            field_accuracy["failures"] = field_accuracy.get("failures", 0) + 1
            continue
        
        # Exact match for name
        if result.full_name.lower() == expected.full_name.lower():
            field_accuracy["full_name"]["correct"] += 1
        field_accuracy["full_name"]["total"] += 1
        
        # Partial match for skills (allowing 80% overlap)
        expected_skills = set(s.lower() for s in expected.skills)
        actual_skills = set(s.name.lower() for s in result.skills)
        overlap = len(expected_skills & actual_skills)
        if overlap / max(len(expected_skills), 1) >= 0.8:
            field_accuracy["skills"]["correct"] += 1
        field_accuracy["skills"]["total"] += 1
    
    # Calculate overall
    return field_accuracy
```

**Expected stuck point:** Human-labeled test data takes TIME. You need 50 manually annotated CVs. Who labels them? How do you ensure the labels are correct?

**Maya's Socratic question:**
> *"You need a test set but labeling 50 CVs takes 5+ hours. What's your strategy for building a validation set without spending all your time annotating?"*

> They should: (1) use Priya to get 10 hand-labeled CVs from the client (they already have their ATS database), (2) use those 10 to measure accuracy, (3) augment with synthetic CVs generated by an LLM with known ground truth.

### Rohan's Mid-Build Interruption

> *Rohan appears while you're testing accuracy.*

"Your overall accuracy claim is **96%.** Show me the per-field breakdown. I bet:
- Name extraction is 99%+
- Skills extraction is 95%+
- Date range extraction is maybe 70-80%

Date ranges are the classic failure mode. *'June 2020 - March 2022'* versus *'June 2020 to present'* versus *'2020-2022'* versus *'6/20 - 3/22'* — your LLM might not normalize these consistently.

Prove me wrong. Run a per-field accuracy breakdown."

### Priya's Requirement Change

> *Priya: "TalentScout loved the CSV output. Now they want it to ALSO detect the best-fit job category for each candidate — frontend engineer, data scientist, product manager, etc. They use categories to route candidates to the right recruiters. 50+ categories. Two extra days."*

This forces: adding a classification field to the schema, testing accuracy across 50 categories, handling candidates who could fit multiple categories.

---

## Phase 4: Review (Rohan)

> *You submit: batch pipeline, cost logs, per-field accuracy breakdown, failure analysis, category classification.*

### Decision Documentation Required

1. Your per-field accuracy breakdown (which fields are most/least reliable)
2. Your costing model per CV — how you stay under $0.10
3. Your truncation strategy for long CVs — how you decide what to keep and what to drop
4. Your retry strategy — how often retries trigger and whether they improve accuracy

### Rohan's Review

| # | Check | Status | Notes |
|---|---|---|---|
| 1 | **Does it work?** | ✅ PASS | CV in, JSON out. Batch processes reliably. |
| 2 | **Edge cases?** | ✅ PASS | Handles PDF, DOCX, text. Handles garbled input gracefully. Truncation strategy for long CVs. |
| 3 | **Cost-aware?** | ✅ PASS | $0.045/CV with GPT-4o-mini. Well under $0.10 budget. Detailed per-CV cost tracking. |
| 4 | **Observable?** | ✅ PASS | Per-file logs. Batch summary. Cost per file. Failure details. I can audit any extraction. |
| 5 | **Right approach?** | ✅ PASS | Schema-first, zero temperature, validation with retry, truncation for long docs. Solid. |
| 6 | **Decisions justified?** | ✅ PASS | You measured per-field accuracy, you know where it fails, you documented tradeoffs. |
| 7 | **Measurable quality?** | ✅ PASS | 96.2% overall. Per-field breakdown confirms dates (82%) drag the average down — which I predicted. You identified the weakness honestly. |

### Verdict: ✅ APPROVED

*Rohan nods.*

"Your date extraction accuracy is 82% — that's below the 95% gate for the overall score, but dates drag the average down while your other fields are all 95%+. I'm approving because you IDENTIFIED the weakness honestly and documented it. The client should be told: 'we're 96% accurate overall, but dates need manual review — here's how to flag them.'

**One note for next time:** Your category classification is using the same GPT-4o-mini call as extraction. That's smart — no extra cost. But test accuracy per category — some categories might be confused (e.g., 'ML Engineer' vs 'Data Scientist'). You should flag borderline cases for human review."

---

## Phase 5: Debrief (Maya)

> *After APPROVED.*

**Maya:** "This project wasn't about AI. It was about **reliability engineering.** Let me explain:

- **Schema design IS the hard part.** The Pydantic model took you 30 minutes. But getting it right — fields, types, validators, edge cases — that's where the quality lives. This is true for every structured extraction system.
- **Retry with validation doubles reliability.** Your first-pass accuracy might be 85%. With validation + retry, it's 96%. Two LLM calls with feedback is cheaper than one call with perfect prompting.
- **Per-field accuracy matters more than overall.** Overall 96% hides date accuracy of 82%. When you present quality metrics, always break them down by field. Every senior engineer knows this.
- **Cost at scale is the real constraint.** $0.045/CV at 1,000 CVs/week = $45/week. That's a rounding error for a client saving 250 hours of human labor. But if you'd used GPT-4o, it'd be $0.40/CV = $400/week. Model selection IS business strategy.

**What you'll see again:**
- **Retry with feedback** — This pattern comes back in every extraction system. You'll use it in Project 9 (financial RAG) for citation verification.
- **Schema-first design** — Every complex output system starts here. You're building the habit.
- **Cost-per-document tracking** — Project 7 (model routing) is entirely about this.

> *Maya: "One more project in Tier 2. And it's a fun one — you get to play with model routing and LiteLLM. Basically, you get to build the system that decides WHICH model to use for every query. It's the economic brain of the operation."*
