# 🧪 Drill — Build a Production CLI Chat with Streaming

## 🎯 Purpose & Goals

> **🛑 STOP. Your mission:**
>
> Build a terminal-based ChatGPT client from scratch. Type a question, see the response stream in **real time**, token by token — just like the web UI.
>
> **Constraints:**
> - No framework like LangChain — raw API calls only
> - Must handle: streaming display, multi-line input, errors, disconnects
> - Must support switching between OpenAI and any OpenAI-compatible provider (Ollama, Groq, etc.)
> - Must be a single Python file you can `python chat.py` and go
>
> *Before scrolling down — sketch the architecture. What classes? What async pattern? How do you handle partial stream output?*

---

**By the end of this drill, you will:**
- Build a streaming CLI chat that feels like ChatGPT in your terminal
- Master `asyncio` patterns for real-time I/O
- Handle every failure mode: network errors, API errors, partial responses, Ctrl+C
- Have a portable tool you'll actually use daily

**⏱ Time Budget:** 2 hours (1 hour build, 30 min debug challenges, 30 min extensions)

---

## 📖 The Architecture

```python
"""
SYSTEM ARCHITECTURE:

┌─────────────────────────────────────────────────────────┐
│                     chat.py                              │
│                                                          │
│  ┌──────────────────┐     ┌──────────────────────────┐  │
│  │    ChatSession    │     │    StreamingDisplay      │  │
│  │  - conversation  │────→│  - print_streaming()    │  │
│  │  - system_prompt │     │  - handle_backspace()    │  │
│  │  - stream()      │     │  - show_usage()          │  │
│  └────────┬─────────┘     └──────────────────────────┘  │
│           │                                              │
│           ▼                                              │
│  ┌──────────────────┐     ┌──────────────────────────┐  │
│  │   LLMClient      │     │     MessageHistory       │  │
│  │  - chat()        │     │  - messages[]           │  │
│  │  - stream_chat() │     │  - to_openai_format()   │  │
│  │  - count_tokens()│     │  - truncate_context()   │  │
│  └──────────────────┘     └──────────────────────────┘  │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │           InputHandler                           │   │
│  │  - read_multiline()    - handle_commands()       │   │
│  │  - read_key()          - show_help()            │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘

DATA FLOW:
1. User types multi-line input (or /command)
2. InputHandler parses — /commands handled directly, messages go to ChatSession
3. ChatSession appends user message to MessageHistory
4. ChatSession calls LLMClient.stream_chat()
5. Each token → StreamingDisplay (prints inline, no newline)
6. On complete → token usage shown, MessageHistory updated with assistant response
7. Loop back to step 1
"""
```

---

## 💻 Build Phase — Your Implementation

### Step 1: The LLM Client

Build a client that can stream from any OpenAI-compatible API:

```python
# llm_client.py — paste this, then extend it
import json
import httpx
from typing import AsyncGenerator, Optional
from dataclasses import dataclass, field


@dataclass
class LLMConfig:
    """Configuration for an LLM provider. Change ONE value to switch providers."""
    api_key: str = ""
    base_url: str = "https://api.openai.com/v1"
    model: str = "gpt-4o-mini"
    max_tokens: int = 2048
    temperature: float = 0.7


class LLMClient:
    """
    Streaming LLM client for OpenAI-compatible APIs.
    Works with: OpenAI, Ollama, Groq, Together AI, Perplexity, AnyScale, etc.
    """
    
    def __init__(self, config: LLMConfig):
        self.config = config
        self._client = httpx.AsyncClient(
            base_url=config.base_url,
            timeout=httpx.Timeout(60.0, connect=10.0),
        )
    
    async def stream_chat(
        self,
        messages: list[dict],
    ) -> AsyncGenerator[str, None]:
        """
        Stream a chat completion token by token.
        
        Yields individual content tokens as they arrive.
        Handles the SSE (Server-Sent Events) protocol that OpenAI-compatible
        APIs use for streaming.
        """
        headers = {
            "Content-Type": "application/json",
            "Authorization": f"Bearer {self.config.api_key}",
        }
        
        body = {
            "model": self.config.model,
            "messages": messages,
            "max_tokens": self.config.max_tokens,
            "temperature": self.config.temperature,
            "stream": True,  # <-- THIS is what makes it stream
        }
        
        # 🔴 SENIOR ENGINEERING MOVE: The `stream=True` changes the response
        # from a single JSON to a stream of SSE events. Each event is:
        #   data: {"choices":[{"delta":{"content":"Hello"},"index":0}]}
        # 
        # The last event is:
        #   data: [DONE]
        #
        # We need to parse this line-by-line, extract content from the delta.
        
        async with self._client.stream(
            "POST",
            "/chat/completions",
            headers=headers,
            json=body,
        ) as response:
            response.raise_for_status()
            
            async for line in response.aiter_lines():
                if not line.startswith("data: "):
                    continue
                
                data_str = line[6:].strip()  # Remove "data: " prefix
                
                if data_str == "[DONE]":
                    return
                
                try:
                    chunk = json.loads(data_str)
                except json.JSONDecodeError:
                    continue  # Malformed chunk — skip, don't crash
                
                # Extract content from the delta
                delta = chunk.get("choices", [{}])[0].get("delta", {})
                content = delta.get("content", "")
                
                if content:
                    yield content
    
    async def close(self):
        await self._client.aclose()
```

