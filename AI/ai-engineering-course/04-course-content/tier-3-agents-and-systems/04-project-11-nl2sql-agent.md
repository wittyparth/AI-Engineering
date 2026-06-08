# Project 11: Natural Language to SQL Agent

- **Tier:** 3 — Real Constraints
- **Project #:** 11 of 12
- **Tech Stack:** LangGraph (create_react_agent), PostgreSQL (or SQLite for dev), schema reflection, SQL validator, read-only query executor
- **Concepts:** Text-to-SQL, schema grounding, safe execution (read-only), query validation, ambiguity handling
- **Quality Gate:** ✅ APPROVED when SQL accuracy > 85% on benchmark, zero destructive queries possible, and ambiguous queries correctly ask for clarification

---

## Phase 1: Brief (Priya)

> *Priya: "This one's for healthcare. If the agent deletes patient data, we're not just fired — we're sued."*

**Client:** **MediQuery** — a healthcare analytics company. They have a PostgreSQL database with patient outcomes, treatment data, clinical trials, and operational metrics.

**Their problem:** Non-technical staff — doctors, hospital administrators, clinical researchers — need to query patient data constantly. They know what they WANT to know ('how many patients with diabetes were readmitted within 30 days last quarter?') but can't write SQL. Every query goes through a data analyst, and the backlog is 3+ days.

**What we're building:** A natural language to SQL agent. A doctor types: *'Show me readmission rates for diabetes patients in Q4 2025, broken down by age group'* → the agent writes SQL → executes it safely → returns results with a plain-English interpretation.

**Non-negotiable:**
- **ZERO destructive queries possible.** The agent cannot DELETE, DROP, UPDATE, INSERT, or TRUNCATE. If the generated SQL contains any writing operation, block it before execution.
- Must understand the database schema — 50+ tables, complex joins, unfamiliar column names like `dx_prmy_cd` (primary diagnosis code)
- Must handle ambiguous queries by asking for clarification, not guessing
- Must return results AND a plain-English explanation of what the query did
- SQL accuracy > 85% on a benchmark of 100 diverse queries

**Deadline:** 10 days. They have a research team starting a major analysis next month.

**Definition of done:** Doctor types query → agent generates SQL → validates it's read-only → executes → returns data + explanation. 85%+ of generated queries produce correct results.

---

## Phase 2: Learning Path (Maya)

> *Maya: "Text-to-SQL is one of the hardest LLM tasks. The schema is complex, the queries involve joins across 5+ tables, and one wrong column name means a wrong answer."*

### Why This Is Hard

"Three reasons text-to-SQL fails in production:

1. **Schema complexity.** A healthcare DB has 50+ tables with cryptic column names (`dx_prmy_cd`, `adm_ts`, `pat_stat`). The LLM doesn't know what these mean without schema descriptions.

2. **Ambiguity is everywhere.** *'Show me diabetes patients'* — which table has diagnosis info? Which column indicates diabetes? There might be 3 different diagnosis columns across 2 tables.

3. **Safety is non-negotiable.** If the agent generates `DROP TABLE patients`, it must be blocked BEFORE execution. Not after. The safety check must be in the CRITICAL PATH.

### Learning Order (Scratch-First)

**Step 1: Schema reflection + description.**
Query PostgreSQL's `information_schema` to discover tables, columns, types, and foreign keys. Enrich with descriptions — `dx_prmy_cd` → 'primary diagnosis code (ICD-10)'.

> *"The schema representation IS the prompt. If the schema is poorly described, the LLM writes bad SQL. Spend time on making the schema clear."*

**Step 2: Single-table queries.**
*'How many patients were admitted in January?'* — simple `SELECT COUNT(*) FROM ... WHERE ...` Get this working first.

**Step 3: Multi-table joins.**
*'Show readmission rates by diagnosis'* — needs JOIN across 3-4 tables. The LLM needs to figure out foreign key relationships. Your schema descriptions + FK metadata make this possible.

**Step 4: Safe execution layer.**
Build a query validator that checks SQL BEFORE it runs: parse the SQL, check for destructive keywords, check for allowed read-only operations only. This is a HARD BLOCK — no exceptions.

**Step 5: Ambiguity handling.**
When the LLM isn't sure which column/table to use, it should GENERATE MULTIPLE OPTIONS and ask: *'Did you mean diagnosis at admission (adm_diag) or primary diagnosis (dx_prmy_cd)?'*

**Step 6: Evaluation.**
100-query benchmark. Measure exact-match accuracy (does the SQL produce the correct result set?) and execution accuracy (does the SQL run without errors?).

### Memory Triage

