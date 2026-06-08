# 06 — Fine-Tuning for Function Calling: Teaching Models to Use Tools

## 🎯 Purpose & Goals

> 🛑 STOP. Before we talk about function-calling training data and tool formats, you need to understand a fundamental truth about LLMs:

**LLMs don't "use tools." They GENERATE tool calls.**

There's no magical "tool-use neuron" in the model. When a model "calls a function," it's simply generating text that conforms to a function-call format (JSON, XML, etc.). The model learned, through training data, that when a user asks "what's the weather in Tokyo," the correct next tokens are `{"function": "get_weather", "args": {"location": "Tokyo"}}`.

This means:
- Function-calling is a FORMAT, not a capability
- Any model that can generate structured text can be trained to call functions
- The challenge is not "teaching tool use" — it's teaching the model to (a) know WHEN to call which function and (b) extract the CORRECT arguments from context

**The goal of this module: Learn how to fine-tune models for reliable function calling, with proper training data, evaluation, and debugging of failures.**

---

### 🤔 Discovery Question 1: The Function Choice Problem

You fine-tune a model on 5,000 examples of weather lookup, calendar management, and email drafting.

| Test Scenario | Base Model Accuracy | FT Model Accuracy |
|--------------|-------------------|-------------------|
| Weather lookup | 72% | 94% |
| Calendar management | 68% | 91% |
| Email drafting | 65% | 89% |
| A NEW function: "restaurant booking" | 60% | **41%** |

**🤔 Before reading on, answer these:**

1. The FT model improved on ALL the training functions. But it got WORSE at calling a new function it wasn't trained on — even though the base model could do it. What happened? (Hint: The FT model "specialized" in the 3 trained functions and started PREFERRING them over unknown functions. It learned a BIAS toward known functions.)

2. When you look at the restaurant booking failures, you notice: in 35% of failures, the model calls "get_weather" instead of "book_restaurant." In 30%, it calls "create_calendar_event." Why would a model trained on weather, calendar, and email prefer those functions over a new one? What does this tell you about how the model REPRESENTS function choice internally?

3. **The hardest question:** You're building a platform where users can ADD custom functions. You can't retrain the model every time a user adds a function. **How do you fine-tune a model to be "ready to learn new functions" without actually training on every possible function?** What training data structure would teach the model that "function calling is a general skill, not a memorized mapping of function names to tasks"?

---

### 🤔 Discovery Question 2: The Argument Extraction Failure

The model correctly chooses "get_weather" but passes WRONG arguments:

```
User: "What's the weather in Paris tomorrow?"
Model calls: get_weather(location="France", date="2024-01-15")
```

The location is wrong ("France" instead of "Paris") and the date is wrong (a hardcoded date instead of "tomorrow").

**🤔 Before reading on, answer these:**

1. The model correctly identified "this needs weather lookup." That's the HARD part? But it failed at argument extraction, which seems easier. Why can the model make the right FUNCTION choice but the wrong ARGUMENT extraction? (Hint: Function choice is a classification problem — pick 1 of 3. Argument extraction is a SPAN-PREDICTION problem — find the right text in the user's query and map it to the right parameter. These use different internal mechanisms.)

2. You look at the training data. Your argument examples look like this:
   ```json
   {"location": "Paris", "date": "tomorrow"}
   ```
   But the user might say: "what's the forecast for Paris, France tomorrow afternoon?" — which contains "Paris, France" not just "Paris." Your training examples have CLEAN extractions. Real queries have MESSY phrasing. **What training data would you create to handle the FULL VARIETY of ways users express the same information?**

3. **The hardest question:** Some parameters are OPTIONAL. The user says "weather in Paris" without specifying a date. Should the model: (a) ask for the date, (b) default to today, (c) call the function with date=None? Your training data has EXHAUSTIVE parameters in every example because you generated them from templates. **How do you train the model to handle MISSING parameters when your training data always has complete information?** What's the training data pattern for "optional parameters" and "default values"?

---

### 🤔 Discovery Question 3: The Tool Hallucination

Your FT function-calling model sometimes invokes functions that DON'T EXIST:

```
User: "Can you help me with my homework?"
Model calls: do_homework(subject="math")
```

But there's no `do_homework` function in your system. The model HALLUCINATED a function name.

**🤔 Before reading on, answer these:**

1. Where did "do_homework" come from? It's not in your training data for function names (get_weather, create_calendar_event, send_email). Did the model invent it? Or did it learn this from its PRE-TRAINING data? (Hint: The base model was pre-trained on the entire internet, including millions of software documentation examples where functions like `do_homework()` appear.)

2. Your FT model sees 3 function names in 5,000 training examples. But it knows MILLIONS of function names from pre-training. The fine-tuning TELLS the model "these are the valid functions" but doesn't ERASE the pre-training knowledge. **Why does the model fall back to pre-training function names under some conditions?** (Hint: When the user query doesn't clearly match any trained function, the model's pre-training prior dominates.)

3. **The hardest question:** You cannot enumerate every function a user might request. But your model MUST ONLY call the 3 defined functions. Calling hallucinated functions is a safety issue (could execute arbitrary actions). **Design a ML technique that prevents the model from generating function names outside the allowed set.** The obvious solution is output filtering (regex match). But what if the model calls "get_the_weather" instead of "get_weather"? Or "weather()" instead of "get_weather()"? Can you design a training approach that makes the model RELIABLY stay within the allowed function set?

---

## ⏱ Time Budget

| Section | Time |
|---------|------|
| Discovery Questions | 30 min |
| Concept Deep-Dive | 1 hr |
| Code Examples (2) | 2 hr |
| Drills (3) | 1 hr |
| Gate Check | 30 min |
| **Total** | **~5 hr** |

---

## 📖 Concept Deep-Dive

### 1. How Function Calling Actually Works in LLMs

Internally, function calling is NEXT-TOKEN PREDICTION with a constrained format:

```
User: What's the weather in Paris?
Assistant: [thinking] I need to call get_weather with location=Paris
            [generating] {"function": "get_weather", "args": {"location": "Paris"}}
```