> **🤔 YOUR TURN:** Add a non-streaming `chat()` method that returns the full response at once. Then add error handling — what happens if the API key is wrong? Network drops mid-stream?

---

### Step 2: The Streaming Display

Building streaming output in a terminal is harder than it looks. You need to:
- Print tokens as they arrive (no newline)
- Handle backspace (if using a "thinking..." spinner then replacing it)
- Support colored output (user input blue, assistant green, errors red)

```python
# display.py
import sys
from datetime import datetime
from typing import Optional


class Colors:
    """Terminal ANSI color codes. Usage: print(f"{Colors.GREEN}Hello{Colors.RESET}")"""
    RESET = "\033[0m"
    BOLD = "\033[1m"
    DIM = "\033[2m"
    
    # Text colors
    BLUE = "\033[94m"
    GREEN = "\033[92m"
    YELLOW = "\033[93m"
    RED = "\033[91m"
    CYAN = "\033[96m"
    
    # Background for tokens
    USER_BG = "\033[104m"  # Blue background
    ASSISTANT_BG = "\033[102m"  # Green background
    
    @classmethod
    def user(cls, text: str) -> str:
        return f"{cls.BLUE}{text}{cls.RESET}"
    
    @classmethod
    def assistant(cls, text: str) -> str:
        return f"{cls.GREEN}{text}{cls.RESET}"
    
    @classmethod
    def error(cls, text: str) -> str:
        return f"{cls.RED}{text}{cls.RESET}"
    
    @classmethod
    def dim(cls, text: str) -> str:
        return f"{cls.DIM}{text}{cls.RESET}"
    
    @classmethod
    def bold(cls, text: str) -> str:
        return f"{cls.BOLD}{text}{cls.RESET}"


class StreamingDisplay:
    """
    Manages real-time streaming output in the terminal.
    
    Prints tokens inline as they arrive, then finishes with formatting.
    Handles the tricky parts:
    - No newlines between tokens
    - Flush after every token (terminal buffering is real)
    - Proper cleanup on completion
    """
    
    def __init__(self):
        self.token_count = 0
        self.start_time: Optional[datetime] = None
        self.first_token_time: Optional[datetime] = None
    
    def display_header(self):
        """Show the welcome banner."""
        print(f"\n{Colors.bold('🧠 AI CLI Chat')} — {Colors.dim('Streaming • type /help for commands')}")
        print(f"{Colors.dim('─' * 60)}\n")
    
    async def stream_print(self, token: str):
        """
        Print a single token as it arrives.
        
        🔴 SENIOR MOVE: Always flush! Python buffers stdout by default.
        Without `flush=True`, tokens batch up and you lose streaming.
        """
        if self.token_count == 0:
            self.first_token_time = datetime.now()
            # Print assistant prefix on first token
            print(f"{Colors.assistant('→')} ", end="", flush=True)
        
        print(token, end="", flush=True)
        self.token_count += 1
    
    def finish(self):
        """Called when streaming completes. Shows summary."""
        elapsed = (datetime.now() - self.first_token_time).total_seconds() if self.first_token_time else 0
        print()  # Newline after stream
        print(f"{Colors.dim(f'  ─ {self.token_count} tokens in {elapsed:.1f}s │ {self.token_count/elapsed:.0f} t/s')}")
        print()
    
    def show_error(self, message: str):
        """Display an error message."""
        print(f"\n{Colors.error(f'✖ {message}')}\n")
```

