# 💚 How to train your own model

## 💛 What is it?

- "Training your own model" almost never means building one from scratch. It means taking an **open model** (Qwen, Llama, Mistral) and **fine-tuning** it on your own examples so it does **one job** well: classify, tag, extract.
- Think of it as hiring a smart generalist and training them on your team's one task, instead of paying an expensive consultant (a frontier API like GPT or Gemini) for every single question.
- Good fit: closed-label tasks with a fixed list of answers (post type, sentiment, language, category).
- Bad fit: storing facts (use retrieval instead, facts in weights go stale) and hard reasoning (keep a frontier model for that).

## 💛 Why do we need it?

- **Cost:** an API bills every call. A GPU you already run costs the same whether it answers 1,000 or 1,000,000 requests.
- **Control:** you pin the exact model version. A vendor upgrade can't silently change your labels overnight.
- **Stability:** same input, same answer (temperature 0, fixed seed).
- **Data stays home:** prompts never leave your infrastructure.
- **The catch:** it only pays off at volume and for a task you can measure. The real cost is people time (labeling, evaluation), not GPUs.

## 💛 How does it work?

It's a loop of six steps. Skipping any one of them is how teams ship a model that looks great in a notebook and fails in production.

### 🤍 Request Flow

```text
 Collect --> Label --> Train --> Test (gate) --> Serve (shadow, cutover) --> Watch
    ^                                                                       |
    +---------------------- drift or audit says retrain --------------------+
```

### 🤍 1. Collect: build the dataset

- Pull real inputs from your data store and render the **exact prompt production sends**, byte for byte. The model must see in training what it will see in serving.
- Split into piles: **train** (learn), **validation** (tune settings), **test** (pick the best run), **gold** (the final exam).
- Split **by group** (user, customer, artist), never randomly. If the same author appears in train and test, the model passes by recognizing the author instead of reading the text. That's called leakage.
- Add a **time holdout**: test on newer data than you trained on, like studying past papers and then sitting a new exam.
- Save each dataset as an immutable, numbered version with a manifest (sources, counts, code version).

### 🤍 2. Label: get answers you're allowed to learn from

- Three sources of labels: people, a bigger "teacher" model (called distillation), or labels you already have.
- **Check the license first.** Many API terms forbid training a model on their outputs. An open-weights teacher with a permissive license avoids that.
- Build a **gold set** by hand: about 1,000 to 2,000 examples, two people label each one independently, a third settles disagreements.
- Measure agreement with **Cohen's kappa** (1 is perfect, 0 is chance). Below about 0.6, your labeling guideline is unclear: fix it before going on.
- **Never train on gold.** It's the exam. If the model studies it, the score means nothing.

### 🤍 3. Train: fine-tune with QLoRA

- **Full fine-tune:** updates every weight. Best quality, needs several big GPUs.
- **LoRA:** freezes the model and trains small add-on matrices (adapters). Like sticky notes in a textbook instead of rewriting it.
- **QLoRA:** LoRA on a model compressed to 4-bit. An 8B model fits on one 24 to 48 GB GPU. This is the default starting point.
- Common starting values: rank 16, alpha 32, dropout 0.05, learning rate 2e-4 with a cosine schedule, 2 epochs.
- Run a **learning curve** (for example 5k, 10k and 20k rows) to see whether more data still helps.
- Log every run (MLflow or similar): dataset version, code commit, seed, settings, scores. Without it you can't reproduce the winner.
- Pick the best run on the **test** pile, not on gold. Then export it to **GGUF**, the single-file format Ollama and llama.cpp load.

### 🤍 4. Test: a fair exam with a gate

- Score the candidate next to: the current production model, an untrained (zero-shot) open model, and a dumb baseline such as TF-IDF plus logistic regression. The baseline tells you the floor.
- Test **exactly as you serve**: same model file, same chat template, same runtime version, same request body.
- Score each label field on its own, with **accuracy** plus a **bootstrap** confidence range (re-score 1,000 reshuffled copies of the gold set).
- Use a **non-inferiority gate**: "the new model is no worse than production by more than a small margin", checked at the unlucky (low) end of the range. A cheaper model only needs to match, not beat.
- Also require a high **valid-output rate** (for example 98%): parseable JSON, every field present, only allowed values.
- Production gets scored against the human labels too. It's a model, not the truth.

### 🤍 5. Serve: shadow first, then cut over

- Serve with Ollama (simple, one box) or vLLM (high throughput).
- Keep a **registry** of versions and point a moving alias such as `classifier:production` at the live one. Clients always call the alias, so upgrades and rollbacks never touch client code.
- **Shadow** for a week: the new model answers a slice of real traffic on the side, and those answers are scored and thrown away.
- Load test at real peak concurrency, then cut over once, with a **fallback** to the old model if a call fails.

### 🤍 6. Watch: monitor and retrain

- Daily: invalid-output rate, fallback rate, label **drift** (has the mix of answers shifted since shadow week?), and whether the upstream prompt changed.
- Monthly: a small human-labeled **audit** of live traffic, compared with the gold score.
- When drift or the audit trips, go back to step 1 with fresh data.