The model learns to:
1. **Detect intent**: Does the user want a function call or a text response?
2. **Select function**: Which of the available functions matches this intent?
3. **Extract arguments**: What values should each parameter have?
4. **Format output**: Generate valid JSON (or whatever format you chose)

**Key insight**: Steps 1-3 are DISTINCT SKILLS that share model weights. Improving function selection (step 2) might WORSEN intent detection (step 1) because both compete for the same representational capacity.

### 2. Training Data Structure

For function-calling fine-tuning, your training examples must include:

```
System: You have access to these functions:
- get_weather(location: str, date: str): Get weather forecast

User: What's the weather in Paris tomorrow?

Assistant: get_weather(location="Paris", date="2024-01-15")
```

**Critical elements:**
- **Function definitions** in the system prompt (same format as inference)
- **User query** that triggers the function
- **Correct function call** as the target output
- **Multiple functions per example** should appear in training to teach discrimination

### 3. Single-Function vs. Multi-Function Training

| Approach | How It Works | Pro | Con |
|----------|-------------|-----|-----|
| Single-function per example | Each example has 1 user query and teaches 1 function call | Simple to generate | Model doesn't learn COMPARISON between functions |
| Multi-function per example | Each example has 2+ relevant functions, model must choose | Teaches discrimination | Harder to generate, more tokens per example |
| Negative examples | User query with NO matching function → model says "I can't do that" | Prevents hallucination | Requires careful negative example design |

**Senior Engineer Insight**: Multi-function training is the single biggest quality lever. A model trained on single-function examples will fail when given 3 functions to choose from at inference time.

### 4. The Rejection Training Pattern

One of the most CRITICAL patterns for production function calling: teaching the model WHEN NOT TO CALL A FUNCTION.

Your training data must include examples where:
- User asks something no function can do → model should say "I can't help with that" (not hallucinate)
- User query is ambiguous → model should ASK for clarification (not guess)
- User query matches multiple functions → model should pick the BEST match or ask

**The distribution matters**: If your training data is 100% function-calling examples, the model will ALWAYS try to call a function. You need 10-20% "no function needed" examples.

### 5. Evaluation of Function Calling

Standard text generation metrics DON'T work for function calling. You need specialized metrics:

| Metric | What It Measures | How to Calculate |
|--------|-----------------|-----------------|
| Function Selection Accuracy | Did the model pick the RIGHT function? | Exact match of function name |
| Argument Extraction F1 | Did the model extract correct args? | Token-level F1 on extracted values |
| JSON Validity Rate | What % of outputs parse as valid JSON? | JSON.parse success rate |
| Hallucination Rate | What % of outputs call nonexistent functions? | Function name not in allowed set |
| Rejection Accuracy | When no function matches, does model refuse? | Correct rejection rate |

---

## 💻 CODE EXAMPLES

### Example 1: The Broken Function-Calling FT (Full of Bugs)

```python
"""
WHAT NOT TO DO — A function-calling fine-tuning script with 10+ hidden bugs.

This "works" in the sense that training runs and loss decreases.
But the resulting model hallucinates function names and extracts wrong arguments.
"""

import json
from datasets import Dataset
from transformers import AutoTokenizer, AutoModelForCausalLM, TrainingArguments
from trl import SFTTrainer
import torch

# BUG 1: No function definitions in the training data
# The training examples have raw function calls without function definitions.
# The model never learns to MATCH a function to its description — it just memorizes
# "weather query → call get_weather"

training_data = [
    {
        "messages": [
            {"role": "user", "content": "What's the weather in Paris?"},
            {"role": "assistant", "content": 'get_weather(location="Paris", date="today")'}
        ]
    },
    {
        "messages": [
            {"role": "user", "content": "Schedule a meeting tomorrow at 3pm"},
            {"role": "assistant", "content": 'create_event(title="Meeting", time="tomorrow 3pm")'}
        }
    ],
    # ... 5,000 more examples like this
]

# BUG 2: Single function per example
# Every example has exactly ONE correct function. Model never learns
# to CHOOSE between competing functions.

# BUG 3: No negative examples
# No examples where the user asks something the model can't do.
# Model will ALWAYS try to call a function, even for "hello how are you"

# BUG 4: No formatting variety
# Every function call uses EXACTLY the same formatting.
# Model can't handle format variations at inference time.

# BUG 5: No argument type variety
# "date" is always a string like "today" or "tomorrow"
# Never teaches the model to extract specific dates like "2024-01-15"

model_name = "mistralai/Mistral-7B-v0.1"  # BUG 6: Using base model, not instruct model
# Base models need specific chat templates. Instruct models handle multi-turn better.

tokenizer = AutoTokenizer.from_pretrained(model_name)

# BUG 7: No chat template applied
# The training data has raw {"role", "content"} dicts
# But the tokenizer's apply_chat_template is NOT called
# The model sees raw JSON, not formatted messages

def format_func(example):
    """Format examples for training."""
    # BUG 8: This doesn't apply the chat template
    # The messages are just concatenated with "User: " and "Assistant: " prefixes
    # But the actual tokenizer might expect different formatting
    text = ""
    for msg in example['messages']:
        role = msg['role']
        content = msg['content']
        text += f"{role}: {content}\n"
    return {"text": text}

dataset = Dataset.from_list(training_data)
dataset = dataset.map(format_func)

# BUG 9: No LoRA config
# Full fine-tuning 7B model — will take forever and might OOM
# Most function-calling FT should use LoRA or QLoRA

training_args = TrainingArguments(
    output_dir="./ft-function-calling",
    per_device_train_batch_size=1,
    # BUG 10: No gradient accumulation
    # Effective batch size = 1, very unstable training
    learning_rate=2e-5,  # BUG 11: Too high for LoRA (if using LoRA)
    # Standard LR: 5e-5 for full FT, 1e-4 to 2e-4 for LoRA
    num_train_epochs=3,
    logging_steps=10,
    save_steps=500,
    # BUG 12: No evaluation during training
    # No eval_dataset, no eval_steps — blind training
)

model = AutoModelForCausalLM.from_pretrained(
    model_name,
    # BUG 13: No quantization config
    # Loading in full precision — might OOM on consumer GPU
    torch_dtype=torch.float16,
)

trainer = SFTTrainer(
    model=model,
    args=training_args,
    train_dataset=dataset,
    # BUG 14: No formatting_func, no response_template
    # The trainer doesn't know which part is the "response" to compute loss on
    # It computes loss on ALL tokens, including the prompt
    # This teaches the model to predict the user's query, not just the function call
)

trainer.train()
```

