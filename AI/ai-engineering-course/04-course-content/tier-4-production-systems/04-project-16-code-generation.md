# Project 16: AI Code Generation Platform

- **Tier:** 4 -- Portfolio-Level Complex Systems
- **Project #:** 16 of 16
- **Tech Stack:** LangGraph (iterative loop), sandboxed code execution (Docker/Firecracker pattern), test framework integration, FastAPI, Langfuse eval pipeline
- **Concepts:** Code generation with iterative refinement, sandboxed execution, test-driven evaluation, spec-to-code pipeline, compilation recovery, safety and security
- **Quality Gate:** [APPROVED] when system passes 70%+ of its own generated test suites on first try on a benchmark of 50 programming tasks

---

## Phase 1: Brief (Priya)

> *Priya: "Final project. You're building something that writes code and runs it. Safety is everything -- one mistake and we're executing unverified code on production infrastructure."*

**Client:** **Internal Dev Tools team at NexaAI** -- they build tooling for the company's 200+ engineers. Their current project: an AI assistant that takes feature specs in natural language and generates working code.

**Their problem:** Engineers spend 40% of their time writing boilerplate code -- API endpoints, data models, CRUD operations, test skeletons. It's necessary work that doesn't require senior engineer judgment, but it still needs to be correct and follow company patterns.

**What we're building:** A system that:
1. Takes a natural language specification
2. Generates the implementation code
3. Generates tests for that code
4. Executes the tests in a sandboxed environment
5. If tests fail, examines the errors, iterates on the code, and retries (up to 3 iterations)
6. If tests pass, presents the final code with a summary

**Non-negotiable:**
- ALL code execution must be sandboxed -- no generated code touches the host system
- Must handle compilation/syntax errors gracefully (not just runtime test failures)
- Generated tests must meaningfully test the code (not trivial pass-through tests)
- The spec-to-code pipeline must be observable: you can trace every generation, every test run, every iteration

**Deadline:** 14 days. The internal tools team has a monthly review and wants to demo.

**Definition of done:** Paste a specification -> system generates code + tests -> runs them in sandbox -> iterates if needed -> returns passing code with explanation. System passes 70%+ of 50 benchmark tasks on first generation attempt.

---

## Phase 2: Learning Path (Maya)

> *Maya: "This is the most technically complex project in the course because it combines everything: structured outputs, tool use, evaluation, safety, and iteration. If you can build this, you can build almost anything in AI engineering."*

### What Makes This Hard

1. **Sandboxed execution.** AI-generated code is, by definition, untrusted. It might delete files, read passwords, install malware, or infinite-loop. You need an execution environment that is completely isolated -- the generated code cannot touch your host system, your network, or your data. This is the same problem Vercel Sandbox (Firecracker microVMs), OpenAI Codex (containers with network disabled), and Isola (gVisor application kernel) solve.

2. **Iterative self-correction.** The system generates code, runs tests, sees failures, and fixes its own code. This loop (generate -> test -> observe -> regenerate) is the fundamental pattern of agentic coding. The hard part is: how does the model know what went wrong? It needs to trace a test failure back to the specific line of code that is wrong and fix it.

3. **Spec-to-code fidelity.** The generated code must do what the spec says, not just pass the tests. You need to evaluate not just "does the test pass?" but "does the implementation match the specification?"

### Learning Order

**Step 1: Build the sandbox.**
Start simple: use Docker to spawn a container, write code into it, execute it, read the output, destroy the container. Do not use LangGraph yet -- just prove you can safely execute arbitrary code in isolation.

```python
import docker
client = docker.from_env()
container = client.containers.run(
    "python:3.12-slim",
    command="python /tmp/script.py",
    remove=True,
    network_disabled=True,   # NO network access
    read_only=True,          # NO filesystem writes
    mem_limit="256m",
    cpu_period=100000,
    cpu_quota=50000,         # half a CPU
    timeout=30
)
```

*Production pattern: network_disabled=True, read_only=True, memory limits, CPU limits, and a hard timeout. Research reality (2026): OpenAI Codex uses isolated containers with network disabled by default. Vercel Sandbox uses Firecracker microVMs. Isola uses gVisor application kernel.*

**Step 2: Build the code generator.**
A LangGraph node that takes a spec and generates Python code with structured output defining files, entry point, and dependencies.

**Step 3: Build the test generator.**
A node that takes the generated code AND the original spec and generates pytest tests. Having both prevents the model from writing trivial tests that pass for the wrong reasons.

**Step 4: Build the execution node.**
Write code to files, spin up sandbox, install dependencies, run pytest, capture output, destroy container.

**Step 5: Build the iteration loop.**
If tests fail, feed output + error messages back into the generator. Limit to 3 iterations.

**Step 6: Add spec compliance eval.**
Use LLM-as-judge to compare generated code against the original spec. Score compliance and flag discrepancies.

### Memory Triage

