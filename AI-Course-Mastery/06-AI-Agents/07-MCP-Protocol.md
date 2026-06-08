# 07 — MCP: Model Context Protocol — The Universal Tool Standard

## 🎯 Purpose & Goals

### 🛑 STOP. Before reading about MCP, think about the tool problem you already have.

Throughout Files 01-06, you've built tools the SAME way every time:

```python
def search_web(query: str) -> str:
    # Tool implementation
    
def calculate(expression: str) -> str:
    # Tool implementation

# Then you manually register them with your agent
agent = Agent(tools=[search_web, calculate, ...])
```

This works. But there's a HIDDEN problem that becomes painful at scale.

---

### 🤔 Scenario 1: The Tool Duplication Problem

You've built 3 agents over your projects:

```
Agent 1 (Research Bot): search_web, fetch_page, calculate
Agent 2 (Customer Support): search_knowledge_base, query_orders, send_email
Agent 3 (DevOps Bot): query_logs, restart_service, deploy_code
```

Now your company acquires a customer data API. You need ALL agents that deal with customers to use it.

**Your options:**
- A) Add the tool to each agent individually (brittle, you'll miss one)
- B) Create a shared tool registry (better, but still per-agent)
- C) Have the API EXPOSE itself as a tool that ANY agent can discover

**🤔 Before reading on:** What approach scales best? What if you have 50 agents? 200 tools? How do you manage tool versions, permissions, and updates across all of them?

---

### 🤔 Scenario 2: The Context Problem

Your agent uses `search_web`. Fine. But what about:

- **Files** — Your agent needs to read files from Google Drive, Notion, and local disk
- **Databases** — Your agent needs to query Postgres, MongoDB, and Redis
- **APIs** — Your agent needs to call Stripe, Slack, GitHub, and your internal APIs
- **Services** — Your agent needs to trigger Docker containers, deploy to Kubernetes, restart servers

**Each integration requires:**
1. Install a Python SDK
2. Write authentication code
3. Build a tool wrapper
4. Handle errors for that specific API
5. Register with each agent

**Your challenge:** You have 10 services to integrate. Each takes 2 hours to build a tool for. If you have 5 agents, each needs all 10 tools. That's 50 integration points to maintain.

**How do you reduce this to 10 integrations (one per service) that ALL agents can use?**

---

### 🤔 Scenario 3: The Non-Python Problem

Your agents are built in Python. But your company has:

- A Go service that processes payments
- A Node.js service that handles real-time notifications
- A Rust service that does performance-critical computation
- A PostgreSQL database with customer data
- A Redis cache with session data

**Your challenge:** Your agent (Python) needs to call all of these. Currently:
- You'd need Python SDKs for each (some don't exist)
- You'd need to write wrappers for non-Python services
- You'd need to handle different auth mechanisms
- You'd need to deploy tool code alongside your agent

**What if there was a STANDARD PROTOCOL that let ANY service expose its capabilities as tools — regardless of the language it's written in?**

---

> **MCP (Model Context Protocol) is that standard.**
>
> By the end of this file, you'll understand how it works, when to use it, and how it compares to building tools by hand.

