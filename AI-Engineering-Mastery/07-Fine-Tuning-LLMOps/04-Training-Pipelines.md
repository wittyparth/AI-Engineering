# Training Pipelines

## The Training Stack

```
Data → Tokenization → Model → Forward Pass → Loss → Backward Pass → Optimizer Step
                                                                         ↓
                                                                   Log to WandB
```

## Training Loop (Conceptual)

```python
for epoch in range(num_epochs):
    for batch in dataloader:
        # 1. Forward pass
        outputs = model(batch["input_ids"], labels=batch["labels"])
        loss = outputs.loss

        # 2. Backward pass
        loss.backward()

        # 3. Gradient clipping
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_grad_norm=1.0)

        # 4. Optimizer step
        optimizer.step()
        scheduler.step()
        optimizer.zero_grad()

        # 5. Logging
        wandb.log({"loss": loss.item(), "lr": scheduler.get_last_lr()[0]})
```

## Multi-GPU Training

```python
# DeepSpeed config
deepspeed_config = {
    "train_batch_size": 64,
    "gradient_accumulation_steps": 8,
    "fp16": {"enabled": True},
    "zero_optimization": {
        "stage": 3,  # ZeRO-3: shard optimizer states, gradients, and parameters
        "offload_optimizer": {"device": "cpu"},
    },
}

training_args = TrainingArguments(
    deepspeed=deepspeed_config,
    per_device_train_batch_size=2,  # Small per GPU, gradient accumulation makes up
    gradient_checkpointing=True,  # Trade compute for memory
)
```

## Training Monitoring Dashboard

Track these metrics during training:

```
Loss (should decrease)
Perplexity (should decrease)
Gradient norm (should be stable)
Learning rate (should follow schedule)
GPU memory (should be stable)
Tokens per second (throughput)
```

## 🔴 Senior: Common Training Failures

| Symptom | Cause | Fix |
|---------|-------|-----|
| Loss spikes | Learning rate too high | Lower LR or warmup |
| Loss plateaus | Model not learning | Check data quality, increase LR |
| OOM | Batch size too large | Reduce batch size, enable gradient checkpointing |
| Slow training | Data loading bottleneck | Use num_workers, prefetch |
| Overfitting | Too many epochs | Early stopping, more data, lower rank |
| Underfitting | Not enough capacity | Higher rank, more epochs, better data |