**Memorize cold:**
- Docker SDK container run pattern (with all security flags)
- Code generation output schema (files, entry point, dependencies)
- Iteration loop pattern: generate -> execute -> observe -> regenerate
- Test generator needs BOTH spec and code to avoid gaming

**Look up when needed:**
- Specific Docker SDK method signatures
- pytest exit codes and output parsing
- LangGraph self-loop patterns with step counting

**Understand deeply:**
- Why sandboxing is hard -- "Disabling network access removes 90% of attack surface. Read-only filesystem removes another 9%."
- Why spec compliance matters more than test passing -- "An LLM can trivially generate passing tests. The real question: does the code do what the user asked?"
- The iteration ceiling -- "3 iterations is the production sweet spot. Enough to fix bugs, not enough to introduce new ones."

### First Concrete Step

> Open a terminal. Verify Docker is installed. Run `docker run --rm --network-disabled python:3.12-slim python -c "print('hello sandbox')"`. Then write a Python script using the Docker SDK that generates code, executes it in a sandbox, captures output, and prints it.

### Resources (Just-in-Time)

- **Docker SDK for Python** -- container.run(), exec_run()
- **OpenAI Codex Safety Whitepaper** -- sandbox architecture reference
- **Vercel Sandbox GA announcement** -- Firecracker microVM patterns
- **Isola (open-source)** -- gVisor-based sandboxing on Kubernetes
- **LangGraph Loop Patterns** -- self-loop nodes with step counting
- **Anthropic: Building Effective Agents** -- agent design patterns

---

## Phase 3: The Build

### Milestone 1: The Sandbox

Build a SandboxExecutor class that spawns Docker containers with strict security:
- No network access (network_disabled=True)
- Read-only filesystem (read_only=True) except /tmp
- 256MB memory limit
- 30 second hard timeout
- Auto-destroy on completion (--rm)

```python
class SandboxResult(BaseModel):
    success: bool
    exit_code: int
    stdout: str
    stderr: str
    duration_ms: int

class SandboxExecutor:
    def execute(self, files, command) -> SandboxResult:
        # Create container with security restrictions
        # Copy generated files into container
        # Run command (install deps + run tests)
        # Capture stdout/stderr/exit code
        # Destroy container
        # Return result
```

**Expected stuck point:** docker.from_env() fails because Docker is not running or lacks permissions.

**Maya's Socratic question:**
> *"Docker is not running. Your entire system depends on it. How would you design for graceful infrastructure failure?"*
> They should: implement health checks, meaningful error messages, and graceful degradation.

### Milestone 2: Code Generator Node

Build the LangGraph node using structured outputs:

```python
class GeneratedFile(BaseModel):
    path: str
    content: str

class CodeGenerationOutput(BaseModel):
    files: list[GeneratedFile]
    entry_point: str
    dependencies: list[str]
    explanation: str

def generate_code(state, llm) -> dict:
    result = llm.with_structured_output(CodeGenerationOutput).invoke(prompt)
    return {"generated_code": result}
```

**Expected stuck point:** Generated code imports packages not installed in the sandbox.

**Maya's Socratic question:**
> *"Your generated code imports FastAPI but the sandbox only has python:3.12-slim. Where does the dependency list come from?"*
> They should: use the `dependencies` field from structured output to pre-install packages.

### Milestone 3: Test Generator + Execution

Build the test generator that creates pytest tests from both spec and code:

```python
class TestOutput(BaseModel):
    files: list[GeneratedFile]
    test_command: str

def generate_tests(state, llm) -> dict:
    result = llm.with_structured_output(TestOutput).invoke(prompt)
    return {"generated_tests": result}
```

Then wire execution: combine code + test files, install deps, run pytest, capture results.

### Milestone 4: The Iteration Loop

Use a LangGraph conditional self-loop with step counting. Max 3 iterations:

```python
def should_continue(state) -> Literal["fix_code", "evaluate"]:
    if state.get("test_results", {}).get("success"):
        return "evaluate"
    if state.get("iteration_count", 0) >= 3:
        return "evaluate"
    return "fix_code"
```

The fix node feeds complete error context: full traceback with line numbers, the failing code, and previous attempt history.

**Expected stuck point:** Model makes the same mistake across iterations.

**Maya's Socratic question:**
> *"Your model keeps making the same error on iteration 3. What is wrong with your error feedback?"*
> They should: realize vague feedback is the problem. Provide the COMPLETE traceback WITH the failing code.

### Rohan's Mid-Build Interruption

> *"I notice you're running the sandbox on your local Docker. What happens with malicious code like os.system('rm -rf /')? Yes, you have network_disabled and read_only. But I want the full threat model."*

This forces the learner to enumerate:
- Containment: Docker process isolation blocks 90% of attacks
- Resource exhaustion: --memory=256m prevents fork bombs
- Persistence: --rm flag ensures zero state survival
- Remaining risk: kernel exploits (mitigated by microVMs/gVisor in production)

### Priya's Requirement Change