**By the end of this file, you will:**
1. Understand what MCP is and the problem it solves
2. Build an MCP server that exposes tools (in ANY language)
3. Connect an MCP client to your agent (so it can use any MCP server's tools)
4. Compare MCP vs. hand-built tools — when each makes sense
5. Build a practical integration: MCP server for a database, file system, or API

---

## ⏱ Time Budget

| Section | Estimated Time |
|---------|---------------|
| Purpose & Discovery Scenarios | 15 min |
| What Is MCP? (The Problem It Solves) | 20 min |
| MCP Architecture: Server, Client, Protocol | 15 min |
| Code: Building Your First MCP Server | 40 min |
| Code: Connecting MCP to Your Agent | 30 min |
| Code: MCP Client in Python | 25 min |
| MCP vs. Hand-Built Tools: Tradeoffs | 15 min |
| Good Output Examples | 10 min |
| Antipatterns & Failure Modes | 15 min |
| Drills & Challenges | 30 min |
| Gate Check | 10 min |
| **Total** | **~4 hours** |

---

## 📖 Concept Deep-Dive

### 1. What Is MCP?

MCP = **M**odel **C**ontext **P**rotocol

Created by Anthropic in late 2024, MCP is a standard for how AI models (agents) discover and use tools.

**The analogy that makes it click:**

> MCP is to AI tools what **USB** is to computer peripherals.

```
Without USB: Every device needs a custom connector, custom driver, custom software
With USB: Plug in any device → standard protocol → it just works

Without MCP: Every tool needs custom code per agent framework
With MCP: Any MCP server → any MCP client → it just works
```

**How it works:**

```
┌─────────────────────────────┐    ┌─────────────────────────────┐
│        MCP CLIENT          │    │        MCP SERVER           │
│       (Your Agent)         │    │     (Your Tool/Service)     │
│                            │    │                             │
│ 1. Connects to server ────→│    │                             │
│ 2. Lists available tools ←──│    │  "I offer: search_db,      │
│    "What can you do?"      │    │   get_user, send_email"     │
│                            │    │                             │
│ 3. Calls a tool ──────────→│    │                             │
│    "search_db({...})"      │    │                             │
│ 4. Gets result ←───────────│    │  "Here's the data"         │
│                            │    │                             │
└─────────────────────────────┘    └─────────────────────────────┘
```

### 2. The MCP Architecture

```
CONCEPT         │ WHAT IT IS
────────────────┼────────────────────────────────────
MCP Server      │ A process that exposes tools, resources, and prompts
                │ Can be written in ANY language (Python, JS, Go, Rust)
                │ Runs as a separate process (stdio) or over HTTP (SSE)

MCP Client      │ Your agent code that connects to MCP servers
                │ Discovers available tools
                │ Calls tools and gets results

Transport       │ How client and server communicate
                │ stdio: Server runs as subprocess (for local tools)
                │ SSE: Server runs as HTTP service (for remote tools)

Capabilities    │ What a server can offer
                │ Tools: Functions the agent can call
                │ Resources: Data the agent can read
                │ Prompts: Templates the agent can use
```

### 3. MCP vs. Hand-Built Tools

**🤔 The honest comparison: when to use each?**

```
FACTOR              │ HAND-BUILT TOOLS            │ MCP
────────────────────┼─────────────────────────────┼─────────────────────────
Complexity          │ Simple: just a Python fn    │ More setup: server + transport
Language            │ Same as agent (Python)      │ ANY language
Discovery           │ Manual registration         │ Automatic (list_tools)
Versioning          │ Ship with agent code        │ Independent deployment
Auth                │ Code-based                  │ Standardized auth
Error handling      │ Custom                      │ Standard format
Debugging           │ Same process                │ Cross-process
Performance         │ Direct function call        │ IPC overhead (stdio/HTTP)
Tool count          │ 1-10                        │ 10-100+
Agent count         │ 1-3                         │ 3-50+
```

**When to hand-build:**
- 1-5 simple tools, same language, same repo
- Prototyping and learning
- Performance-critical tools

**When to use MCP:**
- Many tools from different services
- Tools written in different languages
- Tools owned by different teams
- You want to share tools across multiple agents

---

## 💻 Code Examples

### ⚙️ Setup

```bash
pip install mcp httpx
```

### Example 1: Building an MCP Server (Python)

Let's build an MCP server that exposes database query tools:

```python
"""
MCP Server: Database Tools
=========================
This server exposes database tools via the Model Context Protocol.

Any MCP-compatible client (Claude Desktop, your custom agent, etc.)
can discover and use these tools WITHOUT writing any integration code.

🤔 Key insight: This server is INDEPENDENT of any agent.
We build it ONCE. Any agent that connects to it gets these tools.
"""

import json
import sqlite3
import os
from typing import Any
from mcp.server import Server
from mcp.server.stdio import stdio_server
import mcp.types as types


# ── Database Connection ──

DB_PATH = os.getenv("MCP_DB_PATH", "data.db")


def get_db() -> sqlite3.Connection:
    """Get database connection with row factory."""
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    return conn


# ── Create the MCP Server ──

server = Server("database-server")


@server.list_tools()
async def list_tools() -> list[types.Tool]:
    """
    Tell clients what tools this server offers.
    
    This is how MCP achieves DISCOVERY — agents can ask
    "What can you do?" and get a complete list with schemas.
    
    No manual registration needed. Add a tool here, and
    every connected agent instantly sees it.
    """
    return [
        types.Tool(
            name="query_products",
            description="""Search for products in the database.
            Use this to find product information, pricing, inventory.
            
            Examples:
            - All products: {"sql": "SELECT * FROM products LIMIT 10"}
            - By category: {"sql": "SELECT * FROM products WHERE category = 'electronics'"}
            - By price: {"sql": "SELECT * FROM products WHERE price < 100 ORDER BY price"}
            """,
            inputSchema={
                "type": "object",
                "properties": {
                    "sql": {
                        "type": "string",
                        "description": "SQL SELECT query. Only SELECT allowed. Limit results with LIMIT.",
                    }
                },
                "required": ["sql"],
            },
        ),
        types.Tool(
            name="get_product_by_id",
            description="Get full details of a specific product by ID.",
            inputSchema={
                "type": "object",
                "properties": {
                    "product_id": {
                        "type": "integer",
                        "description": "The product ID to look up.",
                    }
                },
                "required": ["product_id"],
            },
        ),
        types.Tool(
            name="get_stats",
            description="Get aggregate statistics about products.",
            inputSchema={
                "type": "object",
                "properties": {
                    "category": {
                        "type": "string",
                        "description": "Optional category filter.",
                    }
                },
            },
        ),
    ]


@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    """
    Handle tool calls from clients.
    
    This is where the actual work happens. The client (your agent)
    calls a tool by name with arguments, and this function executes it.
    """
    
    if name == "query_products":
        return await handle_query_products(arguments)
    elif name == "get_product_by_id":
        return await handle_get_product(arguments)
    elif name == "get_stats":
        return await handle_get_stats(arguments)
    else:
        raise ValueError(f"Unknown tool: {name}")


async def handle_query_products(arguments: dict) -> list[types.TextContent]:
    """Execute a SELECT query with safety validation."""
    sql = arguments["sql"]
    
    # ⚠️ SAFETY: Only allow SELECT statements
    sql_upper = sql.strip().upper()
    if not sql_upper.startswith("SELECT"):
        return [types.TextContent(
            type="text",
            text="❌ Error: Only SELECT queries are allowed."
        )]
    
    # Block dangerous operations
    blocked = ["DROP ", "DELETE ", "UPDATE ", "INSERT ", "ALTER ", "CREATE "]
    for keyword in blocked:
        if keyword in sql_upper:
            return [types.TextContent(
                type="text",
                text=f"❌ Error: '{keyword}' operations are not allowed."
            )]
    
    conn = get_db()
    try:
        cursor = conn.execute(sql)
        rows = cursor.fetchmany(50)  # Limit to 50 rows
        columns = [desc[0] for desc in cursor.description]
        
        result = [dict(zip(columns, row)) for row in rows]
        return [types.TextContent(
            type="text",
            text=json.dumps(result, indent=2, default=str)
        )]
    except Exception as e:
        return [types.TextContent(
            type="text",
            text=f"❌ Query error: {str(e)}"
        )]
    finally:
        conn.close()


async def handle_get_product(arguments: dict) -> list[types.TextContent]:
    """Get a single product by ID."""
    product_id = arguments["product_id"]
    
    conn = get_db()
    try:
        cursor = conn.execute(
            "SELECT * FROM products WHERE id = ?", (product_id,)
        )
        row = cursor.fetchone()
        
        if row:
            columns = [desc[0] for desc in cursor.description]
            return [types.TextContent(
                type="text",
                text=json.dumps(dict(zip(columns, row)), indent=2, default=str)
            )]
        else:
            return [types.TextContent(
                type="text",
                text=f"Product with ID {product_id} not found."
            )]
    finally:
        conn.close()


async def handle_get_stats(arguments: dict) -> list[types.TextContent]:
    """Get aggregate statistics."""
    category = arguments.get("category")
    
    conn = get_db()
    try:
        if category:
            cursor = conn.execute(
                """SELECT COUNT(*) as count, AVG(price) as avg_price, 
                          MIN(price) as min_price, MAX(price) as max_price
                   FROM products WHERE category = ?""",
                (category,)
            )
        else:
            cursor = conn.execute(
                """SELECT COUNT(*) as count, AVG(price) as avg_price,
                          MIN(price) as min_price, MAX(price) as max_price
                   FROM products"""
            )
        
        row = cursor.fetchone()
        columns = [desc[0] for desc in cursor.description]
        return [types.TextContent(
            type="text",
            text=json.dumps(dict(zip(columns, row)), indent=2, default=str)
        )]
    finally:
        conn.close()


# ── Run the Server ──

if __name__ == "__main__":
    import asyncio
    
    print("🚀 MCP Database Server starting...")
    print(f"   Database: {DB_PATH}")
    print(f"   Transport: stdio")
    print(f"   Tools: query_products, get_product_by_id, get_stats")
    
    asyncio.run(stdio_server(server))
```

**To run:**
```bash
python mcp_db_server.py
```

**Test with any MCP client** (Claude Desktop, Cursor, or your custom agent).

---

### 🤔 Checkpoint: What Does MCP Give You?

Think about what we just built:

1. **Discovery:** The server TELLS clients what tools it has (no manual registration)
2. **Schemas:** Each tool's parameters are documented (agents know how to call them)
3. **Safety:** SQL validation is BUILT INTO the server (not the agent)
4. **Language independence:** This could be Node.js, Go, Rust — doesn't matter

**Your challenge:** You have 5 services your agents need: a database, a file system, a Slack API, a GitHub API, and an internal CRM.

**Diagram the MCP architecture:**
- How many MCP servers?
- What tools does each server expose?
- How do your agents discover and use all of them?

---

### Example 2: MCP Client — Connecting to Your Agent

Now let's build the CLIENT side — code that connects your agent to any MCP server:

```python
"""
MCP Client — your agent can now use ANY MCP server's tools.

This is the KEY insight: once you have this client, your agent
can use tools from ANY MCP server without writing integration code.
"""

import json
import subprocess
import asyncio
from typing import Any, Optional
from mcp.client.stdio import stdio_client
from mcp.client.session import ClientSession
from mcp.client.sse import sse_client
from dataclasses import dataclass


@dataclass
class MCPTool:
    """Representation of a tool discovered from an MCP server."""
    server_name: str
    name: str
    description: str
    input_schema: dict
    

class MCPClient:
    """
    Client that connects to MCP servers and exposes their tools.
    
    This is the BRIDGE between MCP servers and your agent.
    Your agent calls tools through this client, which routes
    them to the right MCP server.
    
    🤔 Think of this as a TOOL AGGREGATOR.
    Your agent has ONE interface: "call_tool(name, args)"
    This client handles finding the right server and routing.
    """
    
    def __init__(self):
        self.servers: dict[str, Any] = {}
        self.tools: dict[str, tuple[str, Any]] = {}  # tool_name -> (server_name, schema)
    
    async def connect_stdio(self, name: str, command: str, args: list[str] = None):
        """
        Connect to a local MCP server via stdio.
        
        The server runs as a subprocess. Communication happens
        over stdin/stdout — fast and simple for local tools.
        """
        server_process = await asyncio.create_subprocess_exec(
            command,
            *(args or []),
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
        )
        
        # Create an MCP session over stdio
        async with stdio_client(server_process) as (read, write):
            async with ClientSession(read, write) as session:
                await session.initialize()
                
                # Discover tools
                result = await session.list_tools()
                
                for tool in result.tools:
                    self.tools[tool.name] = (name, tool)
                    print(f"  ✅ [{name}] Discovered tool: {tool.name}")
                
                self.servers[name] = {
                    "session": session,
                    "process": server_process,
                    "tools": result.tools,
                }
    
    async def connect_sse(self, name: str, url: str):
        """
        Connect to a remote MCP server via SSE (Server-Sent Events).
        
        For tools running on other machines or services.
        """
        async with sse_client(url) as (read, write):
            async with ClientSession(read, write) as session:
                await session.initialize()
                
                result = await session.list_tools()
                
                for tool in result.tools:
                    self.tools[tool.name] = (name, tool)
                    print(f"  ✅ [{name}] Discovered tool: {tool.name}")
                
                self.servers[name] = {
                    "session": session,
                    "tools": result.tools,
                }
    
    def get_tool_schemas(self) -> list[dict]:
        """
        Get ALL tools from ALL servers as OpenAI-compatible function schemas.
        
        This is the KEY method — it converts MCP tools into the format
        your agent expects (OpenAI function calling format).
        """
        schemas = []
        for tool_name, (_server_name, tool) in self.tools.items():
            schemas.append({
                "type": "function",
                "function": {
                    "name": tool_name,
                    "description": tool.description,
                    "parameters": tool.input_schema,
                }
            })
        return schemas
    
    async def call_tool(self, tool_name: str, arguments: dict) -> str:
        """
        Call a tool on the right MCP server.
        
        Finds which server owns this tool and routes the call.
        The agent doesn't need to know or care which server handles it.
        """
        if tool_name not in self.tools:
            return f"⚠️ Unknown tool: {tool_name}"
        
        server_name, tool = self.tools[tool_name]
        server_data = self.servers.get(server_name)
        
        if not server_data:
            return f"⚠️ Server '{server_name}' is not connected"
        
        session = server_data["session"]
        
        try:
            result = await session.call_tool(tool_name, arguments)
            return result.content[0].text if result.content else "(no output)"
        except Exception as e:
            return f"⚠️ Error calling {tool_name}: {str(e)}"
    
    async def disconnect_all(self):
        """Clean up all server connections."""
        for name, server_data in self.servers.items():
            if "process" in server_data:
                server_data["process"].terminate()
        self.servers = {}
        self.tools = {}
```

---

### Example 3: Agent Using MCP Tools

Now let's connect your ReAct agent to use MCP tools:

```python
"""
ReAct agent using MCP tools instead of hardcoded tools.
"""

class MCPEnabledAgent:
    """
    A ReAct agent that gets its tools from MCP servers.
    
    Key difference from File 02 agent:
    - Tools are NOT hardcoded. They come from MCP servers.
    - Add new capabilities by connecting to new MCP servers.
    - Agent doesn't know (or care) WHERE tools come from.
    """
    
    def __init__(self, mcp_client: MCPClient, system_prompt: str = None):
        self.mcp = mcp_client
        self.system_prompt = system_prompt or """You are an AI assistant with 
access to tools from multiple services. Use them to accomplish tasks.
Think step by step. Verify information. Cite sources."""
        
        from openai import OpenAI
        self.client = OpenAI()
        self.messages = [{"role": "system", "content": self.system_prompt}]
    
    async def run(self, user_input: str) -> str:
        """Run agent with MCP-powered tools."""
        self.messages.append({"role": "user", "content": user_input})
        
        # Get tool schemas from ALL connected MCP servers
        tools = self.mcp.get_tool_schemas()
        
        if not tools:
            return "⚠️ No tools available. Connect to an MCP server first."
        
        print(f"🔧 Available tools: {[t['function']['name'] for t in tools]}")
        
        iteration = 0
        max_iterations = 10
        
        while iteration < max_iterations:
            iteration += 1
            print(f"\n[Iteration {iteration}]")
            
            response = self.client.chat.completions.create(
                model="gpt-4o-mini",
                messages=self.messages,
                tools=tools,
                tool_choice="auto",
                temperature=0.0,
            )
            
            message = response.choices[0].message
            
            if not message.tool_calls:
                return message.content
            
            for tool_call in message.tool_calls:
                tool_name = tool_call.function.name
                tool_args = json.loads(tool_call.function.arguments)
                
                print(f"  🔧 {tool_name}({json.dumps(tool_args)})")
                
                # Route through MCP client
                result = await self.mcp.call_tool(tool_name, tool_args)
                print(f"  📥 {result[:200]}...")
                
                self.messages.append({
                    "role": "assistant",
                    "content": None,
                    "tool_calls": [{
                        "id": tool_call.id,
                        "type": "function",
                        "function": {"name": tool_name, "arguments": json.dumps(tool_args)}
                    }]
                })
                self.messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result,
                })
        
        return "Max iterations reached."


# ── Usage ──

async def main():
    # Create MCP client
    mcp = MCPClient()
    
    # Connect to MCP servers
    print("🔌 Connecting to MCP servers...")
    
    # Connect to database server (runs locally)
    await mcp.connect_stdio("database", "python", ["mcp_db_server.py"])
    
    # Connect to more servers as needed:
    # await mcp.connect_stdio("filesystem", "python", ["mcp_fs_server.py"])
    # await mcp.connect_sse("remote-api", "https://api.company.com/mcp")
    
    print(f"\n✅ Connected to {len(mcp.servers)} servers with {len(mcp.tools)} tools")
    
    # Create agent with MCP tools
    agent = MCPEnabledAgent(mcp)
    
    # Run
    result = await agent.run("Show me all products under $50")
    print(f"\n\nFinal answer:\n{result}")

# asyncio.run(main())
```

---

## ✅ Good Output Examples

### What Great MCP Integration Looks Like

```
🔌 Connecting to MCP servers...
  ✅ [database] Discovered tool: query_products
  ✅ [database] Discovered tool: get_product_by_id
  ✅ [database] Discovered tool: get_stats
  ✅ [filesystem] Discovered tool: read_file
  ✅ [filesystem] Discovered tool: write_file
  ✅ [slack] Discovered tool: post_message
  ✅ [slack] Discovered tool: search_messages

✅ Connected to 3 servers with 7 tools

🔧 Available tools: ['query_products', 'get_product_by_id', 'get_stats',
                     'read_file', 'write_file', 'post_message', 'search_messages']

User: "Find cheap electronics and post results to #team channel"

[Iteration 1]
  🔧 query_products({"sql": "SELECT * FROM products WHERE category = 'electronics' AND price < 50"})
  📥 [{"id": 1, "name": "USB-C Hub", "price": 29.99}, ...]

[Iteration 2]
  🔧 post_message({"channel": "#team", "text": "Found 5 electronics under $50..."})
  📥 Message posted to #team

✅ Done in 2 iterations
```

### What POOR MCP Integration Looks Like

```
🔌 Connecting to MCP servers...
  ⚠️ [database] Connection failed: Server not found
  ⚠️ [slack] Connection failed: Auth token missing
  ⚠️ [filesystem] Connection failed: Process crashed

✅ Connected to 0 servers with 0 tools

Agent: "⚠️ No tools available."

🔴 PROBLEMS:
  ✗ No health checks before declaring success
  ✗ No helpful error messages about WHAT to fix
  ✗ No fallback tools
  ✗ Agent is completely blocked
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: MCP for Everything

```python
# ❌ WRONG — MCP server for a simple calculation tool
# An MCP server for... addition?
server = Server("math-server")
@server.list_tools()
async def list_tools():
    return [Tool(name="add", description="Add two numbers", ...)]

# ✅ RIGHT — Simple tools as direct functions
def add(a: int, b: int) -> int:
    return a + b
# MCP is for complex, shared, cross-language tools.
# Not for simple functions that live in the same codebase.
```

### Antipattern 2: No Auth on MCP Servers

```python
# ❌ WRONG — Any client can connect
server = Server("database-server")
# Anyone who finds this server can query your database!

# ✅ RIGHT — Add authentication check
@server.call_tool()
async def call_tool(name, arguments, auth_token: str = None):
    if not validate_token(auth_token):
        return [types.TextContent(type="text", text="❌ Unauthorized")]
    # ... proceed
```

### Antipattern 3: Oversized Tool Responses

```python
# ❌ WRONG — Returns everything
cursor.execute("SELECT * FROM products")  # 10,000 products!
return json.dumps(all_rows)  # Agent gets 10MB of data → context overflow

# ✅ RIGHT — Limit and paginate
cursor.execute("SELECT * FROM products LIMIT 50")
return json.dumps(limited_rows)  # Agent gets manageable data
```

---

## 🧪 Drills & Challenges

### Drill 1: Build an MCP File System Server (25 min)

Build an MCP server that exposes file system operations:

- `read_file(path)` — Read a file (constrained to one directory)
- `list_files(directory)` — List files in a directory
- `search_files(pattern)` — Find files matching a pattern
- `read_file_line(path, line_number)` — Read specific line

**Safety constraints:**
- Can only access files under `/app/data/`
- Can't read `.env`, `credentials`, or `secrets`
- Max file size: 100KB

### Drill 2: Connect 2 MCP Servers to One Agent (20 min)

Run the database MCP server AND the file system MCP server simultaneously. Connect your agent to BOTH.

**Test that the agent can:**
1. Query the database
2. Write results to a file
3. Read the file back

**Your agent should be able to do this WITHOUT knowing that the database and file system are different servers.**

### Drill 3: MCP vs. Direct Tools Comparison (20 min)

Build the SAME tool TWO ways:

```python
# Way 1: Direct Python function
def search_web(query: str) -> str:
    """Direct tool — same agent process."""
    return httpx.get(f"https://api.duckduckgo.com/?q={query}").text

# Way 2: MCP server
server = Server("web-search")
@server.tool()  # MCP tool — separate process
async def search_web(query: str) -> str:
    return httpx.get(f"https://api.duckduckgo.com/?q={query}").text
```

**Measure:**
1. Latency difference (same-process vs. cross-process)
2. Code complexity (lines of code)
3. Reusability (can another agent use it?)
4. Failure modes (how does each handle crashes?)

### Drill 4: The Tool Audit (15 min)

You have these existing tools. Which should be MCP servers, and which should stay as direct functions?

```
1. get_current_time() — returns timestamp
2. query_database(sql) — queries production Postgres
3. calculate(expression) — math evaluation
4. send_slack_message(channel, text) — posts to Slack
5. search_web(query) — DuckDuckGo search
6. deploy_service(service_name, version) — deploys to k8s
7. convert_currency(amount, from, to) — currency conversion
8. analyze_sentiment(text) — local ML model inference

For each: MCP or direct? Why?
```

---

## 🚦 Gate Check

Before moving to File 08 (Agent Evaluation), confirm you can:

- [ ] **Explain what MCP is** in 2 sentences (it's USB for AI tools)
- [ ] **Build an MCP server** that exposes 2+ tools
- [ ] **Connect an MCP client** to your agent
- [ ] **Handle MCP tool discovery** (agent asks "what tools are available?")
- [ ] **Compare MCP vs. direct tools** and choose the right approach

### The Real Test

Build a system with:
1. An MCP server for a SQLite database (products, orders, customers)
2. An MCP server for a mock Slack API (post messages, list channels)
3. A ReAct agent connected to BOTH servers

**Test:**
- "Find all products that are out of stock and alert the #inventory channel"
- "Show me the top 5 customers by order value and save the result to a file"

The agent should seamlessly use tools from BOTH servers without knowing they're separate.

### Final Questions

1. **Cross-reference File 03:** You spent a lot of effort on tool safety — schemas, validation gates, confirmation patterns. How does MCP CHANGE your safety model? Does it make it easier or harder? (Hint: the tool implementation is now in a SEPARATE process — how does that affect things?)

2. **The discovery problem:** Your agent connects to 5 MCP servers. Each has 5 tools. Total: 25 tools. The agent's tool list is now 25 items long. How does this affect the LLM's ability to choose the right tool? What's the maximum number of tools you'd give an agent before it gets confused?

3. **The network problem:** Your MCP server runs on a different machine (SSE transport). The network goes down mid-request. Your agent is waiting for a tool result. How do you handle this? What does the agent do — wait forever? Retry? Fail gracefully?

---

## 📚 Resources

**Official:**
- 📄 [MCP Official Documentation](https://modelcontextprotocol.io/) — Specification, tutorials, examples
- 📄 [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk) — Build MCP servers in Python
- 📄 [MCP Specification](https://spec.modelcontextprotocol.io/) — The protocol spec

**Examples:**
- 📄 [MCP Servers Repository](https://github.com/modelcontextprotocol/servers) — Official MCP server implementations (filesystem, GitHub, Slack, etc.)
- 📄 [Building MCP Clients](https://modelcontextprotocol.io/tutorials/building-a-client) — Client implementation guide

**Your MCP Progress Checklist:**
```
☐ MCP concept understood (USB for AI tools)
☐ First MCP server built and running
☐ MCP client connected to agent
☐ Multiple MCP servers connected simultaneously
☐ Tool discovery working (agent finds tools automatically)
☐ MCP vs. direct tools comparison done
☐ Safety/auth considered for MCP servers
```

---

> **Revisit the 3 scenarios from the beginning:**
>
> 1. **The Tool Duplication Problem** — MCP solves this by separating tool implementation from agent configuration. Build an MCP server once. Any agent that connects to it gets those tools automatically. No duplication.
>
> 2. **The Context Problem** — MCP standardizes how services expose tools. A database MCP server, a Slack MCP server, a file system MCP server — all use the same protocol. Your agent connects to all of them through a single client.
>
> 3. **The Non-Python Problem** — MCP servers can be written in ANY language. Your Go payment service can be an MCP server. Your Rust computation engine can be an MCP server. Your Python agent doesn't care — it communicates via the standard protocol.
>
> **Key realization:** MCP doesn't replace tool design (File 03). It standardizes tool DELIVERY. You still need good schemas, safety patterns, and validation. But now you build those ONCE per service, not once per agent per service.
>
> **Next:** File 08 — Agent Evaluation. How do you measure agent quality? Not just answer quality, but decision quality, efficiency, and safety.

---

> **Senior engineer closing thought:**
>
> *"MCP is relatively new (late 2024), and the ecosystem is still evolving. For now, I'd recommend MCP for tools that are: shared across multiple agents, owned by different teams, or written in different languages. For tools that are simple, Python-only, and used by one agent — just use direct functions. MCP adds value through standardization, not through complexity. Use it where standardization matters."*