> **🤔 YOUR TURN:** Add a "thinking..." animation before the first token arrives (time-to-first-token can be 1-5 seconds). When the first token comes, erase the "thinking..." and show the real response. *Hint: Use `\r` (carriage return) to overwrite the line.*

---

### Step 3: The Input Handler

Multiline input and slash commands make this feel like a real tool:

```python
# input_handler.py
import shutil


class InputHandler:
    """
    Handles user input with multi-line support and slash commands.
    
    Supports:
    - Multi-line input (type naturally, press Enter twice to submit)
    - Slash commands: /clear, /model, /help, /system, /exit
    - Line editing (backspace, arrow keys — via readline)
    """
    
    COMMANDS = {
        "/help": "Show available commands",
        "/clear": "Clear conversation history",
        "/model": "Switch model (e.g., /model gpt-4o-mini)",
        "/system": "Change system prompt (/system You are a helpful pirate)",
        "/tokens": "Show current conversation token count",
        "/save": "Save conversation to file",
        "/exit": "Exit the chat",
    }
    
    def read_multiline(self) -> str:
        """
        Read multi-line input. Submit with double Enter (blank line).
        
        Why multi-line? Most AI chat UIs support it. Your users will
        paste code, write prompts with line breaks, etc.
        """
        print(f"{Colors.user('You')} (blank line to submit, Ctrl+C to cancel)")
        print(f"{Colors.dim('│')} ", end="")
        
        lines = []
        try:
            while True:
                line = input()
                if line == "" and len(lines) > 0:
                    # Blank line after content → submit
                    break
                lines.append(line)
        except KeyboardInterrupt:
            print(f"\n{Colors.dim('(cancelled)')}")
            return ""
        
        return "\n".join(lines)
    
    def handle_command(self, text: str) -> bool:
        """
        Check if text is a command. Returns True if handled.
        
        Commands start with /. Returns True if it was a command
        (so the caller doesn't send it to the LLM).
        """
        if not text.startswith("/"):
            return False
        
        parts = text.split(maxsplit=1)
        cmd = parts[0].lower()
        args = parts[1] if len(parts) > 1 else ""
        
        if cmd == "/help":
            self._show_help()
        elif cmd == "/exit":
            print(f"\n{Colors.dim('Goodbye!')}")
            sys.exit(0)
        elif cmd == "/model":
            print(f"{Colors.dim(f'Model would switch to: {args}')}" if args else
                  f"{Colors.error('Usage: /model <model-name>')}")
        else:
            # Other commands handled by the session
            return "handled_by_session"
        
        return True
    
    def _show_help(self):
        """Display available commands."""
        terminal_width = shutil.get_terminal_size().columns
        print(f"\n{Colors.bold('Commands')}")
        print(f"{Colors.dim('─' * terminal_width)}")
        for cmd, desc in self.COMMANDS.items():
            print(f"  {Colors.cyan(cmd):<20} {Colors.dim(desc)}")
        print(f"{Colors.dim('─' * terminal_width)}\n")
    
    @staticmethod
    def get_terminal_width() -> int:
        """Get terminal width for formatting."""
        return shutil.get_terminal_size().columns
```

---

### Step 4: The Chat Session (Putting It Together)