## 💛 Example

### 🤍 Example: one training row (JSONL)

```json
{"prompt": "Classify this social post. Answer in JSON.\nPost: New single drops Friday. Pre-save now, link in bio!", "completion": "{\"post_type\": \"Announcement\", \"topic\": \"New Music\", \"tone\": \"Excited\"}"}
```

### 🤍 Example: QLoRA fine-tune with Hugging Face TRL

```python
import torch
from datasets import load_dataset
from peft import LoraConfig
from transformers import AutoModelForCausalLM, BitsAndBytesConfig
from trl import SFTConfig, SFTTrainer

bnb = BitsAndBytesConfig(load_in_4bit=True, bnb_4bit_quant_type="nf4", bnb_4bit_compute_dtype=torch.bfloat16)
model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen3-8B", quantization_config=bnb, device_map="auto")

train = load_dataset("json", data_files="train.jsonl", split="train")  # prompt + completion columns

trainer = SFTTrainer(
    model=model,
    train_dataset=train,
    peft_config=LoraConfig(r=16, lora_alpha=32, lora_dropout=0.05, target_modules="all-linear", task_type="CAUSAL_LM"),
    args=SFTConfig(
        output_dir="out",
        num_train_epochs=2,
        learning_rate=2e-4,
        lr_scheduler_type="cosine",
        warmup_ratio=0.03,
        per_device_train_batch_size=4,
        gradient_accumulation_steps=8,  # effective batch 32
        seed=42,
    ),
)
trainer.train()  # loss covers only the completion, not the prompt
```

### 🤍 Example: the non-inferiority gate

```python
import random
from collections import defaultdict

def passes_gate(rows, margin=0.05, n=1000, seed=42):
    """rows: one dict per gold example for ONE label field: group, gold, cand, prod."""
    by_group = defaultdict(list)
    for r in rows:
        by_group[r["group"]].append(r)
    groups = list(by_group)
    rng = random.Random(seed)
    diffs = []
    for _ in range(n):
        sample = [r for g in rng.choices(groups, k=len(groups)) for r in by_group[g]]
        cand = sum(r["cand"] == r["gold"] for r in sample) / len(sample)
        prod = sum(r["prod"] == r["gold"] for r in sample) / len(sample)
        diffs.append(cand - prod)
    diffs.sort()
    lower = diffs[int(0.025 * n)]  # low end of the 95% range
    return lower >= -margin, round(lower, 3)
```

On 1,500 simulated examples: an equally good model (0.90 vs 0.90) passes with a low end of -0.027; a worse one (0.80 vs 0.90) fails at -0.129.

### 🤍 Example: serve it with Ollama

```bash
# Modelfile
cat > Modelfile <<'EOF'
FROM ./classifier-8b-q5_k_m.gguf
PARAMETER num_ctx 4096
PARAMETER seed 42
EOF

ollama create classifier:v1 -f Modelfile
ollama cp classifier:v1 classifier:production   # move the alias to promote or roll back

# always send temperature 0 for classification
curl -s http://localhost:11434/v1/chat/completions -d '{
  "model": "classifier:production",
  "temperature": 0,
  "messages": [{"role": "user", "content": "Classify this social post. Answer in JSON.\nPost: ..."}]
}'
```

## 💛 Common traps

- **Random split:** same author in train and test, so the score is inflated.
- **Training on outputs you aren't licensed to use:** read the API terms before distilling.
- **Picking the winner on the exam set:** gold stops being a fair exam.
- **Testing a different prompt or template than production:** the score doesn't transfer.
- **Sampling defaults:** some OpenAI-compatible servers use temperature 1.0 when the request doesn't send one, and a Modelfile setting may not apply on that endpoint. Send temperature 0 in every request.
- **Thinking models:** models like Qwen3 emit a thinking block by default, which can break a JSON parser. Turn thinking off in the chat template.
- **Gating on macro-F1 with a small gold set:** rare labels with a handful of examples make it swing wildly. Gate on accuracy, report F1.

## 💛 What it costs (rough)

- One QLoRA run of an 8B model: a few hours on one cloud GPU, roughly $10 to $25.
- A 1,500-example gold set with 5 fields: about 30 to 40 hours of people time.
- The eval set and the labeling guideline are the expensive, durable part. The weights are the cheap part.

## 💛 References

- QLoRA paper: https://arxiv.org/abs/2305.14314
- LoRA paper: https://arxiv.org/abs/2106.09685
- Hugging Face TRL SFTTrainer: https://huggingface.co/docs/trl/sft_trainer
- Hugging Face PEFT: https://huggingface.co/docs/peft
- Ollama Modelfile reference: https://github.com/ollama/ollama/blob/main/docs/modelfile.md
- llama.cpp (GGUF conversion and quantization): https://github.com/ggml-org/llama.cpp
- MLflow tracking: https://mlflow.org/docs/latest/ml/tracking/
- Label Studio: https://labelstud.io/guide/
- Cohen's kappa: https://en.wikipedia.org/wiki/Cohen%27s_kappa
