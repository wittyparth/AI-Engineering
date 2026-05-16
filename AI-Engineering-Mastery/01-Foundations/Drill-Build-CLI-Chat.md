# Drill: Build a CLI Chat Client

**Time**: 30 min  **Difficulty**: Easy

## Task

Build a command-line chat client that:
1. Connects to OpenAI (or your configured provider)
2. Streams responses token-by-token to stdout
3. Shows token count and cost per message
4. Supports multi-turn conversation (maintains message history)
5. Can be cancelled with Ctrl+C

## Constraints
- Use only the Python standard library + `openai` package
- Must handle Ctrl+C gracefully (clean exit, no traceback)
- Must display cost in real-time

## Starter

```python
import os
import sys
import json
import signal
from openai import OpenAI

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

def chat():
    messages = [{"role": "system", "content": "You are a helpful assistant."}]
    print("AI Chat (Ctrl+C to exit)\n" + "=" * 30)

    while True:
        try:
            user_input = input("\nYou: ")
            messages.append({"role": "user", "content": user_input})
            print("\nAI: ", end="", flush=True)

            response = client.chat.completions.create(
                model="gpt-4o-mini",
                messages=messages,
                stream=True,
            )

            full_response = ""
            for chunk in response:
                if chunk.choices[0].delta.content:
                    content = chunk.choices[0].delta.content
                    print(content, end="", flush=True)
                    full_response += content

            messages.append({"role": "assistant", "content": full_response})
            print()
        except KeyboardInterrupt:
            print("\n\nGoodbye!")
            break
        except Exception as e:
            print(f"\nError: {e}")

if __name__ == "__main__":
    chat()
```

## Extension
Add token counting with `tiktoken` and cost display per response.
