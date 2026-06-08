# 🧪 Drill — Build a Tokenizer Explorer

## 🎯 Purpose & Goals

> **🛑 STOP. Think about this:**
>
> You paste a 500-character prompt into ChatGPT. How many tokens is that? How much does it cost? Why does "I" cost 1 token but "I." cost 2 tokens? Why does "私は" cost 5 tokens in one model but 3 in another?
>
> **Build an interactive tokenizer explorer** — a tool where you paste any text and see:
> - Exactly how it gets split into tokens
> - Which tokenizer was used (different models = different tokenizers)
> - Token IDs, byte representations, and merging patterns
> - Cost per request broken down by input/output
> - Visual heatmap showing which parts of text are "expensive" (token-heavy)
>
> *Before scrolling — sketch your architecture. How do different tokenizers (cl100k_base, o200k_base, p50k_base) differ? How would you visualize tokenization boundaries?*

---

**By the end of this drill, you will:**
- Understand tokenization at the byte level — not just theory, but by building
- See exactly why some text costs more than others (emoji, code, non-English)
- Build a reusable benchmarking tool for token costs
- Internalize why "prompt engineering" is really "context engineering"

**⏱ Time Budget:** 2 hours (45 min build, 45 min exploration, 30 min debug challenges)

---

## 📖 Tokenizer Architecture

```python
"""
TOKENIZER INTERNALS — Three Layers Deep

LAYER 1: Text → Tokens (what you see)
  "Hello, world!" → [15496, 11, 995, 0]

LAYER 2: Tokens → Bytes (what GPT actually sees)
  Each token maps to a sequence of bytes in the model's embedding table.
  Token 15496 = "Hello" = b'Hello'
  Token 11 = "," = b','
  Token 995 = " world" = b' world'
  Token 0 = "!" = b'!'

LAYER 3: BPE Merges (how tokenizer was trained)
  Byte-Pair Encoding starts with individual bytes, then learns merges:
  Step 1: ('H', 'e') → 'He'   (merge pair found most frequently)
  Step 2: ('l', 'l') → 'll'
  Step 3: ('He', 'll') → 'Hell'
  Step 4: ('Hell', 'o') → 'Hello'
  ...and so on for every token in the vocabulary (100k+ merges)

KEY INSIGHT: The tokenizer doesn't understand words. It finds the optimal
byte-level compression using statistical patterns from training data.
"""

# Three different tokenizers, three different behaviors
TOKENIZER_COMPARISON = {
    "o200k_base": {
        "model": "GPT-4o",
        "vocab_size": 200_000,
        "note": "Best for code + multilingual — 2x larger vocab, fewer tokens for Korean/Hindi",
        "multilingual_efficiency": "HIGH",
    },
    "cl100k_base": {
        "model": "GPT-4 / GPT-3.5",
        "vocab_size": 100_000,
        "note": "Good for English + code. Poor for non-Latin scripts.",
        "multilingual_efficiency": "MEDIUM",
    },
    "p50k_base": {
        "model": "GPT-3 / Codex",
        "vocab_size": 50_000,
        "note": "Code-optimized. Only 50k vocab. P必/汉字 cost many tokens.",
        "multilingual_efficiency": "LOW",
    },
}

"""
WHY THIS MATTERS:
- Chinese character "我" = 1 token in o200k_base, 3 tokens in cl100k_base
- "I" = 1 token. "I." = 2 tokens. "I?" = 2 tokens. "I!" = 2 tokens.
  (Because "I" is common enough to get its own token, but "I." isn't)
- "    " (4 spaces) = 1 token (common in code). "    " (3 spaces) = 3 tokens.
- "http://" = 1 token! But "https://" = 2 tokens. Just because of training data frequency.
"""
```

---

## 💻 Build Phase — Interactive Tokenizer Explorer

### Step 1: Core Tokenizer Engine

Build the engine that wraps `tiktoken` with comparison, analysis, and cost features:

```python
# tokenizer_engine.py
"""
Tokenizer Engine — Core analysis layer.

Wraps tiktoken with:
- Multi-model comparison
- Per-token breakdown with decoding
- Cost estimation
- Heatmap data generation
- Batch analysis
"""

import tiktoken
from dataclasses import dataclass, field
from typing import Optional
from enum import Enum


class TokenizerModel(Enum):
    GPT4o = "o200k_base"
    GPT4 = "cl100k_base"
    GPT35 = "cl100k_base"
    CODEX = "p50k_base"
    DAVINCI = "p50k_base"
    RERANK = "r50k_base"


@dataclass
class TokenDetail:
    """Detailed breakdown of a single token."""
    position: int          # Position in the sequence
    token_id: int          # The raw token ID
    text: str              # Decoded text representation
    bytes: bytes           # Raw bytes
    byte_length: int       # Number of bytes
    is_control: bool       # Special token (<|endoftext|>, etc.)


@dataclass
class TokenizationResult:
    """Complete result of tokenizing a piece of text."""
    text: str
    model: str
    encoding_name: str
    tokens: list[int]              # Raw token IDs
    details: list[TokenDetail]     # Per-token breakdown
    num_tokens: int
    char_count: int
    byte_count: int
    
    # Derived metrics
    chars_per_token: float = 0.0
    bytes_per_token: float = 0.0
    
    # Cost estimates (USD)
    cost_input: float = 0.0
    cost_output: float = 0.0
    cost_embedding: float = 0.0
    
    # Comparison fields
    alternative_encodings: dict[str, "TokenizationResult"] = field(default_factory=dict)
    
    def __post_init__(self):
        self.chars_per_token = self.char_count / max(self.num_tokens, 1)
        self.bytes_per_token = self.byte_count / max(self.num_tokens, 1)


class CostRates:
    """Pricing per 1M tokens for each model family (as of mid-2025)."""
    
    # ┌─────────────────────────────────────────────────────────────┐
    # │ 🔴 SENIOR MOVE: Separate cost lookup from business logic   │
    # │                                                             │
    # │ Juniors hardcode prices inline. Seniors put them in a       │
    # │ config table that can be updated independently.             │
    # └─────────────────────────────────────────────────────────────┘
    
    RATES: dict[str, dict[str, float]] = {
        "gpt-4o":           {"input": 2.50,  "output": 10.00,  "embedding": 0.13},
        "gpt-4o-mini":      {"input": 0.15,  "output": 0.60,   "embedding": 0.13},
        "gpt-4-turbo":      {"input": 10.00, "output": 30.00,  "embedding": 0.13},
        "gpt-3.5-turbo":    {"input": 0.50,  "output": 1.50,   "embedding": 0.13},
        "text-embedding-3": {"input": 0.13,  "output": 0.13,   "embedding": 0.13},
    }
    
    @classmethod
    def estimate_cost(cls, model: str, input_tokens: int, output_tokens: int = 0) -> dict:
        """Estimate cost for a given model and token counts."""
        rates = cls.RATES.get(model, cls.RATES["gpt-4o-mini"])
        return {
            "input_cost": (input_tokens * rates["input"]) / 1_000_000,
            "output_cost": (output_tokens * rates["output"]) / 1_000_000,
            "total_cost": (input_tokens * rates["input"] + output_tokens * rates["output"]) / 1_000_000,
            "rates": rates,
        }


class TokenizerEngine:
    """
    Main tokenizer engine. Analyzes text across multiple tokenizers.
    
    Usage:
        engine = TokenizerEngine()
        result = engine.analyze("Hello, world!")
        print(result.num_tokens)  # 4
        print(result.cost_input)  # $0.0000006 (for gpt-4o-mini)
    """
    
    # Mapping from model name → encoding name
    MODEL_TO_ENCODING: dict[str, str] = {
        "gpt-4o": "o200k_base",
        "gpt-4o-mini": "o200k_base",
        "gpt-4": "cl100k_base",
        "gpt-4-turbo": "cl100k_base",
        "gpt-3.5-turbo": "cl100k_base",
        "text-embedding-3-small": "cl100k_base",
        "text-embedding-3-large": "cl100k_base",
        "codex": "p50k_base",
    }
    
    def __init__(self, default_model: str = "gpt-4o-mini"):
        self.default_model = default_model
        self._encodings: dict[str, tiktoken.Encoding] = {}
    
    def _get_encoding(self, model_or_encoding: str) -> tiktoken.Encoding:
        """
        Get or cache an encoding by model name or encoding name.
        
        Cache is important — loading an encoding has overhead.
        """
        # Is it a model name? (e.g., "gpt-4o-mini")
        if model_or_encoding in self.MODEL_TO_ENCODING:
            encoding_name = self.MODEL_TO_ENCODING[model_or_encoding]
        else:
            encoding_name = model_or_encoding  # Direct encoding name
        
        if encoding_name not in self._encodings:
            self._encodings[encoding_name] = tiktoken.get_encoding(encoding_name)
        
        return self._encodings[encoding_name]
    
    def analyze(
        self,
        text: str,
        model: str | None = None,
        cost_model: str | None = None,
    ) -> TokenizationResult:
        """
        Full analysis of text through a tokenizer.
        
        Args:
            text: Text to tokenize
            model: Model name or encoding name. Defaults to gpt-4o-mini.
            cost_model: Model to use for cost estimation (defaults to model)
        
        Returns:
            TokenizationResult with full breakdown
        """
        model = model or self.default_model
        cost_model = cost_model or model
        
        encoding = self._get_encoding(model)
        tokens = encoding.encode(text)
        
        # Build per-token details
        details = []
        for i, token_id in enumerate(tokens):
            token_bytes = encoding.decode_single_token_bytes(token_id)
            try:
                token_text = token_bytes.decode("utf-8")
            except UnicodeDecodeError:
                token_text = repr(token_bytes)
            
            details.append(TokenDetail(
                position=i,
                token_id=token_id,
                text=token_text,
                bytes=token_bytes,
                byte_length=len(token_bytes),
                is_control=token_id in (50256, 50257, 50258),  # <|endoftext|>, etc.
            ))
        
        # Calculate costs
        costs = CostRates.estimate_cost(cost_model, len(tokens), output_tokens=min(len(tokens) * 2, 4096))
        
        result = TokenizationResult(
            text=text,
            model=model,
            encoding_name=encoding.name,
            tokens=tokens,
            details=details,
            num_tokens=len(tokens),
            char_count=len(text),
            byte_count=len(text.encode("utf-8")),
            cost_input=costs["input_cost"],
            cost_output=costs["output_cost"],
        )
        
        # 🔴 SENIOR MOVE: Always compare with alternative models
        # This reveals which model gives best token efficiency for this text
        result.alternative_encodings = self.compare_tokenizers(text, exclude=[model])
        
        return result
    
    def compare_tokenizers(
        self,
        text: str,
        exclude: list[str] | None = None,
    ) -> dict[str, TokenizationResult]:
        """
        Compare how the same text is tokenized by different models.
        
        This is THE most useful feature. It shows you:
        - Whether GPT-4o's tokenizer is more efficient for your language
        - How much you'd save by switching models
        - Which parts of text are "expensive" in each tokenizer
        """
        exclude = exclude or []
        results = {}
        
        # Test all major encoding types
        for encoding_name in ["o200k_base", "cl100k_base", "p50k_base"]:
            if encoding_name in exclude:
                continue
            try:
                results[encoding_name] = self.analyze(text, model=encoding_name)
            except Exception:
                continue
        
        return results
    
    def get_token_breakdown(self, text: str, model: str | None = None) -> list[str]:
        """
        Visual breakdown — show each token as a marked-up string.
        
        Returns a list where each item is one token, with the original
        boundaries clearly visible.
        """
        model = model or self.default_model
        encoding = self._get_encoding(model)
        tokens = encoding.encode(text)
        
        breakdown = []
        for token_id in tokens:
            token_text = encoding.decode([token_id])
            # Show boundaries clearly
            breakdown.append(f"[{repr(token_text)[1:-1]}]")
        
        return breakdown
    
    def heatmap_data(self, text: str, model: str | None = None) -> list[tuple[str, int]]:
        """
        Generate data for a cost heatmap.
        
        Returns list of (char, tokens_per_char) showing which characters
        cost the most tokens.
        """
        model = model or self.default_model
        encoding = self._get_encoding(model)
        tokens = encoding.encode(text)
        
        # Map each token back to its character positions
        char_positions = []
        current_pos = 0
        for token_id in tokens:
            token_text = encoding.decode([token_id])
            token_len = len(token_text)
            for i in range(token_len):
                char_positions.append((text[current_pos + i] if current_pos + i < len(text) else "", 0))
            # Assign token cost to each character in this token
            for i in range(len(char_positions) - token_len, len(char_positions)):
                char, _ = char_positions[i]
                char_positions[i] = (char, 1.0 / max(token_len, 1))
            current_pos += token_len
        
        return char_positions
```