```python
# chat_session.py
import json
from datetime import datetime
from typing import Optional


@dataclass
class Message:
    role: str  # "system" | "user" | "assistant"
    content: str
    timestamp: datetime = field(default_factory=datetime.now)
    tokens: Optional[int] = None


class MessageHistory:
    """
    Manages conversation history with token tracking and context window management.
    
    🔴 SENIOR MOVE: Most beginners send the ENTIRE conversation every time.
    Seniors TRIM history to fit context windows, balancing memory vs cost.
    """
    
    def __init__(self, system_prompt: str = "You are a helpful assistant.", max_context_tokens: int = 8000):
        self.system_prompt = system_prompt
        self.max_context_tokens = max_context_tokens
        self.messages: list[Message] = []
    
    def add_message(self, role: str, content: str, tokens: Optional[int] = None):
        self.messages.append(Message(role=role, content=content, tokens=tokens))
    
    def to_api_format(self) -> list[dict]:
        """Convert to OpenAI-compatible message list."""
        api_messages = [{"role": "system", "content": self.system_prompt}]
        
        # 🔴 SENIOR MOVE: Smart truncation — keep system prompt + recent context
        # A naive approach sends ALL messages. A senior approach:
        # 1. Always includes system prompt
        # 2. Keeps the most recent N messages that fit in context
        # 3. Drops oldest user/assistant pairs first
        total_tokens = self._estimate_tokens(self.system_prompt)
        
        for msg in reversed(self.messages):
            msg_tokens = msg.tokens or self._estimate_tokens(msg.content)
            if total_tokens + msg_tokens > self.max_context_tokens:
                break  # Drop older messages
            api_messages.insert(1, {"role": msg.role, "content": msg.content})
            total_tokens += msg_tokens
        
        return api_messages
    
    @staticmethod
    def _estimate_tokens(text: str) -> int:
        """Rough token estimate. 4 chars ≈ 1 token for English."""
        return len(text) // 4 + 1
    
    def clear(self):
        self.messages.clear()
    
    @property
    def estimated_total_tokens(self) -> int:
        return self._estimate_tokens(self.system_prompt) + sum(
            m.tokens or self._estimate_tokens(m.content) for m in self.messages
        )


class ChatSession:
    """
    Full chat session orchestrating LLM client, message history, and display.
    
    Architecture:
    ┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
    │ ChatSession  │────→│ LLMClient    │────→│ OpenAI/Ollama    │
    │              │     │              │     │                  │
    │ ChatSession  │────→│ MessageHist  │     │                  │
    │              │     │              │     │                  │
    │ ChatSession  │────→│ StreamDisplay│     │                  │
    └──────────────┘     └──────────────┘     └──────────────────┘
    """
    
    def __init__(self, llm_client: LLMClient, system_prompt: str = None):
        self.llm = llm_client
        self.history = MessageHistory(
            system_prompt=system_prompt or "You are a helpful AI assistant."
        )
        self.display = StreamingDisplay()
        self.input_handler = InputHandler()
    
    async def run(self):
        """Main chat loop."""
        self.display.display_header()
        
        while True:
            # Read user input
            user_input = self.input_handler.read_multiline()
            
            if not user_input:
                continue  # Input was cancelled
            
            # Handle commands
            if user_input.startswith("/"):
                if user_input == "/clear":
                    self.history.clear()
                    self.display.show_info("Conversation cleared")
                elif user_input.startswith("/system "):
                    new_prompt = user_input[8:]
                    self.history.system_prompt = new_prompt
                    self.display.show_info(f"System prompt updated")
                elif user_input == "/exit":
                    print(f"\n{Colors.dim('Goodbye!')}")
                    break
                else:
                    self.display.show_info(f"Unknown command: {user_input.split()[0]}. Type /help")
                continue
            
            # Display user message
            print(f"\n{Colors.user('You:')}")
            print(f"{Colors.dim(user_input)}")
            
            # Send to LLM and stream response
            self.history.add_message("user", user_input)
            api_messages = self.history.to_api_format()
            
            try:
                await self._stream_response(api_messages)
            except Exception as e:
                self.display.show_error(f"Failed: {str(e)}")
                # Remove the user message if the response failed
                self.history.messages.pop()
    
    async def _stream_response(self, messages: list[dict]):
        """Stream a response from the LLM and display it."""
        full_response = ""
        
        async for token in self.llm.stream_chat(messages):
            await self.display.stream_print(token)
            full_response += token
        
        self.display.finish()
        self.history.add_message("assistant", full_response)
    
    async def close(self):
        await self.llm.close()
```

> **🤔 YOUR TURN:** Add a `/save` command that saves the conversation as Markdown. Add a `/stats` command showing total token usage and cost for the session.

---

### Step 5: The Main Entry Point