#### 🔍 Find The Bugs

| # | Bug | Why It's Bad | Fix |
|---|-----|-------------|-----|
| 1 | No function definitions in training | Model memorizes "weather → get_weather" without understanding function purpose | Include function definitions as system messages |
| 2 | All examples have 1 function | Model never learns to choose between multiple valid functions | Create examples with 2+ relevant functions |
| 3 | No negative examples | Model always calls a function, never rejects | Add 10-20% "I can't help with that" examples |
| 4 | No formatting variety | Model breaks on format variations at inference | Train with 3+ output formats: JSON, function(), XML |
| 5 | No argument type variety | Model can't handle different date formats, number types | Add varied parameter formats in training |
| 6 | Using base model, not instruct | Base models struggle with multi-turn and instruction following | Use instruct-tuned variant |
| 7 | No chat template applied | Model sees raw JSON, not formatted conversation | Use tokenizer.apply_chat_template() |
| 8 | Format function doesn't use chat template | Training format ≠ inference format | Match training format to inference exactly |
| 9 | No LoRA config | Full fine-tuning is slow and expensive | Use LoRA/QLoRA (rank=16, target modules) |
| 10 | No gradient accumulation | Batch size 1 = noisy gradients | Add gradient_accumulation_steps=4-8 |
| 11 | Incorrect LR for LoRA | 2e-5 is fine for full FT, too low for LoRA | Use 1e-4 to 2e-4 for LoRA |
| 12 | No evaluation during training | Can't detect overfitting or training failures | Add eval_dataset and eval_steps |
| 13 | No quantization | 7B in fp16 needs ~14GB, with optimizer state ~28GB+ | Add BitsAndBytesConfig for 4-bit |
| 14 | Loss computed on all tokens | Model learns to predict user query, not just function call | Use response_template to mask prompt loss |

---

### Example 2: Production Function-Calling Fine-Tuning (Fixed)