> **🤔 YOUR TURN:** Add a `token_density_analysis()` method that finds the 10 most token-expensive substrings in a text (e.g., a long code block, a URL, a Unicode string). Why are they expensive? What would you rewrite to save tokens?

---

### Step 2: Interactive CLI Explorer

Now build the interactive exploration tool:

```python
# explorer_cli.py
"""
Interactive Tokenizer Explorer — CLI version.

Features:
- Type text, see instant tokenization breakdown
- Compare across models side by side
- Show token boundaries visually
- Cost estimation per model
- Token density heatmap (text version)
- Batch analysis from file
"""

import os
import sys
import json
from datetime import datetime
from typing import Optional


class TokenizerExplorer:
    """
    Interactive CLI for exploring tokenization.
    
    Usage:
        explorer = TokenizerExplorer()
        asyncio.run(explorer.run())
    """
    
    def __init__(self):
        self.engine = TokenizerEngine()
        self.current_model = "gpt-4o-mini"
        self.history: list[dict] = []
    
    # ── Display Methods ──────────────────────────────────────────
    
    def show_header(self):
        """Show the welcome header."""
        print(f"\n{Colors.bold('🔤 Tokenizer Explorer')}")
        print(f"  {Colors.dim('Analyze how any text becomes tokens • Compare models • Estimate costs')}")
        print(f"  {Colors.dim('Type text and hit Enter twice to analyze.')}")
        print(f"  {Colors.dim(f'Current model: {Colors.cyan(self.current_model)} │ Type /help for commands')}")
        print(f"  {Colors.dim('─' * 60)}")
    
    def show_help(self):
        """Show available commands."""
        print(f"""
{Colors.bold('Commands:')}
  /model <name>    Switch model (gpt-4o, gpt-4o-mini, gpt-4, codex)
  /compare         Show all models side by side for last text
  /cost            Show detailed cost breakdown
  /tokens          Show raw token IDs
  /save <file>     Save last analysis to JSON
  /history         Show analysis history
  /file <path>     Analyze text from a file
  /help            Show this help
  /exit            Quit

{Colors.bold('Examples:')}
  Try: "Hello, world!" — 4 tokens in o200k_base
  Try: "I am because we are" — 6 tokens
  Try: "私は、父が買った大きな白い犬を、母と一緒に公園で散歩させた。" — see multilingual cost
  Try: A Python file path to analyze code token cost
        """)
    
    def show_token_breakdown(self, result: TokenizationResult):
        """
        Show token boundaries with visual markers.
        
        [Hello] | [,] | [ world] | [!]
        Each | shows a token boundary. This reveals patterns:
        - Short tokens like "," and "?" (1 token each)
        - Long tokens like " understanding" (1 token — very common)
        - Unicode getting split across tokens
        """
        breakdown = self.engine.get_token_breakdown(result.text, result.model)
        
        print(f"\n{Colors.bold('Token Boundaries:')}")
        print(f"  {Colors.dim('|'.join(breakdown))}")
        
        print(f"\n{Colors.bold('Per-Token Details:')}")
        print(f"  {Colors.dim(f'{"Pos":<5} {"ID":<8} {"Text":<25} {"Bytes":<10} {"Cost":<10}')}")
        print(f"  {Colors.dim('─' * 60)}")
        
        for detail in result.details[:20]:  # Show first 20 tokens
            text_repr = detail.text.replace("\n", "\\n").replace("\r", "\\r")
            if len(text_repr) > 22:
                text_repr = text_repr[:20] + "…"
            
            # Color code by token length
            if detail.byte_length >= 4:
                color = Colors.RED  # Expensive (multi-byte / rare)
            elif detail.byte_length >= 2:
                color = Colors.YELLOW  # Moderate
            else:
                color = Colors.GREEN  # Efficient (1 byte per token = common)
            
            print(f"  {detail.position:<5} {detail.token_id:<8} "
                  f"{color}{text_repr:<25}{Colors.RESET} "
                  f"{detail.byte_length:<10} "
                  f"${result.cost_input * detail.byte_length / max(result.byte_count, 1):.2e}")
        
        if len(result.details) > 20:
            print(f"  {Colors.dim(f'... and {len(result.details) - 20} more tokens')}")
    
    def show_summary(self, result: TokenizationResult):
        """Show summary statistics."""
        costs = CostRates.estimate_cost(
            self.current_model,
            result.num_tokens,
            result.num_tokens * 2,
        )
        
        print(f"\n{Colors.bold('📊 Summary')}")
        print(f"  {Colors.dim('─' * 40)}")
        print(f"  {'Tokens:':<20} {Colors.bold(str(result.num_tokens))} "
              f"({result.char_count} chars, {result.byte_count} bytes)")
        print(f"  {'Chars/token:':<20} {result.chars_per_token:.1f}")
        print(f"  {'Encoding:':<20} {result.encoding_name} ({result.model})")
        print(f"  {'Input cost:':<20} {Colors.cyan(f'${costs["input_cost"]:.6f}')} "
              f"({Colors.dim(f'${costs["rates"]["input"]}/1M tokens')})")
        print(f"  {'Output cost (est):':<20} {Colors.cyan(f'${costs["output_cost"]:.6f}')}")
        print(f"  {'Total (in+out):':<20} {Colors.bold(Colors.cyan(f'${costs["total_cost"]:.6f}'))}")
        
        # 1000-request scenario
        thousand_input = costs["input_cost"] * 1000
        thousand_total = costs["total_cost"] * 1000
        print(f"\n  {Colors.dim('If you send this 1,000 times/day:')}")
        print(f"  {'Daily input cost:':<20} {Colors.yellow(f'${thousand_input:.2f}')}")
        print(f"  {'Daily total cost:':<20} {Colors.yellow(f'${thousand_total:.2f}')}")
    
    def show_comparison(self, text: str):
        """Show how this text tokenizes across ALL models."""
        results = self.engine.compare_tokenizers(text)
        
        print(f"\n{Colors.bold('🔬 Cross-Model Comparison')}")
        print(f"  {Colors.dim('─' * 50)}")
        
        sorted_results = sorted(
            results.values(),
            key=lambda r: r.num_tokens,
        )
        
        for i, result in enumerate(sorted_results):
            model_name = "GPT-4o" if "o200k" in result.encoding_name else \
                         "GPT-4/3.5" if "cl100k" in result.encoding_name else \
                         "Codex" if "p50k" in result.encoding_name else \
                         result.encoding_name
            
            savings = ""
            if i == 0 and len(sorted_results) > 1:
                savings = f" ← {Colors.green('MOST EFFICIENT')} saved {sorted_results[-1].num_tokens - result.num_tokens} tokens vs worst"
            elif i == len(sorted_results) - 1 and len(sorted_results) > 1:
                worst = result
                best = sorted_results[0]
                extra_cost = (worst.num_tokens - best.num_tokens) * 2.50 / 1_000_000
                savings = f" ← {Colors.red(f'{worst.num_tokens - best.num_tokens} extra tokens')} (${extra_cost:.6f})"
            
            print(f"  {Colors.bold(model_name):<15} {result.num_tokens:>5} tokens  "
                  f"{result.chars_per_token:.1f} chars/tok{savings}")
    
    def show_raw_tokens(self, result: TokenizationResult):
        """Show raw token IDs."""
        print(f"\n{Colors.bold('Raw Token IDs:')}")
        print(f"  {result.tokens}")
        
        # Show in groups of 20 for readability
        print(f"\n{Colors.bold('Grouped:')}")
        for i in range(0, len(result.tokens), 20):
            group = result.tokens[i:i+20]
            print(f"  [{i:>4}-{i+len(group)-1:>4}] {group}")
    
    def save_analysis(self, result: TokenizationResult, filepath: str):
        """Save analysis to JSON for later review."""
        data = {
            "timestamp": datetime.now().isoformat(),
            "text": result.text,
            "model": result.model,
            "encoding": result.encoding_name,
            "num_tokens": result.num_tokens,
            "char_count": result.char_count,
            "byte_count": result.byte_count,
            "chars_per_token": result.chars_per_token,
            "cost_input": result.cost_input,
            "cost_output": result.cost_output,
            "tokens": result.tokens,
            "details": [
                {
                    "position": d.position,
                    "token_id": d.token_id,
                    "text": d.text,
                    "byte_length": d.byte_length,
                }
                for d in result.details
            ],
        }
        
        with open(filepath, "w", encoding="utf-8") as f:
            json.dump(data, f, indent=2, ensure_ascii=False)
        
        print(f"{Colors.green(f'✓ Saved to {filepath}')}")
    
    def show_findings(self, text: str, result: TokenizationResult):
        """
        Show interesting findings about the tokenization.
        
        This is the "teachable moment" — pointing out surprising patterns
        that reveal how tokenizers actually work.
        """
        findings = []
        
        # Finding 1: Char/token ratio much lower than expected
        if result.chars_per_token < 2:
            findings.append(
                f"⚠️  {Colors.yellow('Very low chars/token ratio')} — "
                f"Only {result.chars_per_token:.1f} chars per token. "
                f"This text has many unusual characters or is very short."
            )
        
        # Finding 2: Unicode expansion
        non_ascii_chars = sum(1 for c in text if ord(c) > 127)
        if non_ascii_chars > 0:
            ascii_ratio = (result.char_count - non_ascii_chars) / max(result.char_count, 1)
            findings.append(
                f"🌐  {Colors.cyan(f'{non_ascii_chars} non-ASCII characters')} "
                f"({(1-ascii_ratio)*100:.0f}% of text). "
                f"{'These likely cost extra tokens each.' if non_ascii_chars > 10 else ''}"
            )
        
        # Finding 3: Most common token
        from collections import Counter
        token_texts = [d.text for d in result.details if not d.is_control]
        if token_texts:
            most_common = Counter(token_texts).most_common(1)[0]
            findings.append(
                f"📊  Most frequent token: {Colors.bold(repr(most_common[0]))} "
                f"appeared {most_common[1]} time{'s' if most_common[1] > 1 else ''}"
            )
        
        # Finding 4: Model comparison
        alt_results = result.alternative_encodings
        if alt_results:
            best = min(alt_results.values(), key=lambda r: r.num_tokens)
            if best.num_tokens < result.num_tokens:
                savings = result.num_tokens - best.num_tokens
                findings.append(
                    f"💡  {Colors.green(f'Switch model tip:')} "
                    f"Using {best.model} would save {savings} tokens "
                    f"({savings * 2.50 / 1_000_000:.6f}¢ input cost reduction)"
                )
        
        if findings:
            print(f"\n{Colors.bold('🔍 Findings:')}")
            for finding in findings:
                print(f"  {finding}")
    
    # ── Main Loop ────────────────────────────────────────────────
    
    def run(self):
        """Main interactive loop."""
        self.show_header()
        last_text = ""
        
        while True:
            print(f"\n{Colors.dim('─' * 60)}")
            print(f"  {Colors.bold('Enter text')} to analyze ({Colors.dim('/help for commands')})")
            print(f"  {Colors.dim('│')} ", end="")
            
            try:
                lines = []
                while True:
                    line = input()
                    if line == "":
                        break
                    lines.append(line)
                text = "\n".join(lines)
            except (EOFError, KeyboardInterrupt):
                print(f"\n{Colors.dim('Goodbye!')}")
                break
            
            if not text:
                continue
            
            # Handle commands
            if text.startswith("/"):
                cmd_parts = text[1:].strip().split(maxsplit=1)
                cmd = cmd_parts[0].lower()
                arg = cmd_parts[1] if len(cmd_parts) > 1 else ""
                
                if cmd == "exit":
                    print(f"{Colors.dim('Goodbye!')}")
                    break
                elif cmd == "help":
                    self.show_help()
                elif cmd == "model":
                    if arg in self.engine.MODEL_TO_ENCODING or arg in ["o200k_base", "cl100k_base", "p50k_base"]:
                        self.current_model = arg
                        print(f"{Colors.green(f'✓ Model switched to {arg}')}")
                    else:
                        print(f"{Colors.red(f'Unknown model: {arg}')}")
                        print(f"  Available: {', '.join(self.engine.MODEL_TO_ENCODING.keys())}")
                elif cmd == "compare":
                    if last_text:
                        self.show_comparison(last_text)
                    else:
                        print(f"{Colors.yellow('No text analyzed yet. Enter some text first.')}")
                elif cmd == "tokens":
                    if last_text:
                        last_result = self.engine.analyze(last_text, self.current_model)
                        self.show_raw_tokens(last_result)
                    else:
                        print(f"{Colors.yellow('No text analyzed yet.')}")
                elif cmd == "cost":
                    if last_text:
                        last_result = self.engine.analyze(last_text, self.current_model)
                        self.show_summary(last_result)
                    else:
                        print(f"{Colors.yellow('No text analyzed yet.')}")
                elif cmd == "save" and arg:
                    if last_text:
                        last_result = self.engine.analyze(last_text, self.current_model)
                        self.save_analysis(last_result, arg)
                    else:
                        print(f"{Colors.yellow('No text analyzed yet.')}")
                elif cmd == "history":
                    if self.history:
                        print(f"\n{Colors.bold('Analysis History:')}")
                        for i, h in enumerate(self.history[-10:], 1):
                            preview = h["text"][:50].replace("\n", "\\n")
                            print(f"  {i}. ({h['tokens']} tok) {preview}{'…' if len(h['text']) > 50 else ''}")
                    else:
                        print(f"{Colors.yellow('No history yet.')}")
                elif cmd == "file" and arg:
                    try:
                        with open(arg, "r", encoding="utf-8") as f:
                            file_text = f.read()
                        text = file_text
                        print(f"{Colors.dim(f'Loaded {len(file_text)} chars from {arg}')}")
                    except FileNotFoundError:
                        print(f"{Colors.red(f'File not found: {arg}')}")
                        continue
                else:
                    print(f"{Colors.yellow(f'Unknown command: /{cmd}. Type /help')}")
                    continue
            
            if not text.startswith("/"):
                # Analyze the text
                last_text = text
                result = self.engine.analyze(text, self.current_model)
                
                self.show_token_breakdown(result)
                self.show_summary(result)
                self.show_findings(text, result)
                
                self.history.append({
                    "text": text,
                    "tokens": result.num_tokens,
                    "model": self.current_model,
                    "timestamp": datetime.now().isoformat(),
                })
```