```python
# chat.py — THE MAIN FILE
"""
AI CLI Chat — A production-quality streaming chat client.

Usage:
    python chat.py                          # Default (OpenAI GPT-4o-mini)
    python chat.py --model gpt-4o           # Switch model
    python chat.py --local                  # Use local Ollama
    python chat.py --system "Be a pirate"   # Custom system prompt

Environment variables:
    OPENAI_API_KEY  — Required for OpenAI models
    OLLAMA_BASE_URL — Optional, default: http://localhost:11434/v1
"""

import os
import sys
import argparse
import asyncio
from dotenv import load_dotenv

load_dotenv()


def parse_args():
    parser = argparse.ArgumentParser(
        description="🧠 AI CLI Chat — Streaming terminal client",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Examples:
  python chat.py                              # OpenAI GPT-4o-mini
  python chat.py --model gpt-4o               # OpenAI GPT-4o
  python chat.py --local                      # Local Ollama
  python chat.py --provider groq              # Groq (set GROQ_API_KEY)
  python chat.py --system "You are a poet"    # Custom system prompt
        """,
    )
    parser.add_argument("--model", default="gpt-4o-mini", help="Model name")
    parser.add_argument("--local", action="store_true", help="Use local Ollama")
    parser.add_argument("--provider", default="openai", choices=["openai", "groq", "together", "ollama"],
                        help="API provider")
    parser.add_argument("--system", default="You are a helpful AI assistant.", help="System prompt")
    parser.add_argument("--api-key", help="API key (defaults to env var)")
    parser.add_argument("--base-url", help="Base URL for API")
    parser.add_argument("--temperature", type=float, default=0.7, help="Model temperature")
    parser.add_argument("--max-context", type=int, default=8000, help="Max context tokens")
    return parser.parse_args()


def create_config(args) -> LLMConfig:
    """Create LLMConfig from CLI args and environment."""
    provider_configs = {
        "openai": {
            "base_url": "https://api.openai.com/v1",
            "api_key_env": "OPENAI_API_KEY",
            "default_model": "gpt-4o-mini",
        },
        "groq": {
            "base_url": "https://api.groq.com/openai/v1",
            "api_key_env": "GROQ_API_KEY",
            "default_model": "llama-3.1-70b-versatile",
        },
        "together": {
            "base_url": "https://api.together.xyz/v1",
            "api_key_env": "TOGETHER_API_KEY",
            "default_model": "meta-llama/Meta-Llama-3.1-70B-Instruct-Turbo",
        },
        "ollama": {
            "base_url": os.getenv("OLLAMA_BASE_URL", "http://localhost:11434/v1"),
            "api_key_env": None,  # No auth needed for local
            "default_model": "llama3.2",
        },
    }
    
    provider = provider_configs[args.provider if not args.local else "ollama"]
    api_key = args.api_key or (os.getenv(provider["api_key_env"]) if provider["api_key_env"] else "")
    
    # 🔴 SENIOR MOVE: Validate API key early, not after user types a message
    if provider["api_key_env"] and not api_key:
        print(f"{Colors.error(f'✖ {provider["api_key_env"]} not set.')}")
        print(f"  Set it: set {provider['api_key_env']}=your_api_key")
        print(f"  Or use --local for Ollama")
        sys.exit(1)
    
    return LLMConfig(
        api_key=api_key,
        base_url=args.base_url or provider["base_url"],
        model=args.model or provider["default_model"],
        temperature=args.temperature,
    )


async def main():
    args = parse_args()
    config = create_config(args)
    
    print(f"\n{Colors.dim(f'Model: {config.model} │ {config.base_url}')}")
    
    client = LLMClient(config)
    session = ChatSession(client, system_prompt=args.system)
    
    try:
        await session.run()
    except KeyboardInterrupt:
        print(f"\n{Colors.dim('\nGoodbye!')}")
    finally:
        await session.close()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## ✅ Good Output Example

Here's what a successful chat session looks like:

```
🧠 AI CLI Chat — Streaming • type /help for commands
────────────────────────────────────────────────────────────

Model: gpt-4o-mini │ https://api.openai.com/v1

You (blank line to submit, Ctrl+C to cancel)
│ Explain the CAP theorem in 3 bullet points.
│