```python
"""
Production Function-Calling Fine-Tuning Pipeline.

Features:
- Proper function definitions in training data
- Multi-function choice examples
- Negative examples (rejection training)
- Format variety
- Argument type diversity
- Proper loss masking (only compute loss on response)
- Comprehensive evaluation
"""

import json
import random
from typing import Dict, List, Optional, Literal
from dataclasses import dataclass, field
from datasets import Dataset
from transformers import (
    AutoTokenizer,
    AutoModelForCausalLM,
    BitsAndBytesConfig,
    TrainingArguments,
)
from trl import SFTTrainer
from peft import LoraConfig, get_peft_model
import torch
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# ──────────────────────────────────────────────
# FUNCTION DEFINITIONS
# ──────────────────────────────────────────────

# These are the functions the model should learn to call.
# They appear in training data as system messages AND at inference time.

FUNCTIONS = [
    {
        "name": "get_weather",
        "description": "Get the weather forecast for a location",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "City name, e.g., Paris, Tokyo",
                },
                "date": {
                    "type": "string",
                    "description": "Date for forecast. Use ISO format (YYYY-MM-DD) or relative (today, tomorrow)",
                },
                "unit": {
                    "type": "string",
                    "enum": ["celsius", "fahrenheit"],
                    "description": "Temperature unit. Default: celsius",
                },
            },
            "required": ["location"],
        },
    },
    {
        "name": "create_calendar_event",
        "description": "Create a new calendar event",
        "parameters": {
            "type": "object",
            "properties": {
                "title": {"type": "string", "description": "Event title"},
                "start_time": {
                    "type": "string",
                    "description": "Start time in ISO format (YYYY-MM-DDTHH:MM)",
                },
                "end_time": {
                    "type": "string",
                    "description": "End time in ISO format (YYYY-MM-DDTHH:MM)",
                },
                "description": {"type": "string", "description": "Optional description"},
            },
            "required": ["title", "start_time"],
        },
    },
    {
        "name": "send_email",
        "description": "Send an email to a recipient",
        "parameters": {
            "type": "object",
            "properties": {
                "to": {"type": "string", "description": "Recipient email address"},
                "subject": {"type": "string", "description": "Email subject"},
                "body": {"type": "string", "description": "Email body text"},
                "priority": {
                    "type": "string",
                    "enum": ["low", "normal", "high"],
                    "description": "Email priority. Default: normal",
                },
            },
            "required": ["to", "subject", "body"],
        },
    },
]


# ──────────────────────────────────────────────
# TRAINING DATA GENERATOR
# ──────────────────────────────────────────────

class FunctionCallingDataGenerator:
    """
    Generate training data for function-calling fine-tuning.
    
    This is NOT a full data generator — it's a TEMPLATE that shows
    the data structures and patterns you need.
    
    You should replace the synthetic examples with REAL data
    from your actual use case.
    """
    
    def __init__(self, functions: List[Dict], tokenizer):
        self.functions = functions
        self.tokenizer = tokenizer
        
        # Add a "none" pseudo-function for rejection training
        self.all_options = functions + [{
            "name": "no_function_needed",
            "description": "Use when the user's request doesn't require calling any function",
            "parameters": {"type": "object", "properties": {}},
        }]
    
    def create_function_list_prompt(self, funcs: List[Dict]) -> str:
        """Format function definitions as a system message."""
        lines = ["You have access to the following functions. Use them if needed:"]
        for f in funcs:
            lines.append(f"\nFunction: {f['name']}")
            lines.append(f"Description: {f['description']}")
            lines.append("Parameters:")
            for param_name, param_info in f['parameters']['properties'].items():
                required = param_name in f['parameters'].get('required', [])
                req_str = " (required)" if required else " (optional)"
                enum_str = ""
                if 'enum' in param_info:
                    enum_str = f" Options: {', '.join(param_info['enum'])}"
                lines.append(f"  - {param_name}: {param_info['description']}{req_str}{enum_str}")
        return "\n".join(lines)
    
    def create_function_call(self, func_name: str, args: Dict) -> str:
        """Format a function call as the assistant's response."""
        args_str = ", ".join(f'{k}="{v}"' if isinstance(v, str) else f"{k}={v}" for k, v in args.items())
        return f"FUNCTION_CALL: {func_name}({args_str})"
    
    def create_rejection_response(self, reason: str = "") -> str:
        """Format a "can't help" response."""
        base = "NO_FUNCTION_NEEDED"
        if reason:
            return f"{base}: {reason}"
        return base
    
    # ─── Example Generators ───
    
    def generate_single_function_examples(self, num: int) -> List[Dict]:
        """Generate examples with EXACTLY one valid function to call."""
        examples = []
        scenarios = [
            {
                "function": "get_weather",
                "queries": [
                    "What's the weather in {location} {date}?",
                    "How's the forecast for {location}?",
                    "Is it going to rain in {location} {date}?",
                    "What's the temperature in {location} right now?",
                    "Should I bring an umbrella to {location} today?",
                ],
                "args_generator": lambda: {
                    "location": random.choice(["Paris", "Tokyo", "London", "New York", "Sydney"]),
                    "date": random.choice(["today", "tomorrow", "2024-06-15", "next Monday"]),
                },
            },
            {
                "function": "create_calendar_event",
                "queries": [
                    "Schedule a meeting called {title} on {date}",
                    "Set up a {title} appointment for {date}",
                    "Add {title} to my calendar at {time}",
                    "Remind me about {title} on {date}",
                    "Book a {title} session for next week",
                ],
                "args_generator": lambda: {
                    "title": random.choice(["Team standup", "Client call", "Review meeting", "Lunch with team", "Doctor appointment"]),
                    "start_time": f"2024-06-{random.randint(10, 20)}T{random.randint(9, 16)}:00",
                },
            },
            {
                "function": "send_email",
                "queries": [
                    "Send an email to {to} about {subject}",
                    "Email {to} regarding {subject}",
                    "Write an email to {to} with subject {subject}",
                    "Draft an email for {to}: {subject}",
                ],
                "args_generator": lambda: {
                    "to": random.choice(["alice@company.com", "bob@work.com", "charlie@team.org"]),
                    "subject": random.choice(["Project update", "Meeting notes", "Action items", "Weekly report"]),
                    "body": random.choice([
                        "Please find the attached report.",
                        "Let me know your thoughts on this.",
                        "Here are the action items from today's meeting.",
                    ]),
                },
            },
        ]
        
        for _ in range(num):
            scenario = random.choice(scenarios)
            func_name = scenario["function"]
            args = scenario["args_generator"]()
            query = random.choice(scenario["queries"]).format(**args)
            
            # Build messages
            messages = [
                {"role": "system", "content": self.create_function_list_prompt(
                    [f for f in self.functions if f["name"] == func_name]
                )},
                {"role": "user", "content": query},
                {"role": "assistant", "content": self.create_function_call(func_name, args)},
            ]
            examples.append({"messages": messages})
        
        return examples
    
    def generate_multi_function_examples(self, num: int) -> List[Dict]:
        """
        Generate examples with MULTIPLE valid function choices.
        This teaches the model to DISCRIMINATE between functions.
        """
        examples = []
        
        # These scenarios have 2 relevant functions
        # The model must choose the BEST one
        multi_scenarios = [
            # Weather + Calendar: user wants weather-based planning
            {
                "relevant_functions": ["get_weather", "create_calendar_event"],
                "correct": "get_weather",
                "queries": [
                    "Will it rain in London this weekend? I need to plan outdoor activities.",
                    "What's the forecast for Tokyo on Friday? I'm scheduling a picnic.",
                    "Is Paris going to be sunny next Tuesday?",
                ],
                "args_generator": lambda func: {
                    "location": random.choice(["London", "Tokyo", "Paris", "New York"]),
                    "date": random.choice(["this weekend", "Friday", "next Tuesday"]),
                } if func == "get_weather" else {
                    "title": "Outdoor activity",
                    "start_time": "2024-06-15T14:00",
                },
            },
            # Calendar + Email: user wants to schedule + notify
            {
                "relevant_functions": ["create_calendar_event", "send_email"],
                "correct": "create_calendar_event",
                "queries": [
                    "Schedule the project review for next Tuesday at 2pm.",
                    "Book a meeting with the design team for Thursday morning.",
                    "Add a reminder for the quarterly review on June 20th.",
                ],
                "args_generator": lambda func: {
                    "title": random.choice(["Project review", "Design team meeting", "Quarterly review"]),
                    "start_time": f"2024-06-{random.randint(10, 25)}T{random.randint(9, 16)}:00",
                } if func == "create_calendar_event" else {
                    "to": "team@company.com",
                    "subject": "Meeting scheduled",
                    "body": "A meeting has been scheduled.",
                },
            },
        ]
        
        for _ in range(num):
            scenario = random.choice(multi_scenarios)
            correct_func = scenario["correct"]
            args = scenario["args_generator"](correct_func)
            query = random.choice(scenario["queries"])
            
            # Include ALL functions (not just relevant ones) for hardest discrimination
            messages = [
                {"role": "system", "content": self.create_function_list_prompt(self.functions)},
                {"role": "user", "content": query},
                {"role": "assistant", "content": self.create_function_call(correct_func, args)},
            ]
            examples.append({"messages": messages})
        
        return examples
    
    def generate_negative_examples(self, num: int) -> List[Dict]:
        """
        Generate examples where NO function should be called.
        Critical for preventing hallucination.
        """
        examples = []
        
        rejection_queries = [
            "Hello, how are you?",
            "Tell me a joke.",
            "What's the capital of France?",
            "Explain quantum computing in simple terms.",
            "Write a poem about AI.",
            "What do you think about the weather?",
            "Can you help me with my math homework?",
            "Translate 'hello' to Spanish.",
            "What's the meaning of life?",
            "Tell me about yourself.",
        ]
        
        for query in rejection_queries[:num]:
            messages = [
                {"role": "system", "content": self.create_function_list_prompt(self.functions)},
                {"role": "user", "content": query},
                {"role": "assistant",
                 "content": self.create_rejection_response(
                     "This request doesn't require any of my available functions."
                 )},
            ]
            examples.append({"messages": messages})
        
        return examples
    
    def generate_all_data(
        self,
        num_single: int = 3000,
        num_multi: int = 1500,
        num_negative: int = 500,
    ) -> List[Dict]:
        """Generate complete training dataset."""
        all_examples = []
        all_examples.extend(self.generate_single_function_examples(num_single))
        all_examples.extend(self.generate_multi_function_examples(num_multi))
        all_examples.extend(self.generate_negative_examples(num_negative))
        random.shuffle(all_examples)
        logger.info(
            f"Generated {len(all_examples)} examples: "
            f"{num_single} single, {num_multi} multi, {num_negative} negative"
        )
        return all_examples


# ──────────────────────────────────────────────
# TRAINING
# ──────────────────────────────────────────────

def train_function_calling_model(
    train_data: List[Dict],
    model_name: str = "mistralai/Mistral-7B-Instruct-v0.2",
    output_dir: str = "./ft-function-calling-v2",
):
    """
    Train a function-calling model with proper settings.
    
    Key differences from the broken version:
    1. Uses instruct-tuned base model
    2. Proper chat template formatting
    3. LoRA for efficiency
    4. Loss masking (only train on assistant responses)
    5. Multi-function training data
    6. Negative examples for rejection training
    """
    
    # ─── Load tokenizer ───
    tokenizer = AutoTokenizer.from_pretrained(model_name)
    tokenizer.pad_token = tokenizer.eos_token
    tokenizer.padding_side = "right"
    
    # ─── Format function ───
    def format_chat_template(example):
        """Apply the model's chat template to training examples."""
        # CRITICAL: Use the tokenizer's own chat template
        # This ensures training format matches inference format EXACTLY
        text = tokenizer.apply_chat_template(
            example['messages'],
            tokenize=False,
            add_generation_prompt=False,
        )
        return {"text": text}
    
    # Create dataset
    dataset = Dataset.from_list(train_data)
    dataset = dataset.map(format_chat_template)
    
    # Split into train/eval
    split_dataset = dataset.train_test_split(test_size=0.05, seed=42)
    
    # ─── Quantization ───
    bnb_config = BitsAndBytesConfig(
        load_in_4bit=True,
        bnb_4bit_compute_dtype=torch.float16,
        bnb_4bit_quant_type="nf4",
        bnb_4bit_use_double_quant=True,
    )
    
    # ─── Model ───
    model = AutoModelForCausalLM.from_pretrained(
        model_name,
        quantization_config=bnb_config,
        device_map="auto",
        torch_dtype=torch.float16,
        attn_implementation="flash_attention_2",  # Faster training
    )
    
    # ─── LoRA Config ───
    lora_config = LoraConfig(
        r=16,
        lora_alpha=32,
        target_modules=[
            "q_proj", "k_proj", "v_proj", "o_proj",  # Attention
            "gate_proj", "up_proj", "down_proj",       # MLP
        ],
        lora_dropout=0.05,
        bias="none",
        task_type="CAUSAL_LM",
    )
    
    model = get_peft_model(model, lora_config)
    model.print_trainable_parameters()  # Should show ~0.1-0.5% trainable
    
    # ─── Training Arguments ───
    training_args = TrainingArguments(
        output_dir=output_dir,
        per_device_train_batch_size=4,
        per_device_eval_batch_size=4,
        gradient_accumulation_steps=4,  # Effective batch size: 4 × 4 = 16
        learning_rate=2e-4,  # LoRA LR (not full FT LR)
        lr_scheduler_type="cosine",
        warmup_ratio=0.03,
        num_train_epochs=3,
        logging_steps=10,
        eval_steps=100,
        save_steps=500,
        evaluation_strategy="steps",
        save_total_limit=3,
        load_best_model_at_end=True,
        metric_for_best_model="eval_loss",
        fp16=True,
        max_grad_norm=0.3,
        optim="adamw_8bit",
        report_to="wandb",
        run_name="ft-function-calling",
    )
    
    # ─── Trainer ───
    trainer = SFTTrainer(
        model=model,
        args=training_args,
        train_dataset=split_dataset['train'],
        eval_dataset=split_dataset['test'],
        tokenizer=tokenizer,
        # CRITICAL: Only compute loss on the assistant's response, not the prompt
        # This prevents the model from learning "predict the user query"
        response_template="<|assistant|>",
        max_seq_length=2048,
        dataset_text_field="text",
    )
    
    # ─── Train ───
    trainer.train()
    
    # ─── Save ───
    trainer.save_model(f"{output_dir}/final")
    tokenizer.save_pretrained(f"{output_dir}/final")
    
    return trainer


# ──────────────────────────────────────────────
# INFERENCE
# ──────────────────────────────────────────────

@dataclass
class FunctionCallResult:
    """Result of a function-calling inference."""
    function_name: Optional[str]
    arguments: Dict
    confidence: float
    raw_output: str
    is_valid: bool
    error: Optional[str] = None


class FunctionCallingInference:
    """
    Inference-time handler for function-calling models.
    
    Features:
    - Parses structured function calls from model output
    - Validates function names against allowed set
    - Falls back to re-prompting on invalid output
    - Keeps model within safe operation boundaries
    """
    
    def __init__(
        self,
        model,
        tokenizer,
        functions: List[Dict],
        max_retries: int = 2,
    ):
        self.model = model
        self.tokenizer = tokenizer
        self.functions = functions
        self.max_retries = max_retries
        self.valid_function_names = {f["name"] for f in functions}
    
    def _build_system_prompt(self) -> str:
        """Build system prompt with function definitions."""
        return FunctionCallingDataGenerator.create_function_list_prompt(self, self.functions)
    
    def _parse_function_call(self, text: str) -> FunctionCallResult:
        """
        Parse model output to extract function call.
        
        Expected format: FUNCTION_CALL: function_name(arg1="val1", arg2="val2")
        Or: NO_FUNCTION_NEEDED: reason
        """
        text = text.strip()
        
        # Check for rejection
        if text.startswith("NO_FUNCTION_NEEDED"):
            return FunctionCallResult(
                function_name=None,
                arguments={},
                confidence=0.9,
                raw_output=text,
                is_valid=True,
            )
        
        # Check for function call
        if not text.startswith("FUNCTION_CALL:"):
            return FunctionCallResult(
                function_name=None,
                arguments={},
                confidence=0.0,
                raw_output=text,
                is_valid=False,
                error="Output doesn't start with FUNCTION_CALL: or NO_FUNCTION_NEEDED",
            )
        
        # Extract function name and arguments
        call_part = text[len("FUNCTION_CALL:"):].strip()
        
        # Parse "function_name(arg1="val1", arg2="val2")"
        # SIMPLIFIED parser — in production, use a proper parser or JSON format
        
        try:
            if "(" not in call_part or not call_part.endswith(")"):
                raise ValueError("Missing function call parentheses")
            
            func_name = call_part.split("(")[0].strip()
            args_str = call_part[call_part.index("(") + 1:-1]
            
            # Validate function name
            if func_name not in self.valid_function_names:
                return FunctionCallResult(
                    function_name=func_name,
                    arguments={},
                    confidence=0.0,
                    raw_output=text,
                    is_valid=False,
                    error=f"Unknown function: {func_name}. Must be one of: {self.valid_function_names}",
                )
            
            # Parse arguments (simplified)
            args = {}
            if args_str.strip():
                for arg in args_str.split(","):
                    if "=" not in arg:
                        continue
                    key, value = arg.split("=", 1)
                    key = key.strip()
                    value = value.strip().strip('"').strip("'")
                    args[key] = value
            
            return FunctionCallResult(
                function_name=func_name,
                arguments=args,
                confidence=0.95,
                raw_output=text,
                is_valid=True,
            )
            
        except Exception as e:
            return FunctionCallResult(
                function_name=None,
                arguments={},
                confidence=0.0,
                raw_output=text,
                is_valid=False,
                error=str(e),
            )
    
    def generate(
        self,
        user_query: str,
        temperature: float = 0.1,
    ) -> FunctionCallResult:
        """Generate a function call for a user query."""
        
        messages = [
            {"role": "system", "content": self._build_system_prompt()},
            {"role": "user", "content": user_query},
        ]
        
        # Apply chat template EXACTLY as during training
        prompt = self.tokenizer.apply_chat_template(
            messages, tokenize=False, add_generation_prompt=True
        )
        
        inputs = self.tokenizer(prompt, return_tensors="pt").to(self.model.device)
        
        with torch.no_grad():
            outputs = self.model.generate(
                **inputs,
                max_new_tokens=128,
                temperature=temperature,
                do_sample=True,
                top_p=0.9,
            )
        
        full_output = self.tokenizer.decode(outputs[0], skip_special_tokens=True)
        
        # Extract just the assistant's response
        assistant_prefix = "<|assistant|>\n"
        if assistant_prefix in full_output:
            response = full_output.split(assistant_prefix)[-1].strip()
        else:
            response = full_output[len(prompt):].strip()
        
        result = self._parse_function_call(response)
        
        # Retry if invalid
        if not result.is_valid and self.max_retries > 0:
            # Different temperature might help
            return self.generate(user_query, temperature=0.3)
        
        return result


# ──────────────────────────────────────────────
# EVALUATION
# ──────────────────────────────────────────────

class FunctionCallingEvaluator:
    """
    Specialized evaluator for function-calling models.
    
    Measures:
    - Function selection accuracy (did it pick the right function?)
    - Argument extraction F1 (did it extract correct args?)
    - Hallucination rate (did it call non-existent functions?)
    - Rejection accuracy (did it correctly refuse when no function applies?)
    """
    
    def __init__(self, inference_pipeline: FunctionCallingInference):
        self.pipeline = inference_pipeline
    
    def evaluate(
        self,
        test_examples: List[Dict],
    ) -> Dict:
        """
        Evaluate function-calling performance.
        
        Each test example:
        {
            "query": "What's the weather in Paris?",
            "expected_function": "get_weather",
            "expected_args": {"location": "Paris"},
            "type": "function_call" | "rejection",
        }
        """
        
        results = {
            "total": len(test_examples),
            "function_accuracy": 0,
            "argument_accuracy": 0,
            "hallucinations": 0,
            "rejection_accuracy": 0,
            "function_confusions": {},  # Which functions get confused with which
            "parameter_extraction_errors": {},  # Which params fail most
            "per_example": [],
        }
        
        function_calls = 0
        rejections = 0
        correct_functions = 0
        correct_args = 0
        correct_rejections = 0
        hallucinations = 0
        
        for example in test_examples:
            result = self.pipeline.generate(example["query"])
            
            expected_type = example.get("type", "function_call")
            is_hallucination = (
                result.function_name is not None
                and result.function_name not in self.pipeline.valid_function_names
            )
            
            # Track per-example result
            entry = {
                "query": example["query"],
                "expected_function": example.get("expected_function"),
                "predicted_function": result.function_name,
                "is_valid": result.is_valid,
                "is_hallucination": is_hallucination,
                "error": result.error,
            }
            
            if expected_type == "rejection":
                rejections += 1
                is_correct = result.function_name is None
                entry["correct_rejection"] = is_correct
                if is_correct:
                    correct_rejections += 1
            else:
                function_calls += 1
                
                # Function selection accuracy
                if result.function_name == example["expected_function"]:
                    correct_functions += 1
                    
                    # Argument extraction accuracy (simplified)
                    expected_args = example.get("expected_args", {})
                    args_correct = True
                    for key, value in expected_args.items():
                        if result.arguments.get(key) != value:
                            args_correct = False
                            # Track which params fail
                            if key not in results["parameter_extraction_errors"]:
                                results["parameter_extraction_errors"][key] = 0
                            results["parameter_extraction_errors"][key] += 1
                    
                    entry["args_correct"] = args_correct
                    if args_correct:
                        correct_args += 1
                else:
                    # Track confusion
                    wrong_func = result.function_name or "NO_FUNCTION"
                    correct_func = example["expected_function"]
                    confusion_key = f"{correct_func}→{wrong_func}"
                    results["function_confusions"][confusion_key] = \
                        results["function_confusions"].get(confusion_key, 0) + 1
                
                if is_hallucination:
                    hallucinations += 1
            
            results["per_example"].append(entry)
        
        # Compute rates
        n_func = max(function_calls, 1)
        n_rej = max(rejections, 1)
        
        results["function_accuracy"] = correct_functions / n_func
        results["argument_accuracy"] = correct_args / n_func
        results["rejection_accuracy"] = correct_rejections / n_rej
        results["hallucination_rate"] = hallucinations / n_func
        
        return results


# ──────────────────────────────────────────────
# USAGE
# ──────────────────────────────────────────────

if __name__ == "__main__":
    """
    Complete pipeline: generate data → train → evaluate.
    
    Requirements:
        pip install transformers datasets trl peft bitsandbytes torch
    
    Usage:
        python ft_function_calling.py
    """
    
    # 1. Tokenizer (needed by data generator)
    tokenizer = AutoTokenizer.from_pretrained("mistralai/Mistral-7B-Instruct-v0.2")
    
    # 2. Generate training data
    generator = FunctionCallingDataGenerator(FUNCTIONS, tokenizer)
    train_data = generator.generate_all_data(
        num_single=3000,
        num_multi=1500,
        num_negative=500,
    )
    
    # Save training data
    with open("ft_training_data.json", "w") as f:
        json.dump(train_data, f, indent=2)
    
    print(f"Generated {len(train_data)} training examples")
    print(f"  - Single-function: 3,000")
    print(f"  - Multi-function:  1,500")
    print(f"  - Negative:          500")
    print(f"  - Total:           5,000")
    
    # 3. Train
    # Uncomment to actually train:
    # trainer = train_function_calling_model(train_data)
    
    print("\nTraining data ready at: ft_training_data.json")
    print("\nNext steps:")
    print("  1. Review training data quality")
    print("  2. Run training (uncomment train call)")
    print("  3. Evaluate with FunctionCallingEvaluator")
```