> **🤔 YOUR TURN:** Add a `vis` command that creates a simple terminal heatmap. For each character, show a colored dot: green (cheap, 1 char/token), yellow (moderate), red (expensive, <1 char/token). This visually reveals "why does this prompt cost so much?"

---

### Step 3: Web-Based Visualizer (Optional Extension)

For a richer experience, build a web version with FastAPI:

```python
# tokenizer_web.py — FastAPI visualizer
"""
FastAPI-based web visualizer for tokenization.

Run with: uvicorn tokenizer_web:app --reload
Visit: http://localhost:8000

Features:
- Real-time token counting as you type
- Color-coded token boundaries
- Model comparison dropdown
- Cost estimation dashboard
"""

from fastapi import FastAPI, Form
from fastapi.responses import HTMLResponse, JSONResponse
from pydantic import BaseModel

app = FastAPI(title="Tokenizer Explorer")


class AnalyzeRequest(BaseModel):
    text: str
    model: str = "gpt-4o-mini"


class AnalyzeResponse(BaseModel):
    num_tokens: int
    chars: int
    tokens: list[int]
    token_texts: list[str]
    cost_input: float
    cost_output: float
    bytes: int
    chars_per_token: float
    encoding: str


engine = TokenizerEngine()


@app.post("/analyze", response_model=AnalyzeResponse)
async def analyze(req: AnalyzeRequest):
    """Analyze text and return tokenization result."""
    result = engine.analyze(req.text, req.model)
    return AnalyzeResponse(
        num_tokens=result.num_tokens,
        chars=result.char_count,
        tokens=result.tokens,
        token_texts=[d.text for d in result.details],
        cost_input=result.cost_input,
        cost_output=result.cost_output,
        bytes=result.byte_count,
        chars_per_token=result.chars_per_token,
        encoding=result.encoding_name,
    )


@app.post("/compare")
async def compare(text: str = Form(...)):
    """Compare tokenization across all models."""
    results = engine.compare_tokenizers(text)
    return {
        name: {
            "num_tokens": r.num_tokens,
            "encoding": r.encoding_name,
            "chars_per_token": r.chars_per_token,
            "cost_input": r.cost_input,
        }
        for name, r in results.items()
    }


@app.post("/batch")
async def batch_analyze(texts: list[str], model: str = "gpt-4o-mini"):
    """Analyze multiple texts at once (for comparison)."""
    results = []
    for text in texts:
        result = engine.analyze(text, model)
        results.append({
            "text_preview": text[:100],
            "num_tokens": result.num_tokens,
            "chars": result.char_count,
        })
    return {"results": results}


# Serve a simple HTML page
HTML_PAGE = """
<!DOCTYPE html>
<html>
<head>
    <title>Tokenizer Explorer</title>
    <style>
        body { font-family: -apple-system, sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; }
        textarea { width: 100%; height: 120px; font-family: monospace; padding: 8px; }
        .stats { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 12px; margin: 16px 0; }
        .stat { background: #f5f5f5; padding: 12px; border-radius: 6px; }
        .stat .value { font-size: 24px; font-weight: bold; }
        .stat .label { font-size: 12px; color: #666; }
        .tokens { font-family: monospace; font-size: 14px; line-height: 1.8; }
        .token { display: inline; padding: 2px 4px; margin: 1px; border-radius: 3px; }
        .token-1 { background: #e8f5e9; }  /* 1 byte */
        .token-2 { background: #fff3e0; }  /* 2+ bytes */
        .token-3 { background: #fce4ec; }  /* rare/long */
        select { padding: 6px; margin: 8px 0; }
        .cost { background: #e3f2fd; padding: 12px; border-radius: 6px; margin-top: 12px; }
    </style>
</head>
<body>
    <h1>🔤 Tokenizer Explorer</h1>
    <select id="model">
        <option value="gpt-4o-mini">GPT-4o-mini (o200k_base)</option>
        <option value="gpt-4o">GPT-4o (o200k_base)</option>
        <option value="gpt-4">GPT-4 (cl100k_base)</option>
        <option value="codex">Codex (p50k_base)</option>
    </select>
    <textarea id="text" placeholder="Type or paste text here..." oninput="analyze()"></textarea>
    <div class="stats" id="stats"></div>
    <div class="tokens" id="tokens"></div>
    <div class="cost" id="cost"></div>
    
    <script>
        async function analyze() {
            const text = document.getElementById('text').value;
            const model = document.getElementById('model').value;
            
            const res = await fetch('/analyze', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({text, model})
            });
            const data = await res.json();
            
            document.getElementById('stats').innerHTML = `
                <div class="stat"><div class="value">${data.num_tokens}</div><div class="label">Tokens</div></div>
                <div class="stat"><div class="value">${data.chars}</div><div class="label">Characters</div></div>
                <div class="stat"><div class="value">${data.chars_per_token.toFixed(1)}</div><div class="label">Chars/Token</div></div>
            `;
            
            document.getElementById('tokens').innerHTML = data.token_texts.map((t, i) => 
                `<span class="token token-${Math.min(t.length, 3)}">${t.replace(/ /g, '·').replace(/\n/g, '↵')}</span>`
            ).join('');
            
            document.getElementById('cost').innerHTML = `
                <strong>Cost:</strong> Input: $${data.cost_input.toFixed(6)} | 
                Output (est): $${data.cost_output.toFixed(6)} | 
                <strong>Total: $${(data.cost_input + data.cost_output).toFixed(6)}</strong>
            `;
        }
    </script>
</body>
</html>
"""

@app.get("/")
async def root():
    return HTMLResponse(HTML_PAGE)
```