→ **CAP Theorem** states that a distributed data store can only provide
  two of three guarantees simultaneously:

  1. **Consistency** — Every read receives the most recent write or an error
  2. **Availability** — Every request receives a (non-error) response, without 
     guarantee it contains the most recent write
  3. **Partition Tolerance** — The system continues to operate despite network
     partitions (messages being dropped/delayed between nodes)

  ─ 89 tokens in 1.8s │ 49 t/s

You (blank line to submit, Ctrl+C to cancel)
│ Follow up: Which one do databases typically sacrifice?
│

→ Most traditional databases (e.g., relational DBs) sacrifice Availability
  during partitions — they choose Consistency + Partition Tolerance (CP).
  
  Modern NoSQL systems (e.g., Cassandra, DynamoDB) often sacrifice 
  Consistency for Availability + Partition Tolerance (AP).

  ─ 42 tokens in 0.9s │ 47 t/s
```

---

## ❌ Antipatterns & Failure Modes

### ❌ Antipattern 1: Blocking on Stream
```python
# BAD ❌  — Reads ALL tokens, then prints all at once
response = client.chat.completions.create(messages=..., stream=True)
full_text = ""
for chunk in response:
    full_text += chunk.choices[0].delta.content or ""
print(full_text)  # Loses the entire point of streaming!
```

### ❌ Antipattern 2: No Flush
```python
# BAD ❌  — Tokens batch in buffer, user sees nothing for 5 seconds
for token in stream:
    print(token, end="")  # No flush! Python waits for newline
```

### ❌ Antipattern 3: Ignoring Partial Tokens
```python
# BAD ❌  — Crashes if chunk has no content (first/last chunks often empty)
for chunk in stream:
    content = chunk["choices"][0]["delta"]["content"]  # KeyError!
```

### ❌ Antipattern 4: Growing History Forever
```python
# BAD ❌  — Eventually blows past context window, costs $$$
messages.append(user_msg)
messages.append(assistant_msg)
# 1000 turns later... messages = 2000 items, half are useless
```

### ❌ Antipattern 5: No Error Handling
```python
# BAD ❌  — Network drops mid-stream? User stares at empty terminal forever
async for token in client.stream_chat(messages):
    print(token, end="", flush=True)
# No timeout! No network error handling! No retry!
```

---

## 🧪 Debug Challenge — Find the 5 Bugs

Here's a broken version of the streaming chat. It compiles but has 5 bugs:

```python
import asyncio
import httpx
import json

class BrokenClient:
    def __init__(self, api_key):
        self.api_key = api_key
        self.client = httpx.AsyncClient()
    
    async def stream(self, messages):
        response = await self.client.post(
            "https://api.openai.com/v1/chat/completions",
            headers={"Authorization": f"Bearer {self.api_key}"},
            json={
                "model": "gpt-4o-mini",
                "messages": messages,
                "stream": True,
            },
        )
        
        for line in response.iter_lines():
            chunk = json.loads(line)
            content = chunk["choices"][0]["delta"]["content"]
            print(content, end="")
        
        return response.choices[0].message.content

async def main():
    client = BrokenClient("sk-...")
    await client.stream([
        {"role": "user", "content": "Say hello"},
    ])

asyncio.run(main())
```

<details>
<summary>👀 Find the bugs first, then check</summary>

1. **Bug: No `.stream()` on the request** — `client.post(...)` sends a normal request. Need `client.stream("POST", ...)` for streaming mode.

2. **Bug: No `aiter_lines()`** — `response.iter_lines()` is sync. `async for` needs `response.aiter_lines()`.

3. **Bug: No SSE parsing** — OpenAI streams use `data: {...}` format, not raw JSON per line. Need to strip `data: ` prefix and skip heartbeats.

4. **Bug: KeyError on empty chunks** — First chunk has `choices[0].delta` but no `content` key. Need `.get("content", "")`.

5. **Bug: `response.choices[0].message.content` on a stream** — Streaming responses don't have `.choices[0].message.content`. The full content must be accumulated from chunks.

Here's the fix:
```python
async with self.client.stream(
    "POST", "/chat/completions",
    headers=headers,
    json=body,
) as response:
    async for line in response.aiter_lines():
        if not line.startswith("data: "):
            continue
        data_str = line[6:].strip()
        if data_str == "[DONE]":
            break
        chunk = json.loads(data_str)
        content = chunk.get("choices", [{}])[0].get("delta", {}).get("content", "")
        if content:
            yield content