---

### 🔍 Critical Questions

1. **Format choice**: The example uses a custom `FUNCTION_CALL:` format instead of standard JSON like `{"function": "get_weather"}`. Why might a custom format be better? (Hint: Models learn format-specific patterns. If you use a format with DELIMITERS, the model can learn "stop generating at FUNCTION_CALL boundaries" more easily than JSON.)

2. **Loss masking**: The `response_template="<|assistant|>"` tells the trainer to only compute loss on tokens after this marker. But what if the model's chat template doesn't use `<|assistant|>` markers? How would you find the correct response template for your model?

3. **Temperature during inference**: The code uses `temperature=0.1` for inference. But what if the model is unsure — low temperature might cause it to ALWAYS pick the most common function (class imbalance). Should you use higher temperature for function selection? How do you tune this?

---

## ✅ Good Output Examples

### What a Well-Functioning Function-Calling Model Looks Like

```
User: What's the weather like in Tokyo right now?
Model: FUNCTION_CALL: get_weather(location="Tokyo", date="today")
✓ Correct function, correct arguments

User: Actually, can you check Paris too?
Model: FUNCTION_CALL: get_weather(location="Paris", date="today")
✓ Handles follow-up context correctly

User: Schedule the team meeting for next Tuesday at 2pm
Model: FUNCTION_CALL: create_calendar_event(title="Team meeting", start_time="2024-06-18T14:00")
✓ Extracts date from "next Tuesday", formats as ISO

User: Tell me a joke about programming
Model: NO_FUNCTION_NEEDED: This request doesn't require any of my available functions.
✓ Correctly rejects when no function matches
```

