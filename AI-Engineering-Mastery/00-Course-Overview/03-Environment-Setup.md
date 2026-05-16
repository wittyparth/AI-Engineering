# Environment Setup

## 1. Python Environment

```bash
pyenv install 3.12
pyenv local 3.12
python -m venv .venv
source .venv/bin/activate  # or .venv\Scripts\activate on Windows
```

## 2. API Keys

```bash
export OPENAI_API_KEY="sk-..."
export ANTHROPIC_API_KEY="sk-ant-..."
export LANGFUSE_PUBLIC_KEY="pk-..."
export LANGFUSE_SECRET_KEY="sk-..."
export JINA_API_KEY="jina_..."  # for embeddings
```

Store in `.env` files per project (never commit).

## 3. Docker

```bash
docker --version  # should be 24+
docker compose version  # should be 2.20+
```

## 4. AWS

```bash
aws configure --profile ai-course
# Set region to us-east-1
aws sts get-caller-identity  # verify
```

## 5. Local LLM (optional but recommended)

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.2:3b  # for development
ollama pull nomic-embed-text  # for embeddings
```

## 6. Verify Setup

```bash
python -c "import openai; print('OpenAI OK')"
python -c "import anthropic; print('Anthropic OK')"
python -c "import fastapi; print('FastAPI OK')"
python -c "import docker; print('Docker SDK OK')"
docker run hello-world
```

## 7. Project Template

Each phase project follows this structure:

```
project/
├── src/
│   ├── __init__.py
│   ├── main.py          # FastAPI app
│   ├── config.py        # Settings via pydantic-settings
│   ├── models/          # Pydantic models
│   ├── services/        # Business logic
│   └── utils/           # Helpers
├── tests/
│   ├── test_main.py
│   └── test_services.py
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
├── pyproject.toml
└── .env.example
```