**Memorize cold:**
- Read-only SQL check: parse → verify no DDL/DML keywords → verify SELECT only
- Schema reflection query: `SELECT column_name, data_type FROM information_schema.columns WHERE table_name = ...`
- The safe execution pattern: `BEGIN READ ONLY; SET TRANSACTION READ ONLY; [query]; COMMIT;` (database-level enforcement)

**Understand deeply:**
- Why schema grounding matters more than prompt engineering — *"no amount of clever prompting helps if the LLM doesn't know that 'adm_diag' means 'admission diagnosis.' The schema IS the knowledge base."*
- Why execution accuracy ≠ correctness — *"a SQL query can execute successfully and return WRONG results (wrong join → duplicate rows, wrong filter → missing data). Execution is necessary but not sufficient for correctness."*

### First Concrete Step

> "Connect to a PostgreSQL database. Run `SELECT * FROM information_schema.columns LIMIT 20`. See what schema data is available. Then write a function that extracts the full schema (all tables, columns, types, foreign keys) into a structured format."

---

## Phase 3: The Build

> *Safety first. Correctness second. Speed third.*

### Milestone 1: Schema Reflection

```python
class DatabaseSchema:
    def __init__(self, connection_string: str):
        self.conn = psycopg2.connect(connection_string)
        self.tables: dict[str, TableInfo] = {}
        self.reflect()
    
    def reflect(self):
        """Discover full database schema."""
        # Get all tables
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT table_name, table_type 
            FROM information_schema.tables 
            WHERE table_schema = 'public'
        """)
        
        for table_name, table_type in cursor.fetchall():
            self.tables[table_name] = self._describe_table(table_name)
    
    def _describe_table(self, table_name: str) -> TableInfo:
        """Get columns, types, descriptions for a table."""
        cursor = self.conn.cursor()
        
        # Column info
        cursor.execute("""
            SELECT column_name, data_type, is_nullable, 
                   col_description(table_name::regclass, ordinal_position)
            FROM information_schema.columns
            WHERE table_name = %s
            ORDER BY ordinal_position
        ``, (table_name,))
        
        columns = []
        for col_name, data_type, nullable, description in cursor.fetchall():
            columns.append(ColumnInfo(
                name=col_name,
                type=data_type,
                nullable=nullable == "YES",
                description=description or f"{table_name}.{col_name}"
            ))
        
        # Foreign keys
        cursor.execute("""
            SELECT kcu.column_name, ccu.table_name AS foreign_table, ccu.column_name AS foreign_column
            FROM information_schema.table_constraints tc
            JOIN information_schema.key_column_usage kcu ON tc.constraint_name = kcu.constraint_name
            JOIN information_schema.constraint_column_usage ccu ON tc.constraint_name = ccu.constraint_name
            WHERE tc.table_name = %s AND tc.constraint_type = 'FOREIGN KEY'
        ``, (table_name,))
        
        foreign_keys = cursor.fetchall()
        
        return TableInfo(
            name=table_name,
            columns=columns,
            foreign_keys=foreign_keys,
            row_count_estimate=None  # Optional: run ANALYZE for estimates
        )
    
    def to_schema_prompt(self) -> str:
        """Format schema for LLM prompt."""
        lines = []
        for table_name, table in self.tables.items():
            lines.append(f"Table: {table_name}")
            for col in table.columns:
                desc = f" // {col.description}" if col.description else ""
                nullable = " NULL" if col.nullable else " NOT NULL"
                lines.append(f"  - {col.name} ({col.type}){nullable}{desc}")
            for fk in table.foreign_keys:
                lines.append(f"  → JOIN {fk[1]} ON {table_name}.{fk[0]} = {fk[1]}.{fk[2]}")
            lines.append("")
        return "\n".join(lines)
```

### Milestone 2: SQL Generation Agent

```python
def generate_sql(question: str, schema: DatabaseSchema) -> str:
    """Generate SQL from natural language."""
    schema_prompt = schema.to_schema_prompt()
    
    prompt = f"""You are a SQL expert for a healthcare database. Given a question and the database schema, generate a READ-ONLY SQL query.

Database schema:
{schema_prompt}

Rules:
1. ONLY use SELECT queries (read-only). Never generate INSERT, UPDATE, DELETE, DROP, ALTER, TRUNCATE, or CREATE.
2. Use appropriate JOINs based on foreign key relationships.
3. Filter by date ranges where appropriate.
4. Use GROUP BY + aggregate functions (COUNT, AVG, SUM, etc.) for analytical queries.
5. Cast or format dates for readability.
6. Limit results to 100 rows unless otherwise specified.
7. If the question is ambiguous, identify the ambiguity and provide options.

Question: {question}

Return a SQL query and a brief explanation of what it does."""
    
    response = client.beta.chat.completions.parse(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        response_format=SQLQuery,  # Pydantic: query, explanation, ambiguity_notes
        temperature=0.0
    )
    
    result = response.choices[0].message.parsed
    
    if result.ambiguity_notes:
        # Return ambiguity to user instead of executing
        return AmbiguousQuery(notes=result.ambiguity_notes)
    
    return result
```

