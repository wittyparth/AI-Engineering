# MCP (Model Context Protocol)

## What Is MCP?

An open protocol (by Anthropic) that standardizes how AI agents connect to tools and data sources. Think "USB-C for AI agents" — one protocol for all tool integrations.

## Core Concepts

```
Agent ←→ MCP Client ←→ MCP Server ←→ External System (API, DB, FS)
```

### MCP Server
```python
from mcp import Server, Tool

class DatabaseMCPServer(Server):
    def __init__(self):
        super().__init__("database-server")

    @Tool()
    async def query_database(self, sql: str) -> list[dict]:
        """Execute a read-only SQL query. Returns results as list of dicts."""
        async with self.db_pool.acquire() as conn:
            rows = await conn.fetch(sql)
            return [dict(row) for row in rows]

    @Tool()
    async def get_table_schema(self, table_name: str) -> str:
        """Get the schema of a database table."""
        # ...
```

### MCP Client
```python
from mcp import Client

class AgentMCPClient(Client):
    def __init__(self, server_url: str):
        super().__init__(server_url)
        self.tools = []

    async def connect(self):
        # Discover available tools from server
        self.tools = await self.list_tools()

    async def call_tool(self, name: str, arguments: dict) -> Any:
        return await self.execute_tool(name, arguments)
```

## MCP + Agent

```python
class MCPAgent:
    def __init__(self):
        # Connect to multiple MCP servers
        self.clients = {
            "database": AgentMCPClient("http://db-mcp:8000"),
            "filesystem": AgentMCPClient("http://fs-mcp:8001"),
            "github": AgentMCPClient("http://github-mcp:8002"),
        }

    async def start(self):
        for client in self.clients.values():
            await client.connect()

    async def run(self, task: str) -> str:
        # All tools from all servers are available to the agent
        all_tools = []
        for name, client in self.clients.items():
            for tool in client.tools:
                all_tools.append({**tool, "server": name})

        # Agent chooses which tool from which server
        return await self._react_loop(task, all_tools)
```

## Building Your Own MCP Server

```python
from mcp.server import Server

class DocumentMCPServer(Server):
    """MCP server that provides access to the document store."""

    @Tool(description="Search documents by semantic similarity")
    async def search_documents(
        self, query: str, k: int = 5
    ) -> list[dict]:
        """Search the document store. Returns relevant document chunks."""
        results = await vector_db.search(query, k=k)
        return [
            {"title": r.metadata["title"], "content": r.content, "score": r.score}
            for r in results
        ]

    @Tool(description="Get document by ID")
    async def get_document(self, doc_id: str) -> dict:
        """Retrieve a specific document by its ID."""
        doc = await document_store.get(doc_id)
        return {"id": doc.id, "title": doc.title, "content": doc.content}

    @Resource(uri="documents://recent/5")
    async def recent_documents(self) -> list[dict]:
        """Resource showing the 5 most recently added documents."""
        return await document_store.recent(5)
```

## 🔴 Senior: When MCP vs Custom Tools

| Factor | MCP | Custom Tools |
|--------|-----|-------------|
| Standardization | Open protocol | Proprietary |
| Tool discovery | Auto-discovery | Hardcoded |
| Multi-server | Yes | Manual aggregation |
| Complexity | Medium | Low |
| Ecosystem | Growing | Custom |
| Control | Shared protocol | Full control |

Use MCP when you want interoperable tools or when different teams own different backends. Use custom tools for simple, single-backend agents.