### What Failed Function Calling Looks Like

```
User: What's the weather in Paris?
Model: FUNCTION_CALL: get_weather(location="Paris")
✗ Missing required parameter "date"

User: What's the weather in Paris next Tuesday?
Model: FUNCTION_CALL: get_weather(location="Paris", date="next Tuesday")
✗ "next Tuesday" should be resolved to an ISO date

User: Schedule the team meeting
Model: FUNCTION_CALL: create_calendar_event(title="Team meeting")
✗ Missing required parameter "start_time"

User: Tell me a joke
Model: FUNCTION_CALL: get_weather(location="joke", date="today")
✗ Hallucination: trying to fit a non-matching query into a known function

User: What's the weather in London?
Model: Hello! I can help you check the weather. Let me look that up for you!
✗ Generated text instead of function call — format confusion
```

---

## ❌ Antipatterns & Failure Modes

### Antipattern 1: Training on Outputs, Not Input-Output Mappings

**Problem**: Training data has perfect function calls but doesn't show the model how to GET from query to function call. The model memorizes patterns but can't handle novel phrasing.

**Fix**: Include diverse query phrasings for each function. Show multiple ways users ask for the same thing.

### Antipattern 2: All Functions, All the Time

**Problem**: Every training example has all 3 functions in the system prompt. Model never learns to operate with 1 or 2 functions. At inference, giving the model 10 functions causes confusion.