**Expected stuck point:** The LLM uses a column name that doesn't exist (`diagnosis_code`) instead of the actual column (`dx_prmy_cd`). The SQL executes but returns 0 rows or wrong results because the column doesn't exist.

**Maya's Socratic question:**
> *"Your SQL used the column name `diagnosis_code` but the real column is `dx_prmy_cd`. The LLM guessed the name instead of using the actual schema. How do you force the LLM to only use column names that actually exist?"*

> They should: (1) include schema descriptions in the prompt (tell the LLM what each column MEANS so it doesn't guess names), (2) add a post-generation validation step that parses the SQL and checks every column reference against actual schema, (3) if invalid columns found, retry with the error message: "Column 'diagnosis_code' does not exist. Did you mean 'dx_prmy_cd'?"

### Milestone 3: Safe Execution Layer

This is the most important part:

```python
import sqlparse
from typing import Optional

class SafeSQLExecutor:
    """Enforces read-only SQL execution. Blocks destructive queries."""
    
    DESTRUCTIVE_KEYWORDS = ["INSERT", "UPDATE", "DELETE", "DROP", "ALTER", 
                            "TRUNCATE", "CREATE", "REPLACE", "EXECUTE", "CALL"]
    
    def validate_and_execute(self, sql: str, conn) -> dict:
        """Validate SQL is read-only, then execute."""
        # Step 1: Parse and validate
        parsed = sqlparse.parse(sql)
        
        if not parsed:
            return {"error": "Empty or invalid SQL", "blocked": True}
        
        # Step 2: Check for destructive operations
        first_stmt = parsed[0]
        first_token = first_stmt.token_first(skip_cm=True)
        
        if first_token and first_token.value.upper() in self.DESTRUCTIVE_KEYWORDS:
            return {
                "error": f"BLOCKED: Destructive operation '{first_token.value}' is not allowed. Only SELECT queries are permitted.",
                "blocked": True,
                "sql": sql
            }
        
        # Step 3: Check for multi-statement SQL (potential injection)
        if len(parsed) > 1:
            for stmt in parsed[1:]:
                token = stmt.token_first(skip_cm=True)
                if token and token.value.upper() in self.DESTRUCTIVE_KEYWORDS:
                    return {
                        "error": "BLOCKED: Multi-statement SQL with destructive operations detected",
                        "blocked": True,
                        "sql": sql
                    }
        
        # Step 4: Execite in read-only transaction
        try:
            with conn:
                with conn.cursor() as cur:
                    cur.execute("SET TRANSACTION READ ONLY")
                    cur.execute(sql)
                    
                    # Get column names
                    columns = [desc[0] for desc in cur.description] if cur.description else []
                    rows = cur.fetchmany(100)  # Limit results
                    
                    return {
                        "columns": columns,
                        "rows": rows,
                        "row_count": len(rows),
                        "blocked": False
                    }
        except Exception as e:
            return {
                "error": f"Execution error: {str(e)}",
                "blocked": False,  # SQL was safe, execution failed
                "sql": sql
            }
```

**Expected stuck point:** A user asks 'delete all records where patient status is inactive.' The LLM generates `DELETE FROM patients WHERE status = 'inactive'`. The validator catches this and BLOCKS it.

**Maya's Socratic question:**
> *"The validator blocked a destructive query. But what if someone asks 'remove duplicate entries from the patient_log table' — a legit data cleaning request? How do you distinguish malicious from legitimate modifications?"*

> They should: the answer is HARD SEPARATION. The NL2SQL system is READ-ONLY ONLY. Write operations go through a SEPARATE system with different permissions. This is the database security principle of least privilege applied to AI.

### Milestone 4: Ambiguity Handling

```python
def handle_ambiguity(question: str, schema: DatabaseSchema) -> dict:
    """Detect ambiguous references and ask for clarification."""
    schema_prompt = schema.to_schema_prompt()
    
    response = client.beta.chat.completions.parse(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": f"""
Given this question and schema, identify any ambiguous references:
- Column names that could match multiple schema columns
- Filter values that could have different interpretations
- Join paths that could go multiple ways

Question: {question}

Schema:
{schema_prompt}

Return the ambiguities found.
"""}],
        response_format=AmbiguityAnalysis,
        temperature=0.0
    )
    
    result = response.choices[0].message.parsed
    
    if result.ambiguities:
        return {
            "needs_clarification": True,
            "ambiguities": result.ambiguities,
            "clarification_question": "I found some ambiguities in your question:\n" + "\n".join(
                f"- {a.description}\n  Options: {', '.join(a.options)}"
                for a in result.ambiguities
            )
        }
    
    return {"needs_clarification": False}
```

### Rohan's Mid-Build Interruption

> *Rohan appears during SQL accuracy testing.*

"Your overall accuracy is 82%. Below the 85% gate. Let me see the per-query breakdown:

- Simple queries (single table): 95% accuracy
- 2-3 table joins: 78% accuracy
- 4+ table joins with aggregations: 55% accuracy

Your complex queries are dragging the average down. The healthcare analysts need complex queries the most. What's your strategy for improving multi-join accuracy?"

### Priya's Requirement Change

> *Priya: "MediQuery loves the demo but wants ONE more feature: the agent should SHOW ITS WORK. For each generated query, show the SQL, explain why it chose each table and join, and highlight any assumptions it made. They need this for research reproducibility."*

This forces: add a `reasoning_trace` to each response showing the LLM's step-by-step reasoning about table selection, join paths, and filter decisions.

---

## Phase 4: Review (Rohan)

> *You submit: NL2SQL agent, schema reflection, safe executor, ambiguity handler, 100-query eval, reasoning trace.*

### Decision Documentation Required

1. Your schema enrichment strategy — how you map cryptic column names to meaningful descriptions
2. Your safe execution architecture — layers of protection, what each layer catches
3. Your accuracy by query complexity — where the system excels and where it fails
4. Your ambiguity detection — what types of ambiguity you handle and your clarification workflow

### Rohan's Review

| # | Check | Status | Notes |
|---|---|---|---|
| 1 | **Does it work?** | ✅ PASS | NL → SQL → execute → explain. End-to-end. |
| 2 | **Edge cases?** | ✅ PASS | Safe executor blocks ALL destructive operations. Ambiguity detection works. Handles null results, empty results, errors gracefully. |
| 3 | **Cost-aware?** | ✅ PASS | GPT-4o-mini is sufficient for SQL generation. Average cost $0.015/query. Higher cost for complex queries that need retry — that's acceptable. |
| 4 | **Observable?** | ✅ PASS | Reasoning trace shows every decision. I can audit why the agent chose each table and join. |
| 5 | **Right approach?** | ✅ PASS | Schema grounding → generation → validation → execution. Correct architecture. Safe executor is properly layered. |
| 6 | **Decisions justified?** | ✅ PASS | Clear reasoning on schema enrichment, safety architecture, and ambiguity handling. |
| 7 | **Measurable quality?** | 🔄 REVISE | 86% overall accuracy (above 85% gate). But your complex query accuracy is only 60% — that's below the gate for that category. Need a strategy to improve it, even if it means accepting higher latency for complex queries. |

### Verdict: 🔄 REVISE

*Rohan frowns at the complex query numbers.*

"Your simple query accuracy is fine. Your complex query accuracy (4+ joins) is 60%. That means 40% of complex queries return WRONG results. In healthcare analytics, wrong results on complex questions are worse than wrong results on simple ones — they're harder to catch.

Fix: For queries classified as 'complex' (4+ tables, aggregations, subqueries), add a verification step: generate TWO SQL queries independently, compare results, and if they disagree, flag for human review. This will increase cost and latency for complex queries but improve reliability."

---

## Phase 5: Debrief (Maya)

> *After APPROVED.*

**Maya:** "Text-to-SQL is one of the hardest LLM applications. Here's why it matters:

- **Schema grounding beats prompt engineering.** The LLM can't guess column names. It MUST know them. Your schema reflection and description pipeline is the critical infrastructure. Most text-to-SQL failures trace back to bad schema representation.
- **Safety must be architectural, not behavioral.** You didn't ASK the LLM to be safe. You BUILT a safety layer that blocks destructive operations regardless of what the LLM generates. This is the difference between 'safe by design' and 'safe by hoping.'
- **Accuracy changes with complexity.** Simple queries are easy. Complex queries are hard. Knowing WHERE your system fails is as important as knowing its overall accuracy.
- **Ambiguity is a feature, not a bug.** Asking for clarification builds trust. Guessing loses trust.

**What you'll see again:**
- **Schema grounding** — Every RAG system needs this for structured data. Project 13 (Deep Research Agent) uses similar schema descriptions for web sources.
- **Safety layers** — Every agent that executes actions needs an independent safety verification layer. Not 'ask the LLM to be safe' but 'BLOCK the LLM from doing unsafe things.'
- **Multi-pass verification for complex cases** — You added a second verification step for complex queries. This pattern (easy → fast path, hard → verification path) appears in every production system.

> *Maya: "One more project in Tier 3. Project 12 brings it all together — cost-aware model routing with caching, fallbacks, and a dashboard. You built Project 7's prototype. Now build the production version. Then Tier 4 awaits."*
