# Tool Design Engineering

## What Makes a Good Tool

A tool is the agent's interface to the world. Bad tools = bad agent behavior.

## Tool Schema

```python
class Tool(BaseModel):
    name: str  # Unique, descriptive
    description: str  # What it does (agent reads this to decide when to use it)
    parameters: dict  # JSON Schema for parameters
    returns: str  # What the tool returns

# Example: the description is what matters most
tools = [
    {
        "name": "search_documentation",
        "description": "Search internal technical documentation. Use when the user asks about APIs, SDKs, or implementation details.",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Search query, use technical terms"}
            }
        }
    },
    {
        "name": "get_weather",
        "description": "Get current weather for a location. Use when the user asks about weather, climate, or temperature.",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "City name"}
            }
        }
    }
]
```

## 🔴 Senior: Tool Design Principles

### 1. Descriptive Names
```
Bad:  tool_a, func1, process
Good: search_documentation, send_email, calculate_total
```

### 2. Clear Descriptions
The agent decides which tool to use based on the DESCRIPTION. This is the most important field.

```
Bad:  "Search for things"
Good: "Search internal knowledge base for company policies. Use when employee asks about HR, benefits, or procedures."
```

### 3. Fail Fast Documentation
Tell the agent what happens on failure:
```
"Returns empty list if no results found. Returns error if database is unavailable."
```

### 4. Parameter Constraints
```python
"temperature": {
    "type": "number",
    "description": "Temperature in Celsius",
    "minimum": -273.15,
    "maximum": 10000,
}
```

### 5. Input Validation
```python
class SearchTool:
    async def run(self, query: str) -> list[dict]:
        if not query or len(query) < 2:
            return {"error": "Query must be at least 2 characters"}
        if len(query) > 500:
            return {"error": "Query too long, max 500 characters"}
        # Actual search...
```

## Drill: Build 3 Production Tools

1. **Database query tool**: Execute read-only SQL, return results with schema info
2. **GitHub API tool**: Search repos, read files, create issues
3. **Slack notification tool**: Send messages to channels, read recent messages

Each must have:
- Clear description
- Input validation
- Error handling
- Return type documentation