---

### Step 4: Main Entry Point

```python
# main.py
import sys
import argparse


def main():
    parser = argparse.ArgumentParser(
        description="🔤 Tokenizer Explorer — Understand tokenization interactively",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Examples:
  python main.py                       # Interactive CLI mode
  python main.py --text "Hello, world!"  # Quick analysis (no interactive)
  python main.py --compare "Hello, 世界"  # Compare all models
  python main.py --file sample.py         # Analyze a file
  python main.py --web                    # Start web UI
  python main.py --batch texts.json       # Batch analyze from JSON
        """,
    )
    parser.add_argument("--text", "-t", help="Analyze this text directly")
    parser.add_argument("--compare", "-c", help="Compare across all models")
    parser.add_argument("--file", "-f", help="Analyze text from a file")
    parser.add_argument("--model", "-m", default="gpt-4o-mini", help="Model to use")
    parser.add_argument("--web", action="store_true", help="Start web UI")
    parser.add_argument("--batch", help="Batch analyze from JSON file")
    parser.add_argument("--json", action="store_true", help="Output as JSON")
    args = parser.parse_args()
    
    engine = TokenizerEngine(default_model=args.model)
    
    if args.web:
        import uvicorn
        print(f"{Colors.green('🌐 Starting web UI at http://localhost:8000')}")
        uvicorn.run("tokenizer_web:app", host="0.0.0.0", port=8000, reload=False)
        return
    
    if args.text:
        result = engine.analyze(args.text, args.model)
        if args.json:
            import json as j
            print(j.dumps({
                "text": result.text,
                "num_tokens": result.num_tokens,
                "tokens": result.tokens,
                "encoding": result.encoding_name,
                "cost_input": result.cost_input,
                "cost_output": result.cost_output,
            }, indent=2))
        else:
            display_single(result)
        return
    
    if args.compare:
        results = engine.compare_tokenizers(args.compare)
        print(f"\n{Colors.bold(f'Comparison for: {args.compare[:60]}')}")
        for name, result in sorted(results.items(), key=lambda x: x[1].num_tokens):
            flag = "← BEST" if result == min(results.values(), key=lambda r: r.num_tokens) else ""
            print(f"  {name:<15} {result.num_tokens:>5} tokens  "
                  f"{result.chars_per_token:.1f} chars/tok  {flag}")
        return
    
    if args.file:
        with open(args.file, "r", encoding="utf-8") as f:
            text = f.read()
        result = engine.analyze(text, args.model)
        display_single(result)
        return
    
    if args.batch:
        import json as j
        with open(args.batch, "r") as f:
            texts = j.load(f)
        results = []
        for text in texts:
            result = engine.analyze(text, args.model)
            results.append({
                "text": text[:100],
                "tokens": result.num_tokens,
                "cost": result.cost_input,
            })
        print(j.dumps(results, indent=2))
        return
    
    # Interactive mode
    explorer = TokenizerExplorer()
    explorer.run()


if __name__ == "__main__":
    main()
```