**Fix**: Vary the number of functions in training. Sometimes 1, sometimes 2, sometimes 3. This teaches the model to work with ANY set of functions.

### Antipattern 3: No Follow-Up Handling

**Problem**: Training data is all single-turn: query → function call. In production, users say "What's the weather in Paris?" → function returns data → user says "How about tomorrow?" The model should call again with updated date.

**Fix**: Include multi-turn training data. After a function call, show the model how to handle follow-up queries that refine arguments.

### Antipattern 4: Perfect Arguments in Training

**Problem**: Training data arguments are always perfectly extracted — never extra text, never missing. Model doesn't learn to handle ambiguity.

**Fix**: Include training examples with AMBIGUOUS queries where arguments aren't clearly specified. Teach the model to ask for clarification.

### Common Training Failures

| Failure | Symptom | Fix |
|---------|---------|-----|
| Function hallucination | Model calls nonexistent functions | Add more negative examples |
| Wrong function choice | Model picks wrong function for query | Add multi-function examples with similar functions |
| Argument missing | Model skips required parameters | Train with more varied argument patterns |
| Format violation | Model doesn't generate valid function call | Match training format to inference format exactly |
| Over-rejection | Model never calls functions in ambiguous cases | Balance negative examples (max 15-20%) |
| Follow-up failure | Model can't handle multi-turn refinement | Add multi-turn training examples |