> *"The team loves the demo but the generated code needs to match the project's existing patterns -- same router prefix, same error handling, same middleware. Code that fits into existing projects, not isolated files."*

This forces: project context analysis to extract conventions, pattern-aware generation prompts, and code that follows established project style.

---

## Phase 4: Review (Rohan)

> *You submit with 72% pass rate on a 50-task benchmark. Full Langfuse tracing, sandbox threat model, iteration logs.*

### Decision Documentation Required

Submit with your code:
1. Threat model for sandboxed execution
2. Iteration loop design rationale
3. Spec compliance methodology
4. Project context integration approach
5. Benchmark results with failure analysis

### Rohan's Review

| # | Check | Status | Notes |
|---|---|---|---|
| 1 | **Does it work?** | [PASS] | End-to-end: spec to passing code |
| 2 | **Edge cases handled?** | [REVISE] | 30s per iteration x 3 = 90s worst-case |
| 3 | **Cost-aware?** | [PASS] | Each iteration ~$0.15-0.30. Hard cap at 3. |
| 4 | **Observable?** | [PASS] | Traces for every generation, execution, iteration. |
| 5 | **Right approach?** | [PASS] | Sandbox-first is the only correct approach. |
| 6 | **Decisions justified?** | [REVISE] | Need model selection rationale with benchmark data. |
| 7 | **Measurable quality?** | [PASS] | 72% pass rate, documented failure distribution. |

### Verdict: [REVISE]

Fix these four:
1. **Model comparison** -- Run benchmark with gpt-4o-mini, gpt-4o, and Sonnet. Show cost vs quality.
2. **Loop limits** -- Inject MAX_ITERATIONS constant into generated code namespace.
3. **Spec compliance checklist** -- Judge needs Yes/No/N/A per requirement, not a single score.
4. **Failure mode analysis** -- Categorize the 28% failures: syntax vs logic vs dependency vs timeout.

---

## Phase 5: Debrief (Maya)

> *After Rohan APPROVES your resubmission.*

**Maya:** "You just built the most technically complex system in this course. Let's close the loop."

**What you should feel confident about now:**
- Building sandboxed execution environments for untrusted AI-generated code
- Designing iteration loops with hard caps and observability
- Spec-to-code pipelines with test generation, execution, and evaluation
- Threat modeling for AI systems executing arbitrary code
- Running benchmarks to measure system quality

**What you'll see again:**
- Every commercial AI code generation tool solves these same problems
- The generate -> test -> fix pattern appears in autonomous bug-fixing and AI CI/CD
- The sandbox pattern generalizes to any untrusted AI action

**The capstone thought:**
> *"You started 16 projects ago asking 'how do I call an API?' You're finishing asking 'how do I safely execute AI-generated code in production?' That's the entire journey of becoming an AI engineer -- not just building things that work, but building things that work safely, reliably, and measurably at scale."*

**One last thing from Rohan:**
> *"You've completed all 16 projects. If you showed me this portfolio in an interview, I'd hire you. You can pick up ANY production AI system now, understand why it's built the way it is, and improve it. That's what senior means. You've earned it."*

---

## Appendix: Production Reference

### Sandbox Threat Model

| Attack Vector | Mitigation | Residual Risk |
|---|---|---|
| File system access | read_only=True, --tmpfs /tmp | None |
| Network exfiltration | network_disabled=True | None |
| Resource exhaustion | mem_limit=256m, cpu_quota=50000 | Contained |
| Fork bomb | --pids-limit=100 | Container crashes only |
| Container escape | Docker seccomp profile | Low (mitigated by non-root) |
| Persistent malware | --rm flag, no volume mounts | None |

### Model Selection Benchmarks

| Model | Pass Rate | Cost/Run | Time/Run |
|---|---|---|---|
| gpt-4o-mini | 72% | $0.08 | 12s |
| gpt-4o | 78% | $0.35 | 18s |
| Claude Sonnet | 81% | $0.42 | 20s |
| DeepSeek-Coder | 68% | $0.03 | 15s |

**Decision:** Default gpt-4o-mini. Escalate to Sonnet after 3 failures. Effective: 76% at $0.11 avg cost.

### Failure Distribution (50-Task Benchmark)

| Failure Type | Frequency | Root Cause |
|---|---|---|
| Import/dependency errors | 35% | Model omits deps from structured output |
| Logic errors | 28% | Spec ambiguity, missed edge cases |
| Syntax/type errors | 18% | Typos, mismatched type annotations |
| Test quality issues | 12% | Tests pass but miss spec requirements |
| Timeout (infinite loop) | 7% | No loop bounds in generated code |

**Fixes by category:**
- Import errors: enforce min 1 dependency at schema level
- Logic errors: add requirement checklist to spec prompt
- Syntax errors: run py_compile pre-check before pytest
- Test quality: spec compliance judge verifies requirement coverage
- Timeout: inject MAX_ITERATIONS into generated code