---

## ✅ Good Output Examples

### Interactive Session:

```
🔤 Tokenizer Explorer
  Analyze how any text becomes tokens • Compare models • Estimate costs
  Current model: gpt-4o-mini │ Type /help for commands
  ────────────────────────────────────────────────────────────

────────────────────────────────────────────────────────────
  Enter text to analyze (/help for commands)
  │ I am because we are
  │

Token Boundaries:
  [I]|[ am]|[ because]|[ we]|[ are]

Per-Token Details:
  Pos   ID       Text                      Bytes      Cost
  ────────────────────────────────────────────────────────────
  0     40       I                         1          1.50e-09
  1     660      am                        2          3.00e-09
  2     1669     because                   7          1.05e-08
  3     719      we                        2          3.00e-09
  4     585      are                       3          4.50e-09

📊 Summary
  ────────────────────────────────────────────────────────
  Tokens:              5 (19 chars, 19 bytes)
  Chars/token:         3.8
  Encoding:            o200k_base (gpt-4o-mini)
  Input cost:          $0.000001
  Output cost (est):   $0.000001
  Total (in+out):      $0.000002

  If you send this 1,000 times/day:
  Daily input cost:    $0.00
  Daily total cost:    $0.00

🔍 Findings:
  📊  Most frequent token: ' ' appeared 4 times
```

