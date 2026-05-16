# Drill: Build a Custom Tool

**Time**: 30 min  **Difficulty**: Easy

## Task

Build 3 production-quality tools that an agent can use:

### Tool 1: Database Query Tool
```python
class DatabaseQueryTool(Tool):
    name = "query_database"
    description = "Execute SQL queries on the PostgreSQL database. Read-only. Returns results as list of dicts."
    parameters = {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "SQL query (SELECT only)"
            }
        },
        "required": ["query"]
    }

    async def run(self, query: str) -> dict:
        # Validate: only SELECT
        if not query.strip().upper().startswith("SELECT"):
            return {"error": "Only SELECT queries are allowed"}
        # Execute
        async with pool.acquire() as conn:
            rows = await conn.fetch(query)
            return {"results": [dict(r) for r in rows], "count": len(rows)}
```

### Tool 2: Calculator Tool
```python
class CalculatorTool(Tool):
    name = "calculate"
    description = "Perform mathematical calculations. Use for arithmetic, statistics, or any numerical computation."
    parameters = {
        "type": "object",
        "properties": {
            "expression": {
                "type": "string",
                "description": "Python arithmetic expression to evaluate"
            }
        }
    }

    async def run(self, expression: str) -> str:
        # SAFE evaluation
        allowed_names = {"abs", "max", "min", "round", "sum", "len", "range", "int", "float", "str"}
        try:
            code = compile(expression, "<string>", "eval")
            for name in code.co_names:
                if name not in allowed_names and not name.startswith("_"):
                    return {"error": f"Function '{name}' not allowed"}
            result = eval(expression, {"__builtins__": {}}, {n: __builtins__.__dict__[n] for n in allowed_names})
            return {"result": result}
        except Exception as e:
            return {"error": str(e)}
```

### Tool 3: File Reader Tool
```python
class FileReaderTool(Tool):
    name = "read_file"
    description = "Read the contents of a file from the allowed directory. Use when user asks about file contents."
    parameters = {
        "type": "object",
        "properties": {
            "path": {"type": "string", "description": "Path relative to allowed directory"}
        }
    }

    def __init__(self, allowed_dir: str = "./data"):
        self.allowed_dir = Path(allowed_dir).resolve()

    async def run(self, path: str) -> str:
        full_path = (self.allowed_dir / path).resolve()
        # Security: prevent path traversal
        if not str(full_path).startswith(str(self.allowed_dir)):
            return {"error": "Access denied: path outside allowed directory"}
        if not full_path.exists():
            return {"error": f"File not found: {path}"}
        return {"content": full_path.read_text()}
```

## Integration Test

Wire all 3 tools into your ReAct agent from the previous drill and test:
```python
agent = ReActAgent(
    tools={
        "query_database": DatabaseQueryTool(),
        "calculate": CalculatorTool(),
        "read_file": FileReaderTool(),
    },
    llm=llm_client,
)

result = await agent.run("Read the sales data file and calculate the total revenue")
print(result)
```
