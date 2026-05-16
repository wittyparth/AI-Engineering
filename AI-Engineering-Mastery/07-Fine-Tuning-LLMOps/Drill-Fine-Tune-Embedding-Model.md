# Drill: Fine-Tune Embedding Model

**Time**: 45 min  **Difficulty**: Medium

## Task

Fine-tune a sentence-transformer embedding model for domain-specific retrieval.

## Setup

```python
from sentence_transformers import (
    SentenceTransformer,
    losses,
    InputExample,
)
from torch.utils.data import DataLoader

# Base model
model = SentenceTransformer("all-MiniLM-L6-v2")

# Training data: (anchor, positive, negative) triplets
train_examples = [
    InputExample(
        texts=[
            "How do I deploy FastAPI to AWS?",  # anchor (query)
            "Deploying FastAPI on AWS ECS with Docker",  # positive (relevant doc)
            "Introduction to Python lists",  # negative (irrelevant doc)
        ],
        label=1.0,  # Not used for triplet loss
    ),
    # Add 50-100 domain-specific triplets
]

train_dataloader = DataLoader(train_examples, shuffle=True, batch_size=16)
train_loss = losses.TripletLoss(model=model)

# Train
model.fit(
    train_objectives=[(train_dataloader, train_loss)],
    epochs=5,
    warmup_steps=100,
    output_path="./finetuned-embedder",
)

# Save
model.save("./finetuned-embedder")
```

## Evaluation

```python
def evaluate_retrieval(model, test_queries, test_docs, relevant_pairs):
    # Before fine-tuning
    original_model = SentenceTransformer("all-MiniLM-L6-v2")
    orig_recall = compute_recall(original_model, test_queries, test_docs, relevant_pairs)

    # After fine-tuning
    ft_recall = compute_recall(model, test_queries, test_docs, relevant_pairs)

    return {
        "baseline_recall@10": orig_recall,
        "finetuned_recall@10": ft_recall,
        "improvement": ft_recall - orig_recall,
    }
```

## Challenge

Use your Phase 3 (Embeddings) or Phase 4 (RAG) project data. Fine-tune the embedding model on your domain and measure retrieval improvement.