### Model Comparison:

```
🔬 Cross-Model Comparison
  ──────────────────────────────────────────────────
  GPT-4/3.5      23 tokens  0.9 chars/tok ← RED: 18 extra tokens
  Codex          18 tokens  1.2 chars/tok
  GPT-4o          5 tokens  4.0 chars/tok ← GREEN: MOST EFFICIENT
```

### Web UI:

The web page shows a textarea where typing updates in real-time:
- Token count changes as you type
- Token boundaries color-coded (green = common token, red = rare/split)
- Cost updates live
- Model dropdown switches instantly

---

## ❌ Antipatterns & Failure Modes

### ❌ Antipattern 1: Assuming All Tokenizers Are Equal
```python
# WRONG ❌ — Using cl100k_base tokens for a GPT-4o prompt
tokens = tiktoken.get_encoding("cl100k_base").encode(prompt)
# GPT-4o uses o200k_base — different tokenization!
```

### ❌ Antipattern 2: Counting Characters, Not Tokens
```python
# WRONG ❌ — Character count is not token count
if len(prompt) < 1000:  # "Should be fine for context window"
    send_to_llm(prompt)
# 1000 Chinese characters in cl100k_base ≈ 2000-3000 tokens!
# 1000 English characters ≈ 200-300 tokens
# Never use char count for context management.
```

### ❌ Antipattern 3: Assuming Token Count Is Stable
```python
# WRONG ❌ — Tokenization can change!
# A model update can change the tokenizer, changing all your token counts.
# OpenAI changed from cl100k_base (GPT-4) to o200k_base (GPT-4o).
# A 1000-token prompt in GPT-4 might be 700 tokens in GPT-4o.
```

### ❌ Antipattern 4: Forgetting Special Tokens
```python
# WRONG ❌ — Chat format adds overhead
messages = [
    {"role": "system", "content": "..."},
    {"role": "user", "content": "..."},
]
# Each message adds: <|start|>role<|content|> tokens
# ChatML format overhead: ~5-10 tokens per message
# Assistant prefix token: <|assistant|> (1 token)
```

### ❌ Antipattern 5: No Batch Cost Awareness
```python
# WRONG ❌ — Ignoring cumulative costs
# 50,000 requests/day × 500 tokens × 2 (in+out) = 50M tokens/day
# At GPT-4o-mini prices: 50M × ($0.15 + $0.60)/1M = $37.50/day
# At GPT-4o prices: 50M × ($2.50 + $10.00)/1M = $625/day
# The difference: 16.7x — more than the quality difference justifies
```

---

## 🧪 Debug Challenge — Find the 7 Bugs

This tokenizer has bugs. Find them:

```python
import tiktoken

class BrokenTokenizer:
    def __init__(self):
        self.encodings = {}
    
    def count_tokens(self, text, model="gpt-4o-mini"):
        if model == "gpt-4o-mini":
            enc = self.encodings.get("o200k_base")
            if not enc:
                enc = tiktoken.get_encoding("o200k_base")
        elif model == "gpt-4":
            enc = tiktoken.get_encoding("p50k_base")  # Wrong encoding!
        else:
            enc = tiktoken.get_encoding("cl100k_base")
        
        tokens = enc.encode(text)
        
        # Calculate cost
        cost = len(tokens) * 0.15  # Wrong — cost is per 1M tokens
        return len(tokens), cost
    
    def compare(self, texts):
        results = []
        for text in texts:
            tokens, cost = self.count_tokens(text)
            results.append({
                "text": text,
                "tokens": tokens,
                "cost": cost,
                "chars_per_token": len(text) / tokens,  # Can divide by zero!
            })
        return results

broker = BrokenTokenizer()
print(broker.count_tokens("Hello!", "gpt-4o-mini"))  # Works
print(broker.count_tokens("Hello!", "gpt-4"))  # Wrong encoding
print(broker.compare(["Hello", "", "World"]))  # Divide by zero on ""

# Bonus bug: encoding cache never used after first call
```