---

## 🧪 Drills & Challenges

### Drill 1: Build a Confusion Matrix for Function Selection (45 min)

**Goal**: Analyze which functions get confused with each other and understand WHY.

**Task**: Extend `FunctionCallingEvaluator` to generate a confusion matrix:

```python
# Example output
confusion_matrix = {
    ("get_weather", "get_weather"): 93,      # Correct
    ("get_weather", "create_calendar_event"): 4,  # Confused
    ("get_weather", "send_email"): 2,             # Confused
    ("get_weather", None): 1,                     # Rejection
}
```

**Then answer:**
1. Which function pairs are most commonly confused? Why might they share triggering patterns?
2. Create training examples SPECIFICALLY for the confused pairs — queries that could plausibly match EITHER function but should clearly match ONE
3. Does confusion increase with more functions? Plot confusion rate vs. number of available functions

---

### Drill 2: Build the Argument Extractor Evaluation (45 min)

**Goal**: Evaluate argument extraction quality independently from function selection.

**Task**: Build an evaluator that ONLY measures argument extraction:

1. For each function, create a set of queries where the function is ALREADY KNOWN to be correct
2. For each query, annotate: which arguments should be extracted, and what their values should be
3. Measure: per-parameter accuracy, per-parameter F1, and overall argument extraction score
4. Identify which parameters are hardest to extract (dates? names? locations?)

**Questions to answer:**
1. Are some parameter types (enum, string, number) harder to extract than others?
2. Does argument accuracy depend on where the parameter value appears in the query (beginning vs. end)?
3. For date parameters, does the model handle relative dates (tomorrow, next week) differently from absolute dates (2024-06-15)?

---

### Drill 3: Design a Robust Function-Calling Format (30 min)

**Goal**: Design a function-calling output format that MINIMIZES hallucination.

**Current format** (from code above):
```
FUNCTION_CALL: function_name(arg1="val1", arg2="val2")
NO_FUNCTION_NEEDED: reason
```

**Task**: Design TWO alternative formats and argue for which is better:

1. **JSON format**: `{"function": "name", "arguments": {"key": "val"}}`
2. **XML format**: `<function_call><name>get_weather</name><arg name="location">Paris</arg></function_call>`
3. **Markdown format**: `**Function:** get_weather\n**Arguments:** location="Paris"`

**Evaluate each format on:**
- **Parseability**: How easy is it to reliably parse?
- **Token efficiency**: How many tokens per function call?
- **Hallucination resistance**: Does the format make it harder to invent functions?
- **Model learnability**: Does the model naturally generate this format?
- **Multi-function support**: Can the model call multiple functions in one response?

**Then build a PARSER for each format and measure:**
- Parse success rate on 100 model outputs
- False positive rate (parsing something that isn't a function call as one)
- False negative rate (missing a valid function call)

---

## 🚦 Gate Check

Before moving to File 07 (LLMOps & Model Registry), verify you can:

1. **Design training data** — For a 5-function system (search_web, read_document, summarize, translate, send_message), design:
   - 3 multi-function training examples (model must choose between 2+ functions)
   - 2 negative examples (no function needed)
   - 1 multi-turn example (follow-up to a function response)
   - 1 ambiguous argument example (model should ask for clarification)

2. **Diagnose a bad model** — Given these evaluation results:
   ```
   Function accuracy: 82%
   Argument accuracy: 67%
   Hallucination rate: 8%
   Rejection accuracy: 45%
   ```
   What are the TOP 3 problems? What training data changes would fix each?

3. **Build an eval for function calling** — Design an evaluation that measures ALL FOUR dimensions (selection, extraction, hallucination, rejection) with minimum 95% confidence

4. **Choose training format** — For a production system with 50+ functions, would you use JSON, custom format, or function-calling API format? Why?

5. **Debug argument extraction** — The model correctly calls `get_weather` but passes `location="Paris France"` instead of `location="Paris"`. The training data always has clean city names. How do you fix this without retraining? (Output post-processing?) How do you fix this WITH retraining? (Data augmentation?)

---

## 📚 Resources

### Function-Calling Research
- **[Gorilla: Large Language Model Connected with Massive APIs](https://arxiv.org/abs/2305.15334)** — Foundational paper on function-calling fine-tuning
- **[Toolformer: Language Models Can Teach Themselves to Use Tools](https://arxiv.org/abs/2302.04761)** — Self-supervised tool use learning
- **[ToolLLM: Facilitating Large Language Models to Master 16000+ Real-world APIs](https://arxiv.org/abs/2307.16789)** — Large-scale function-calling dataset and evaluation

### Production Patterns
- **[OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling)** — Industry standard format
- **[Anthropic Tool Use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)** — Alternative approach with explicit tool schemas
- **[LangChain Tool Integration](https://python.langchain.com/docs/modules/agents/tools/)** — Framework for tool-using agents

### Phase Cross-References
- **Phase 6 (Agents)** → Function calling is the FOUNDATION of agent tool use. Your Phase 6 agent framework uses function-calling to let models interact with tools.
- **Phase 4 (RAG)** → RAG can be implemented as a function call: `search_knowledge_base(query, top_k=5)`. Fine-tuning for RAG query generation is a function-calling problem.
- **Phase 8 (Guardrails)** → Hallucinated function calls are a SAFETY issue. Your Phase 8 input validation should reject function calls to unknown functions.
- **Phase 7 (Evals)** → Use your Phase 7 evaluation platform to evaluate function-calling quality over time and detect regressions.

> **Next up: File 07 — LLMOps & Model Registry.** You've fine-tuned models, evaluated them, and created specialized function-calling variants. Now you need to MANAGE them — versioning, experiment tracking, model routing, and lifecycle management. This is where fine-tuning meets operational reality.