```
</details>

---

## 🧪 Extension Challenges

After the basic version works, try these extensions in order:

### Challenge 1: Token Counter
Add accurate token counting using `tiktoken`:
```python
import tiktoken

def count_tokens(text: str, model: str = "gpt-4o-mini") -> int:
    enc = tiktoken.encoding_for_model(model)
    return len(enc.encode(text))
```

Show running token count per message and total for session.

### Challenge 2: Cost Tracking
Track costs per session. With current pricing (~$0.15/M input tokens, ~$0.60/M output tokens for GPT-4o-mini):
```python
class CostTracker:
    RATES = {
        "gpt-4o-mini": {"input": 0.15, "output": 0.60},  # per 1M tokens
        "gpt-4o": {"input": 2.50, "output": 10.00},
    }
    
    def add_usage(self, model: str, input_tokens: int, output_tokens: int):
        rates = self.RATES.get(model, self.RATES["gpt-4o-mini"])
        cost = (input_tokens * rates["input"] + output_tokens * rates["output"]) / 1_000_000
        self.total_cost += cost
```

### Challenge 3: Conversation Persistence
Add `/save` and `/load` commands that save/restore conversation as JSON:
```json
{
    "model": "gpt-4o-mini",
    "system_prompt": "You are a helpful assistant.",
    "messages": [
        {"role": "user", "content": "Hello", "timestamp": "2025-05-16T10:00:00"},
        {"role": "assistant", "content": "Hi there!", "timestamp": "2025-05-16T10:00:03"}
    ],
    "total_cost": 0.00042
}
```

### Challenge 4: Markdown Rendering
Add basic terminal markdown rendering:
- `**bold**` → ANSI bold
- `*italic*` → ANSI italic (or dim)
- `` `code` `` → colored background
- ``` ```code blocks``` ``` → box with dim background

### Challenge 5: Retry with Backoff
Add automatic retry for transient failures:
```python
async def stream_chat_with_retry(self, messages, max_retries=3):
    for attempt in range(max_retries):
        try:
            async for token in self.stream_chat(messages):
                yield token
            return  # Success
        except (httpx.TimeoutException, httpx.NetworkError) as e:
            if attempt == max_retries - 1:
                raise
            wait = 2 ** attempt  # Exponential backoff: 1s, 2s, 4s
            print(f"{Colors.dim(f'Retrying in {wait}s... ({attempt+1}/{max_retries})')}")
            await asyncio.sleep(wait)
```

---

## 🚦 Gate Check

Before moving to the next drill, verify:

- [ ] CLI starts and accepts multi-line input
- [ ] Streaming output shows tokens one-by-one (watch for batching — if nothing appears for >2s, your flush is broken)
- [ ] `/help` shows all available commands
- [ ] `/clear` resets conversation
- [ ] Switching models via env/config works
- [ ] Ctrl+C handles gracefully (doesn't crash with traceback)
- [ ] Wrong API key shows a clear error message (not a traceback)
- [ ] Network disconnect mid-stream shows an error (doesn't hang forever)
- [ ] Token counter and time-per-token display works
- [ ] You ran it for 5+ turns and verified conversation history is accumulating

> **🛑 STOP. Honest check:** Did you actually type and run the code? Or just read it?
>
> This chat client is code you'll use every single day during this course. It's your daily driver. If you just read and nodded — go back and BUILD it. Type every character. Break it. Fix it. Then move on.

---

## 📚 Cross-References

- [API Integration Patterns](07-API-Integration-Patterns.md) — Provider abstraction used here
- [Streaming, SSE & WebSockets](08-Streaming-SSE-WebSockets.md) — SSE protocol details
- [Python AI Patterns](06-Python-AI-Patterns.md) — Async patterns, error handling
- [Phase 6: AI Agents](../06-AI-Agents/) — You'll extend this chat into an agent interface later
- [Cost Awareness Culture](../00-Engineering-Mindset/05-Cost-Awareness-Culture.md) — Track those token costs!
- [Project: Multi-Provider LLM Gateway](Project-Cornerstone-Multi-Provider-LLM-Gateway.md) — This CLI becomes the gateway's test client