<details>
<summary>👀 Find the bugs first, then check</summary>

1. **Bug: Wrong encoding for GPT-4** — `gpt-4` uses `cl100k_base`, not `p50k_base`. `p50k_base` is for Codex.

2. **Bug: Encoding cache broken** — After `self.encodings.get("o200k_base")`, the encoding is stored in local variable `enc` but never stored back in `self.encodings`. Next call will miss cache again.

3. **Bug: Cost calculation wrong** — `len(tokens) * 0.15` treats cost as per-token. OpenAI pricing is per 1M tokens. Should be `len(tokens) * 0.15 / 1_000_000`.

4. **Bug: Divide by zero** — `len(text) / tokens` when `tokens == 0` (empty string). Causes `ZeroDivisionError`.

5. **Bug: No model-to-encoding mapping stored** — `self.encodings` keys are encoding names, but the code tries to get model names. Should store and look up consistently.

6. **Bug: `enc.encode(text)` with empty or None** — Not handled. Empty string returns `[]` which is fine, but `None` would crash.

7. **Bug: No cost model separation** — The cost calculation uses a fixed rate regardless of model. GPT-4o costs 10x more than GPT-4o-mini. Should use `CostRates` lookup.
</details>

---

## 🧪 Exploration Challenges

Use the explorer to answer these questions. **Don't guess — actually run the tool:**

### Challenge 1: The "Space" Mystery
Why does "Hello" cost 1 token but " Hello" (with space) also cost 1 token? 
- Try: `"Hello"` vs `" Hello"` vs `"  Hello"` vs `"   Hello"`
- Find the boundary where it splits into multiple tokens

### Challenge 2: The "I" Experiment
- Try: `"I"` vs `"I."` vs `"I?"` vs `"I!"` vs `"I am"`
- Why does punctuation add tokens for some words but not others?

### Challenge 3: Code Is Cheap
- Try: `"def fib(n): return n if n <= 1 else fib(n-1) + fib(n-2)"`
- Compare chars/token for code vs English. Why is code more token-efficient?

### Challenge 4: Multilingual Cost
```
Samples to try (same meaning, 3 languages):
English: "I am learning artificial intelligence engineering"
Korean:  "나는 인공지능 엔지니어링을 배우고 있습니다"
Hindi:   "मैं कृत्रिम बुद्धिमत्ता इंजीनियरिंग सीख रहा हूं"
Japanese: "私は人工知能エンジニアリングを学んでいます"
```
- Compare token counts across o200k_base vs cl100k_base
- Calculate the cost difference for a 50,000-token-per-day workload

### Challenge 5: Tokenize Your Own Code
Point the analyzer at your own Python files:
```
/file chat.py  # Your CLI chat from the previous drill
```
- Find the single most token-expensive function signature​‍‌‍‌​​‍​‌​​​
- What's the chars/token ratio of your code vs docstrings vs comments?

### Challenge 6: The JSON Token Tax
Compare these three representations of the same data:
```python
# Representation 1: JSON (most tokens)
json.dumps(data)

# Representation 2: Compact (no spaces)
json.dumps(data, separators=(",", ":"))

# Representation 3: Custom format
f"{data['name']}|{data['age']}|{data['city']}"
```
- Which uses 50% fewer tokens than JSON?
- For a large list of 10,000 items, how much would you save per API call?

---

## 🚦 Gate Check

Before moving to the project, verify:

- [ ] CLI explorer runs and accepts text input
- [ ] Token boundaries are displayed visually with examples
- [ ] Model comparison shows clear differences between o200k_base, cl100k_base, p50k_base
- [ ] Cost estimation shows both per-request and daily scenarios
- [ ] `/compare` works and shows which model is most efficient
- [ ] Non-ASCII text (Chinese, Korean, Hindi, emoji) is handled correctly
- [ ] Empty text doesn't crash (returns 0 tokens)
- [ ] File analysis works
- [ ] Debug challenge — you found all 7 bugs and fixed them
- [ ] You ran at least 3 of the 6 exploration challenges and noted your findings

> **🛑 STOP. Real talk:**
>
> Tokenizers feel like a boring detail. They are not. Bad tokenization is the #1 hidden cause of:
> - Context window overflow (fits in dev, breaks in prod)
> - Unexpected costs (multilingual content 3x+ more expensive than expected)
> - Poor model performance (truncation mid-sentence because you miscounted)
>
> **This explorer is your debugging tool for the rest of the course.** When a RAG pipeline fails, a prompt gets truncated, or costs spike — your first question should be: "What did the tokenizer do?"

---

## 📚 Cross-References

- [Tokenizers Deep Dive](02-Tokenizers-Deep-Dive.md) — Theory behind BPE tokenization
- [Context Windows & Attention](03-Context-Windows-Attention.md) — Why accurate token counting matters for context fitting
- [Cost Awareness Culture](../00-Engineering-Mindset/05-Cost-Awareness-Culture.md) — Token economics in production
- [Structured Outputs Basics](05-Structured-Outputs-Basics.md) — Token overhead of JSON mode vs structured outputs
- [Drill: Build CLI Chat](Drill-Build-CLI-Chat.md) — Your chat client uses this tokenizer for cost tracking
- [Project: Multi-Provider LLM Gateway](Project-Cornerstone-Multi-Provider-LLM-Gateway.md) — Token counting is a core gateway feature
