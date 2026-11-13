# 💚 How to train your own model

## 💛 How to use this guide

- **Who it's for:** an engineer who has to own a model project (decide, review, run it) without an ML background.
- **The goal:** you can read any training or eval report, ask the right questions, catch a misleading number, and make the calls only the owner makes: what "good enough" means, when to promote, when to roll back, what to spend.
- **The shape:** 10 modules, ordered the way a measure-first project runs: understand the model, learn to measure, get labels, serve, then train. Each module has:
	- **Why it matters:** what goes wrong if you skip it
	- **Core ideas:** the concepts in plain words
	- **Decisions you own:** the calls you make as the owner
	- **Do this:** a hands-on exercise, mostly free, mostly on a laptop
	- **Self-check:** questions you should answer without notes
- **Study links** are all in References at the end, grouped by module and tagged:
	- **[Start]** read or watch this first
	- **[Deep]** once the basics click
	- **[Ref]** look up when you need it
- **Pace:** about 5 hours a week for 8 weeks (plan near the end). Skim every module's Core ideas in one sitting first, so you have the map, then go deep one module at a time.

## 💛 The big picture

The whole job is one loop. Each step has a module.

### 🤍 Request Flow

```text
 Collect --> Label --> Train --> Test (gate) --> Serve (shadow, cutover) --> Watch
    ^                                                                       |
    +---------------------- drift or audit says retrain --------------------+

 Module 1-2  vocabulary for everything
 Module 3    Test: how to measure (do this first, before any training)
 Module 4    Collect + Label
 Module 5    Serve (also how you measure an untrained open model)
 Module 6    Train
 Module 7    Track, promote, watch
 Module 8    GPUs and the cluster
 Module 9    Rights and licenses
```

- **Measure first:** before training anything, score today's production model, an untrained open model, and a cheap baseline on the same human-labeled set. Sometimes the untrained open model is already good enough, and you skip training.
- **The expensive, durable part is people time:** the label guideline, the gold set and the eval. Weights are cheap and replaceable.

## 💛 Module 0: Set up your toolkit

### 🤍 Why it matters

- Every exercise runs in Python. Most need no GPU: a Mac runs small models fine.

### 🤍 Core ideas

- **Python and notebooks:** Jupyter locally, or Google Colab in the browser (free, with a T4 GPU for short sessions). Kaggle Notebooks gives about 30 free GPU hours a week.
- **uv:** a fast replacement for pip and venv. One tool for Python environments.
- **Ollama on your laptop:** runs small open models (0.6B to 8B parameters) on the Mac's GPU. Enough for every serving and eval exercise here.
- **A Hugging Face account:** free. It's where open models and datasets live. Paid GPU jobs only if you want them.
- **Math you actually need:** averages and percentages, basic probability ("70% chance"), what a vector is (a list of numbers), what a matrix multiply is (many weighted sums at once), logarithms, and the idea of a slope. No calculus proofs.

### 🤍 Do this

```bash
brew install uv ollama
uv venv && source .venv/bin/activate
uv pip install jupyter pandas scikit-learn transformers datasets mlflow
ollama serve &              # or open the Ollama app
ollama pull qwen3:0.6b
ollama run qwen3:0.6b "Answer in JSON: what color is the sky?"
```

### 🤍 Self-check

- Can you open a notebook, load a CSV with pandas, and call a local model from Python?

## 💛 Module 1: How an LLM works (enough to reason about it)

### 🤍 Why it matters

- Almost every production surprise (JSON broke, answer changed between runs, out of memory, too slow) traces back to one of these ideas.

### 🤍 Core ideas

- **Tokens:** models read chunks of text, not words. "Pre-save" may be 3 tokens. Emoji and non-English text cost more tokens. Cost, speed and context limits are all counted in tokens.
- **Next-token prediction:** an LLM is one function: given the tokens so far, output a score (a **logit**) for every token in its vocabulary (about 150,000 for Qwen3). **Softmax** turns scores into probabilities. Pick one token, append it, repeat. That loop is all generation is.
- **Temperature:** scales the scores before picking. 0 means always take the top token (greedy). 1 means sample from the model's real distribution (varied, "creative"). For classification, always 0.
- **Why temperature 0 can still wobble:** GPUs add numbers in different orders depending on how many requests are batched together, so tiny rounding differences can flip a close call. Expect a small wobble and measure it (score the same set twice).
- **Embeddings:** each token becomes a vector of a few thousand numbers that encodes its meaning in context.
- **Attention:** lets each token look back at earlier tokens and mix in what's relevant ("it" looks back at "the new single"). A **transformer** is 30 to 60 layers of attention plus small neural networks, stacked.
- **Parameters (weights):** the learned numbers. "8B" means 8 billion. Memory just to hold them is parameters times bytes per parameter: at 16-bit (2 bytes) an 8B model needs about 16 GB; at 4-bit, about 5 GB.
- **Context window and KV cache:** per request, the model keeps a cache of every token it has read. Longer prompts and more parallel requests need more GPU memory on top of the weights.
- **Base, instruct and thinking models:**
	- base: continues raw text; not useful for tasks on its own
	- instruct: trained to follow chat instructions
	- thinking: writes a reasoning block before answering; slower, and the extra text can break a JSON parser. Hybrid models like Qwen3 switch thinking on and off.
- **Chat template:** the exact special tokens that wrap each message (for Qwen: `<|im_start|>user ... <|im_end|>`). A model only behaves as trained when it gets the template it was trained with. A wrong template silently costs accuracy.
- **Dense vs Mixture-of-Experts (MoE):** a dense model uses every weight for every token. An MoE model routes each token to a few "expert" sub-networks. Qwen3-30B-A3B has 30B parameters but only about 3B active for each token, so you pay memory for 30B and get speed closer to a 3B model.

### 🤍 Decisions you own

- Which model family and size to try (start small: bigger is slower and costlier to serve).
- Thinking on or off (off for classification).
- The longest input you must support (drives memory).

### 🤍 Do this

- Paste a real post into Tiktokenizer and count its tokens. Try an emoji-heavy post and a non-English one.
- Run the same prompt 5 times at temperature 1 and 5 times at temperature 0 in Ollama. Compare.
- Work out how much memory Qwen3-8B's weights need at 16, 8 and 4 bits (code in Examples).

### 🤍 Self-check

- Why can't turning up temperature fix "the model doesn't know our new label"?
- Why can a 30B MoE model answer faster than a 14B dense one, yet still not fit on a 16 GB GPU?
- What breaks if training used one chat template and serving uses another?

## 💛 Module 2: ML basics for classification

### 🤍 Why it matters

- Fine-tuning an LLM to classify is classic supervised learning in an LLM costume. Train/test discipline and the metrics all come from here.

### 🤍 Core ideas

- **Supervised learning:** examples with known answers (labels). Training nudges the weights to reduce the mistakes (the **loss**) on them.
- **Generalization:** the only score that matters is on data the model never saw.
- **Overfitting:** memorizing the training examples instead of learning the pattern. The sign: training loss keeps falling while validation loss starts rising.
- **The four piles:**
	- train: the model learns from it
	- validation: you tune settings and decide when to stop
	- test: you pick the best run
	- gold (the final exam): the verdict on the chosen candidate; never used for anything else
- **Hyperparameters:** settings you choose, not learned: learning rate, epochs, batch size.
- **Baselines** (always compare against them):
	- majority class: always answer the most common label
	- a cheap classic model: TF-IDF word counts plus logistic regression
	- whatever runs in production today
	- If you can't beat the cheap model by much, the LLM isn't earning its cost.
- **Metrics:**
	- accuracy: share of answers that are right. Simple, but misleading when classes are imbalanced.
	- confusion matrix: a table of true label by predicted label. It shows *which* mistakes happen.
	- precision: when it says X, how often is it really X?
	- recall: of all the real X, how many did it find?
	- F1: one number balancing precision and recall.
	- macro-F1: F1 per class, averaged, so rare classes count as much as common ones. Swings a lot when rare classes have few examples.
- **Class imbalance:** if 70% of posts are "Promotion", answering "Promotion" every time scores 70%. That's the floor, not a result.
- **Data beats model:** for a narrow task, better labels beat a bigger model nearly every time.

### 🤍 Decisions you own

- Which metric decides (the gate) and which ones are only reported.
- Which classes may get a bit worse (rare, low value) and which may not.

### 🤍 Do this

- Run scikit-learn's text classification example, then swap in 500 of your own labeled texts: TF-IDF plus LogisticRegression. Print `classification_report` and the confusion matrix.
- Compute the majority-class accuracy of your data. That's your floor.

### 🤍 Self-check

- Model A scores 92%, model B 90%, but B finds the rare class you care about far more often. Which ships? (Answer: whichever your written gate says; that's why you write the gate first.)
- Training loss keeps falling and validation loss rose after epoch 1. What happened, and what do you change?

## 💛 Module 3: Evaluation, the owner's main job

### 🤍 Why it matters

- The eval decides everything: whether to ship, which run wins, when to roll back. A weak eval lets a worse model ship behind a confident number.
- If you learn one module deeply, make it this one.

### 🤍 Core ideas

- **Gold set:** a held-out set labeled by people, used only to evaluate. Freeze it, version it, never train on it, never tune on it.
- **Sample it like production:** same platforms, languages and time period as live traffic, with a cap per author so one busy account doesn't dominate.
- **Test as you serve:** same model file, same quantization, same chat template, same runtime version, same request settings, through the same endpoint. Otherwise you scored a different model.
- **Compare against:** production, an untrained (zero-shot) open model, and a cheap baseline. Score production against the human labels too: it's a model, not the truth.
- **Every score is an estimate:** on 1,500 examples at 90% accuracy, the 95% range is about plus or minus 1.5 points (formula: 1.96 times the square root of p(1-p)/n). Two models 1 point apart on 500 examples are a tie.
- **Bootstrap:** resample the gold set with replacement 1,000 times, rescore each time, and take the 2.5th and 97.5th percentiles as the range. Works for any metric.
- **Cluster bootstrap:** posts by the same author are alike, so they aren't independent evidence. Resample whole authors (groups) instead of single posts. The range gets wider and more honest.
- **Paired comparison:** both models answer the same items, so compare them item by item. The range of the *difference* is much narrower than comparing two separate ranges. McNemar's test is the classic paired test.
- **Non-inferiority:** when the new model is cheaper, it doesn't need to win, only to not lose by more than a margin you fix in advance (for example 5 points). It passes if the *low end* of the difference's range is above minus the margin. Choose the margin before seeing any results.
- **Power (sizing the gold set):** pick the size so an equally good model passes almost every time. Too small, and good models fail by bad luck. Simulate it before anyone labels.
- **Valid-output rate:** the share of answers that parse and use only allowed labels. A model that's accurate when it answers but breaks the JSON 5% of the time is not shippable.
- **Slices:** score per platform, language and label. An average can hide one slice that got much worse.
- **Error analysis:** read 50 to 100 mistakes by hand and sort them into causes: unclear guideline, wrong gold label, model wrong, genuinely ambiguous. Numbers tell you *that* something is wrong; reading tells you *why*.
- **Gold has errors too:** when the model disagrees with gold, check the gold. Fix it through adjudication and bump its version.
- **LLM-as-judge:** a model grading answers. Useful for open-ended text. For fixed labels you don't need it: compare with gold directly.
- **Harnesses:** promptfoo compares prompts and models on your own test set. lm-evaluation-harness, lighteval and Inspect run standard benchmarks. Public benchmarks say little about *your* task.

### 🤍 Decisions you own

- The gate: metric, margin, interval method, valid-output threshold, slices that must not regress. Written down before the first candidate is scored.
- Gold size, who adjudicates disagreements, and when the gold set gets refreshed.

### 🤍 Do this

- Take 200 labeled examples. Run two Ollama models (for example `qwen3:0.6b` and `qwen3:4b`) at temperature 0. Compute each one's accuracy with a bootstrap range, then the paired difference with the gate code in Examples.
- Read 30 of the bigger model's mistakes and sort them into causes.

### 🤍 Self-check

- Candidate 91.0%, production 91.5% on 1,500 gold posts, difference range -2.4 to +1.4 points. Pass with a 5-point margin? With a 1-point margin? (Pass; fail.)
- Why is picking the winning training run by its gold score a problem?
- Why does resampling by author give a wider range than resampling by post, and which one is honest?

## 💛 Module 4: Data and labeling

### 🤍 Why it matters

- For a narrow task, the data *is* the model. Most "the model is bad" problems are label problems.

### 🤍 Core ideas

- **Task spec first:** the inputs, the label fields (often called heads), the allowed values per field, and a definition with examples for each value. If two people can't agree on a label, neither can a model.
- **Labeling guideline:** about a page per field: a definition per value, 2 to 3 examples each, the edge cases, and "when unsure, pick X". Revise it after each pilot round.
- **Annotators:** two people label each gold item independently, and a third settles disagreements (adjudication).
- **Agreement:** Cohen's kappa (two annotators) or Krippendorff's alpha (more annotators, or missing labels) corrects for agreeing by chance. A rough reading:
	- below 0.4: weak
	- 0.4 to 0.6: moderate
	- 0.6 to 0.8: substantial
	- above 0.8: strong
	- Low agreement means fixing the guideline or merging confusable values, not training harder.
- **Throughput:** time a pilot. At about a minute per item per person, 1,500 items times 2 people is about 50 hours before adjudication. Book people early.
- **Where training labels come from:** people (slow, best), a bigger teacher model (fast, cheap), or labels you already have (free; check quality and rights).
- **Distillation:** a big teacher model labels lots of data, and a small student learns from it. The student usually gets close to the teacher on that task, and rarely beats it. Score the teacher on gold first: its accuracy is roughly your ceiling.
- **Splits that don't leak:** group by author and near-duplicate text, and hold out time (gold newer than training). A random split inflates the score.
- **Dedup:** exact and near-duplicate text (reposts, templates) must never cross splits.
- **Dataset versions:** each dataset is immutable with a manifest: sources, queries, counts per split and class, code version, prompt template hash. You must be able to say exactly what trained a model.
- **Label noise:** confident learning (the cleanlab library) flags likely mislabeled examples for review.
- **Imbalance:** rare classes need enough training examples (collect more or oversample) and enough gold examples to measure; otherwise only report them.
- **Datasheet:** a short doc per dataset: what it is, why it exists, how it was collected, known gaps.

### 🤍 Decisions you own

- The label set and guideline sign-off; who labels and when; the gold size.
- The teacher model, and whether its license allows training on its outputs.

### 🤍 Do this

- Run Label Studio locally with Docker, create a text classification project, and label 50 items. Have a colleague label the same 50, then compute kappa with scikit-learn (code in Examples).
- Write a one-page guideline for one field, discuss the disagreements, and re-label them.

### 🤍 Self-check

- Kappa on "tone" is 0.45. What do you do before training anything?
- Why must every gold item be newer than every training item?
- The teacher scores 85% on gold. Can you expect the student to reach 92%?

## 💛 Module 5: Serving: quantization, GGUF and Ollama

### 🤍 Why it matters

- You measure an untrained open model and serve a trained one on the same runtime. Serving details change accuracy, not just speed.

### 🤍 Core ideas

- **Quantization:** storing weights in fewer bits: 16-bit, then 8-bit, then 4-bit. Smaller and faster, with some quality loss. 8-bit is usually near-lossless, 4 to 5 bits is the common sweet spot, and below 4 bits quality drops fast. Always measure the exact quant you'll serve on gold.
- **GGUF:** one file holding weights, tokenizer and metadata, loaded by llama.cpp and Ollama. Names like Q4_K_M, Q5_K_M and Q8_0 encode the bits and the method.
- **GPU memory budget:** weights, plus KV cache (per token, times context length, times parallel requests), plus runtime overhead. Leave 20 to 30% headroom.
- **Ollama:** llama.cpp wrapped with a model store and an HTTP API.
	- Modelfile: `FROM` (weights), `TEMPLATE` (chat template), `PARAMETER` (`num_ctx`, `seed`, and so on)
	- `ollama create` builds a model, `ollama cp` copies it to another name (an alias), `ollama rm` deletes it
	- two APIs: native `/api/chat` and OpenAI-compatible `/v1/chat/completions`. Defaults differ between them, so send temperature on every request.
	- concurrency settings: `OLLAMA_NUM_PARALLEL`, `OLLAMA_MAX_LOADED_MODELS`, `OLLAMA_KEEP_ALIVE`
	- structured outputs: pass a JSON schema in `format` and the output is forced to match it
	- `think: false` turns thinking off on the native API
- **vLLM:** a high-throughput server (continuous batching, PagedAttention). Better for big batch jobs and many users, heavier to run.
- **Throughput vs latency:** batching raises total throughput but each request waits longer. Measure at your real peak concurrency.
- **Aliases:** clients always call `name:production`. Promotion and rollback just repoint the alias, so client code never changes.

### 🤍 Decisions you own

- The quant level (from gold scores, not intuition), the runtime, max concurrency, context size, and the rollback procedure.

### 🤍 Do this

- Pull `qwen3:4b-q4_K_M` and `qwen3:4b-q8_0`, score both on your 200 items, and measure speed with `ollama run --verbose`.
- Send the same request through `/api/chat` and `/v1/chat/completions`, with and without temperature. Compare.
- Use structured outputs with a JSON schema that lists your allowed labels (code in Examples).

### 🤍 Self-check

- Q4 scores 1.5 points below Q8 on gold but runs 40% faster. How do you decide?
- Why must the eval go through the same endpoint the app calls?
- What happens to memory when you double `OLLAMA_NUM_PARALLEL`?

## 💛 Module 6: Fine-tuning with LoRA and QLoRA

### 🤍 Why it matters

- This is the "train" step. As owner you won't tune every setting, but you must be able to read a training run and know when something's off.

### 🤍 Core ideas

- **Prompting, RAG or fine-tuning:**
	- prompting: try first, cheapest
	- RAG (retrieval): when the model needs facts it doesn't have
	- fine-tuning: when you need a fixed behavior and format, cheaper or more consistent, and you have plenty of labeled examples
- **SFT (supervised fine-tuning):** show prompt and correct-answer pairs and train the model to produce the answer. For classification, the answer is the JSON with the labels.
- **Completion-only loss:** only the answer tokens count toward the loss, not the prompt.
- **Full fine-tune vs LoRA:** a full fine-tune updates every weight (best quality, heavy hardware). LoRA freezes the model and learns two small matrices per layer whose product is the change. The **rank** sets their size. Only about 0.1 to 1% of the parameters train.
- **QLoRA:** the frozen base is loaded in 4-bit (NF4) and the LoRA adapters train in 16-bit. An 8B model trains on one 24 GB GPU.
- **The settings that matter** (start from known-good defaults, change one at a time):
	- learning rate: the most important; LoRA usually works around 1e-4 to 2e-4
	- epochs: 1 to 3; more tends to overfit
	- rank and alpha: 16 and 32 are typical; higher rank rarely helps a narrow task
	- target modules: all linear layers is the robust default
	- effective batch size: per-device batch times gradient accumulation times GPUs
	- warmup and a cosine schedule
	- max sequence length, and packing short examples together
	- seed
- **Memory savers:** gradient checkpointing (recompute instead of store), bf16 mixed precision, 8-bit optimizers.
- **Reading loss curves:**
	- training loss falls smoothly, validation loss falls then flattens: healthy
	- validation loss rises: overfitting; use an earlier checkpoint or fewer epochs
	- loss spikes or becomes NaN: learning rate too high, or bad data
	- loss barely moves: learning rate too low, or the answers aren't reaching the loss (a template bug)
- **Learning curve:** train on 25%, 50% and 100% of the data. If the score is still climbing, more data will help more than tuning settings.
- **Merge and export:** merge the adapter into the base, convert to GGUF, quantize, and evaluate *that* file.
- **Tools:** Hugging Face Transformers with PEFT and TRL (the standard), Unsloth (about 2x faster, less memory, free notebooks), Axolotl (config-file driven).
- **Beyond SFT** (know they exist): preference tuning (DPO) and reinforcement learning shape style and reasoning. Not needed for label classification.
- **Reproducibility:** same data version, code, seed and library versions give close scores, not bit-identical ones, on GPUs. Log all of it.

### 🤍 Decisions you own

- Base model and size; GPU budget per run and per month; how many runs.
- A stop rule, for example: "if three runs miss the gate, fix the data, not the settings".

### 🤍 Do this

- Open a free Unsloth Colab notebook for Qwen3. Swap in 500 to 2,000 of your labeled rows, train one epoch, and watch the loss. Export GGUF, load it in Ollama, and score it on your 200 gold items.
- Repeat with half the data and compare: that's your first learning curve.

### 🤍 Self-check

- Validation loss was lowest after epoch 1 and the run did 3 epochs. Which checkpoint do you evaluate?
- Why evaluate the quantized GGUF rather than the training checkpoint?
- Rank 64 and rank 16 gave the same gold score. What does that tell you?

## 💛 Module 7: MLOps: track, register, promote, watch

### 🤍 Why it matters

- It's the difference between a demo and a system you can be on call for.

### 🤍 Core ideas

- **Experiment tracking (MLflow):** every run logs its settings, metrics, data version, code commit, image digest and files. Compare runs side by side and reproduce the winner.
- **Model registry:** model versions with aliases (production, candidate). It can live in MLflow or as a reviewed file in git, so a promotion is a pull request.
- **Lineage:** model version to run, to dataset version, to sources and their rights. You need it for legal questions and for debugging.
- **CI for models:** every candidate gets evaluated automatically, and the gate result is attached to its promotion PR.
- **Shadow:** the candidate answers real traffic on the side, and its outputs are scored and thrown away. Catches format breaks, latency and surprises before anyone sees them.
- **Cutover, fallback, rollback:** switch once, with a fallback path (retry on the old provider) and a rollback that is one change.
- **Load test:** at peak concurrency and the worst-case batch, before cutover.
- **Monitoring:**
	- health: errors, latency, invalid-output rate, fallback rate
	- data drift: the inputs change (new platform, different language mix, an upstream prompt edit)
	- prediction drift: the label mix shifts compared with a reference week (Jensen-Shannon divergence or PSI)
	- concept drift: the same input should now get a different label (new trends). Only fresh human labels catch it, hence a regular audit.
- **Retraining triggers:** a drift alarm, an audit score drop, a label set change, an upstream prompt change.
- **Model card:** one page per version: intended use, data, eval results, known limits, owner.
- **Technical debt:** ML systems rot through hidden dependencies: upstream data, prompt changes, glue code.

### 🤍 Decisions you own

- The promotion process and approvers; shadow length and pass criteria; the rollback trigger and who pulls it; monitoring thresholds; audit cadence; retention.

### 🤍 Do this

- Start MLflow with a database (`mlflow server --backend-store-uri sqlite:///mlflow.db`), log three of your Module 6 runs with their settings and gold scores (code in Examples), compare them in the UI, register the best one and give it the alias `production`.
- Write a one-page model card for your toy model.

### 🤍 Self-check

- A user reports wrong labels since Tuesday. Which three things do you check first?
- The label mix drifted, but the monthly audit score didn't move. Do you retrain?
- For the live model, can you name the dataset version and code commit that produced it?

## 💛 Module 8: GPUs and the cluster

### 🤍 Why it matters

- Familiar ground for an infra engineer, but GPU workloads break the usual rules: huge images, scarce capacity, expensive idle time, and memory that is all-or-nothing.

### 🤍 Core ideas

- **What limits a GPU job:** memory (does it fit at all), memory bandwidth (how fast inference generates tokens), and compute (training speed). Inference is usually bandwidth-bound; training is compute-bound.
- **Common GPUs by memory:** T4 16 GB (old, cheap), L4 24 GB, A10G 24 GB, RTX 5090 32 GB (consumer, fast), L40S 48 GB, A100 and H100 80 GB. Check memory first.
- **Training memory math:**
	- full fine-tune: about 16 bytes per parameter (weights, gradients, optimizer state), so an 8B model needs well over 100 GB
	- QLoRA: about 0.5 to 1 byte per parameter for the frozen base, plus small adapters and activations, so an 8B model fits in 24 GB
- **Kubernetes GPU scheduling:** the NVIDIA device plugin advertises `nvidia.com/gpu`. Pods request it in their limits. GPU nodes carry a taint so only GPU pods land on them.
- **Node provisioning:** a Karpenter NodePool with GPU instance types, a hard limit on GPUs, scale-to-zero when empty, and a GPU-ready image with drivers.
- **Jobs:** training and eval run as Kubernetes Jobs with `activeDeadlineSeconds` (a cost cap per run), `backoffLimit` and `ttlSecondsAfterFinished`.
- **Spot vs on-demand:** spot is much cheaper but can disappear mid-run. Fine for training only with checkpoints saved to storage and resume support.
- **Images and weights:** CUDA images are often over 10 GB, and model weights 5 to 60 GB. Cache weights in object storage or a volume, and pin images by digest.
- **Observability:** the DCGM exporter sends GPU utilization, memory and temperature to Prometheus. A training job at 30% utilization is usually waiting on data loading.
- **Cost control:** tags, a budget with alerts, hard NodePool limits, and deadlines on every Job.
- **GPU sharing:** time-slicing or MIG lets small pods share one GPU. Not needed when each training job gets its own GPU.

### 🤍 Decisions you own

- Instance types and the max count; spot or on-demand; the monthly budget and alert thresholds; where weights and datasets live; who can launch GPU jobs.

### 🤍 Do this

- Read Transformer Math 101 and work out QLoRA memory for an 8B model at your sequence length.
- Write a Job manifest that requests `nvidia.com/gpu: 1`, tolerates the GPU taint and sets a deadline. Apply it to a test cluster with no GPU nodes and watch it stay Pending.

### 🤍 Self-check

- Why does a 24 GB GPU fit QLoRA on 8B but not a full fine-tune?
- Training runs at 25% GPU utilization. Where do you look?
- What stops a forgotten GPU job from running all weekend?

## 💛 Module 9: Rights, licenses and privacy

### 🤍 Why it matters

- A model inherits the legal status of everything it learned from, and you can't un-train it.

### 🤍 Core ideas

- **Model licenses:** open weights come with a license. Apache 2.0 (Qwen3, for example) is permissive. Others, like the Llama community license, add conditions (naming, user-count thresholds, an acceptable-use policy).
- **Outputs as training data:** many commercial API terms forbid using the service to build competing models, so distilling from their outputs can breach the terms. Use an open-weights teacher with a permissive license, or get a legal ruling.
- **Input data rights:** the content you train on (user posts, third-party data) has its own terms: platform rules, contracts, copyright. Training and evaluation can get different answers; record both.
- **Privacy:** models can memorize and repeat personal data from training. Minimize it, strip identifiers where you can, and keep data in your own accounts.
- **Rights register:** one file listing every source (input data, labels, weights) with its training and eval status, set by legal. The dataset builder refuses any source that isn't cleared.
- **"Open source AI":** open weights aren't the same as open source. The OSI definition also asks for data information and code.

### 🤍 Decisions you own (with legal)

- Which sources are cleared for training and which for evaluation; which teacher is allowed; retention; what goes in the model card.

### 🤍 Self-check

- Could a paid API's labels be allowed for *evaluating* but not for *training*? Who decides?
- What happens to a model version trained on a source that legal later revokes?

## 💛 Owner's checklist

### 🤍 Before labeling starts

- Task spec with the allowed values per field, written and reviewed
- Guideline v1, with pilot agreement per field around 0.6 kappa or better
- Gold size from a power check, a sampling plan, annotators booked
- Every source listed and cleared for its use

### 🤍 Before training

- Baselines measured on gold: production, a zero-shot open model, TF-IDF
- The gate written down: metric, margin, interval method, valid-output threshold, slices
- Dataset version frozen with a manifest; the leakage check passed
- Budget per run and per month; a stop rule

### 🤍 Before promotion

- Candidate scored as served: same GGUF, template, runtime and request
- Gate passed on every field; error analysis read; model card written
- Shadow passed (days, valid rate, latency at peak); load test passed
- Rollback rehearsed once

### 🤍 After cutover

- Monitors and alerts live, each with an owner; fallback rate tracked
- Audit scheduled; retraining triggers written down

## 💛 Red flags when reviewing a model result

- One number, with no baseline, no range, or no named test set
- "95% accuracy" on a set built like the training data (same authors, same period)
- The winning run chosen on the final exam set
- An eval run through a different prompt, template, quantization or endpoint than production
- Only an average, with no per-field or per-slice numbers
- Teacher labels used both as training data and as the truth to score against
- "Loss went down, so it's better"
- No valid-output rate reported
- Results that can't be reproduced from the logged run
- A prompt or guideline tweaked after reading the gold mistakes, then re-scored on the same gold (that's training on the exam by hand)

## 💛 8-week study plan (about 5 hours a week)

- **Week 1:** Module 0 setup. Module 1 videos (3Blue1Brown on transformers and attention, Karpathy's 1-hour intro). Token and temperature exercises.
- **Week 2:** Module 2. Google ML Crash Course (classification and overfitting sections). scikit-learn baseline on your own data.
- **Week 3:** Module 3. Hamel Husain's evals posts and Anthropic's statistical approach. Score two Ollama models with a bootstrap. Error analysis.
- **Week 4:** Module 4. MIT Data-Centric AI lectures on label errors and dataset creation. Label Studio and the kappa exercise. Write a guideline.
- **Week 5:** Module 5. The visual guide to quantization, the Ollama docs. The quant comparison exercise.
- **Week 6:** Module 6. The Hugging Face course SFT chapter, LoRA Without Regret, Raschka's LoRA tips. The Unsloth Colab fine-tune.
- **Week 7:** Module 7. The MLflow exercise, Rules of ML, the hidden technical debt paper. Write a model card.
- **Week 8:** Modules 8 and 9. Transformer Math 101, GPU memory math, licenses. Then run the whole loop end to end on a toy task.
- **After:** Chip Huyen's *AI Engineering* cover to cover, and Karpathy's *Zero to Hero* if you want to know how it works inside.

## 💛 Glossary

- **Adapter:** the small set of LoRA weights trained on top of a frozen model.
- **Alias:** a moving name (like `classifier:production`) that points at one model version.
- **Attention:** the mechanism that lets each token use information from earlier tokens.
- **Baseline:** a simple model you must beat for the complex one to be worth it.
- **Batch size:** how many examples the model sees per training step.
- **bf16:** a 16-bit number format used for training and serving.
- **Bootstrap:** estimating a score's range by rescoring many resamples of the test set.
- **Chat template:** the special-token wrapping a model expects around each message.
- **Checkpoint:** a saved copy of the weights partway through training.
- **Cohen's kappa:** agreement between two annotators, corrected for chance.
- **Completion-only loss:** training counts only the answer tokens, not the prompt.
- **Confusion matrix:** a table of true labels against predicted labels.
- **Context window:** the most tokens a model can read in one request.
- **Data drift:** live inputs changing compared with what the model was evaluated on.
- **Distillation:** training a small model on a bigger model's answers.
- **DPO:** preference tuning from pairs of better and worse answers.
- **Embedding:** a vector of numbers that represents a token or a text.
- **Epoch:** one full pass over the training data.
- **F1:** a single score balancing precision and recall.
- **Fine-tuning:** further training a pretrained model on your own examples.
- **GGUF:** the single-file model format used by llama.cpp and Ollama.
- **Gold set:** human-labeled examples used only for evaluation.
- **Gradient accumulation:** adding up several small batches before one update, to act like a bigger batch.
- **Gradient checkpointing:** recomputing values instead of storing them, to save memory.
- **Head:** one label field of a task (for example "tone").
- **Hyperparameter:** a setting you choose rather than one the model learns.
- **Inference:** running a trained model to get answers.
- **Jensen-Shannon divergence:** a 0-to-1 distance between two label distributions, used for drift.
- **KV cache:** memory a model keeps per request for the tokens it has already read.
- **Leakage:** test information sneaking into training, which inflates scores.
- **Learning rate:** how big a step each training update takes.
- **Logit:** the raw score a model gives each possible next token.
- **LoRA:** training small low-rank matrices instead of all the weights.
- **Loss:** the number training tries to reduce; how wrong the model is on the training data.
- **Macro-F1:** F1 per class, averaged so every class counts equally.
- **MoE (Mixture of Experts):** a model that uses only some of its weights for each token.
- **Model card:** a one-page summary of a model's use, data, results and limits.
- **Non-inferiority:** a test that a new model is not worse than the old one by more than a set margin.
- **Overfitting:** memorizing training data instead of learning the pattern.
- **Parameter:** one learned number in the model.
- **Precision:** of the items labeled X, the share that really are X.
- **QLoRA:** LoRA on a base model loaded in 4-bit.
- **Quantization:** storing weights in fewer bits.
- **RAG:** retrieving documents and adding them to the prompt, so the model can use facts it wasn't trained on.
- **Rank (LoRA):** the size of the LoRA matrices.
- **Recall:** of the items that really are X, the share the model found.
- **Seed:** the starting number for randomness, fixed for reproducibility.
- **SFT:** supervised fine-tuning on prompt and answer pairs.
- **Shadow:** running a candidate on real traffic without using its answers.
- **Softmax:** turns raw scores into probabilities that sum to 1.
- **Teacher and student:** the big model that labels data, and the small model that learns from it.
- **Temperature:** how random token picking is; 0 is always the top token.
- **Tokenizer:** splits text into tokens and maps them to numbers.
- **Top-p:** sampling only from the most likely tokens that add up to probability p.
- **Validation set:** data used to tune settings and decide when to stop training.
- **vLLM:** a high-throughput open-source LLM server.
- **VRAM:** GPU memory.
- **Zero-shot:** using a model on a task with no task-specific training.

## 💛 Example

### 🤍 Example: one training row (JSONL)

```json
{"prompt": "Classify this social post. Answer in JSON.\nPost: New single drops Friday. Pre-save now, link in bio!", "completion": "{\"post_type\": \"Announcement\", \"topic\": \"New Music\", \"tone\": \"Excited\"}"}
```

### 🤍 Example: how much GPU memory

```python
def weights_gb(params_billion, bits):
    return params_billion * bits / 8

def kv_cache_gb(layers, kv_heads, head_dim, context, parallel, bytes_per_value=2):
    return 2 * layers * kv_heads * head_dim * context * parallel * bytes_per_value / 1e9  # 2 = key and value

print(weights_gb(8, 16), weights_gb(8, 8), weights_gb(8, 4.5))  # 16.0 8.0 4.5
print(round(kv_cache_gb(36, 8, 128, 4096, 4), 2))  # Qwen3-8B, 4k context, 4 parallel: 2.42
```

Read the layer and head counts from the model's `config.json` on Hugging Face.

### 🤍 Example: agreement between two annotators

```python
from sklearn.metrics import cohen_kappa_score

a = ["Promo", "Promo", "News", "Fan", "Promo", "News", "Fan", "Promo", "News", "Fan"]
b = ["Promo", "News", "News", "Fan", "Promo", "News", "Promo", "Promo", "News", "Fan"]
print(round(cohen_kappa_score(a, b), 2))  # 0.70: substantial
```

### 🤍 Example: how big a gold set

```python
import math

def n_for_range(p, half_width, z=1.96):
    return math.ceil(z**2 * p * (1 - p) / half_width**2)

print(n_for_range(0.90, 0.02))   # 865 posts for plus or minus 2 points at 90%
print(n_for_range(0.90, 0.015))  # 1537 for plus or minus 1.5 points
```

This sizes one model's score. The real gate compares two models and resamples by author, so simulate the gate itself before labeling.

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
        gradient_checkpointing=True,
        bf16=True,
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

### 🤍 Example: log runs to MLflow

```python
import mlflow

mlflow.set_tracking_uri("http://127.0.0.1:5000")  # after: mlflow server --backend-store-uri sqlite:///mlflow.db
mlflow.set_experiment("toy-classifier")

with mlflow.start_run(run_name="qlora-r16"):
    mlflow.log_params({"base": "Qwen/Qwen3-0.6B", "rank": 16, "lr": 2e-4, "epochs": 1, "dataset": "v3", "seed": 42})
    mlflow.log_metrics({"gold_accuracy": 0.884, "valid_rate": 0.995})
```

### 🤍 Example: serve it with Ollama

```bash
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

### 🤍 Example: force valid labels with structured outputs

```bash
curl -s http://localhost:11434/api/chat -d '{
  "model": "qwen3:4b",
  "stream": false,
  "think": false,
  "options": {"temperature": 0, "seed": 42},
  "format": {
    "type": "object",
    "properties": {"post_type": {"type": "string", "enum": ["Announcement", "Promotion", "Personal", "Other"]}},
    "required": ["post_type"]
  },
  "messages": [{"role": "user", "content": "Classify this post. Post: New single drops Friday, pre-save now!"}]
}'
```

## 💛 Common traps

- **Random split:** the same author in train and test inflates the score.
- **Training on outputs you aren't licensed to use:** read the API terms before distilling.
- **Picking the winner on the exam set:** gold stops being a fair exam.
- **Testing a different prompt or template than production:** the score doesn't transfer.
- **Sampling defaults:** some OpenAI-compatible servers use temperature 1.0 when the request doesn't send one, and a Modelfile setting may not apply on that endpoint. Send temperature 0 in every request.
- **Thinking models:** models like Qwen3 write a thinking block by default, which can break a JSON parser. Turn thinking off.
- **Gating on macro-F1 with a small gold set:** rare labels with a handful of examples make it swing wildly. Gate on accuracy, report F1.
- **Evaluating the training checkpoint instead of the served file:** quantization and the template change results.
- **Tuning settings when the data is the problem:** if agreement is low or the learning curve is still climbing, fix or add data first.
- **No deadline on GPU jobs:** a stuck job burns money all weekend.

## 💛 What it costs (rough)

- **Learning:** almost everything here is free: courses, docs, Colab's free GPU, Kaggle's free weekly GPU hours, a Mac running Ollama. Books cost about $40 to $80 each.
- **Practice GPUs on Hugging Face Jobs (pay per second):** T4 $0.40 an hour, L4 $0.80, A10G $1.00. A toy fine-tune of a 0.6B to 4B model: under $5.
- **One real QLoRA run of an 8B model:** a few hours on one cloud GPU, roughly $10 to $25.
- **A 1,500-example gold set with 5 fields:** about 30 to 50 hours of people time, the biggest cost of all.

## 💛 References

### 🤍 Module 0: toolkit

- [Start] Google Colab, free notebooks with a GPU: https://colab.research.google.com/
- [Ref] Kaggle Notebooks, free weekly GPU hours: https://www.kaggle.com/docs/notebooks
- [Ref] uv, Python environments: https://docs.astral.sh/uv/
- [Ref] Jupyter: https://jupyter.org/
- [Ref] Ollama docs: https://docs.ollama.com/
- [Start] 3Blue1Brown, Neural networks series (chapters 1 to 4: what a network is, gradient descent, backpropagation; about 1 hour, free): https://www.3blue1brown.com/topics/neural-networks
- [Ref] Khan Academy, statistics and probability (free refresher): https://www.khanacademy.org/math/statistics-probability

### 🤍 Module 1: how an LLM works

- [Start] 3Blue1Brown, Transformers, the tech behind LLMs (27 min): https://www.3blue1brown.com/lessons/gpt
- [Start] 3Blue1Brown, Attention in transformers, step by step (26 min): https://www.3blue1brown.com/lessons/attention
- [Start] Andrej Karpathy, Intro to Large Language Models (1 hour talk): https://www.youtube.com/watch?v=zjkBMFhNj_g
- [Start] Transformer Explainer, interactive, runs GPT-2 in your browser: https://poloclub.github.io/transformer-explainer/
- [Start] Jay Alammar, The Illustrated Transformer: https://jalammar.github.io/illustrated-transformer/
- [Start] Tiktokenizer, see text split into tokens: https://tiktokenizer.vercel.app/
- [Start] Hugging Face LLM Course (free, chapters 1 to 3 for the basics): https://huggingface.co/learn/llm-course/chapter1/1
- [Deep] Andrej Karpathy, Deep Dive into LLMs like ChatGPT (3.5 hours, the whole pipeline from data to chat): https://www.youtube.com/watch?v=7xTGNNLPyMI
- [Deep] LLM Visualization, a 3D walk through a transformer: https://bbycroft.net/llm
- [Deep] Jay Alammar, The Illustrated GPT-2: https://jalammar.github.io/illustrated-gpt2/
- [Deep] Andrej Karpathy, Let's build the GPT Tokenizer (2 hours): https://www.youtube.com/watch?v=zduSFxRajkE
- [Deep] Andrej Karpathy, Let's build GPT from scratch, in code (2 hours): https://www.youtube.com/watch?v=kCc8FmEb1nY
- [Deep] Andrej Karpathy, Neural Networks: Zero to Hero (full free course): https://karpathy.ai/zero-to-hero.html
- [Deep] Thinking Machines, Defeating Nondeterminism in LLM Inference (why temperature 0 still varies): https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/
- [Deep] Stanford CS336, Language Modeling from Scratch (graduate course, free materials): https://stanford-cs336.github.io/
- [Deep] Book: Jay Alammar and Maarten Grootendorst, Hands-On Large Language Models (O'Reilly, 2024); free code: https://github.com/HandsOnLLM/Hands-On-Large-Language-Models
- [Deep] Book: Sebastian Raschka, Build a Large Language Model (From Scratch) (Manning, 2024): https://www.manning.com/books/build-a-large-language-model-from-scratch
- [Deep] Its free code and notebooks: https://github.com/rasbt/LLMs-from-scratch
- [Ref] Hugging Face, chat templates: https://huggingface.co/docs/transformers/chat_templating
- [Ref] Qwen3 release post (sizes, MoE, thinking on and off): https://qwenlm.github.io/blog/qwen3/
- [Ref] Qwen docs (running with Ollama, llama.cpp and vLLM; fine-tuning): https://qwen.readthedocs.io/en/latest/
- [Ref] Qwen3-8B model card: https://huggingface.co/Qwen/Qwen3-8B
- [Ref] Qwen3-30B-A3B model card: https://huggingface.co/Qwen/Qwen3-30B-A3B
- [Ref] Attention Is All You Need, the original transformer paper (skim the figures): https://arxiv.org/abs/1706.03762
- [Ref] Artificial Analysis, compare models on quality, speed and price: https://artificialanalysis.ai/

### 🤍 Module 2: ML basics for classification

- [Start] Google Machine Learning Crash Course (free; do logistic regression, classification, datasets and overfitting): https://developers.google.com/machine-learning/crash-course
- [Start] StatQuest, short clear videos on confusion matrices, precision and recall, cross-validation: https://www.youtube.com/@statquest
- [Start] Google, Rules of Machine Learning (read the first 20 rules): https://developers.google.com/machine-learning/guides/rules-of-ml
- [Ref] scikit-learn, text classification example (the TF-IDF baseline): https://scikit-learn.org/stable/auto_examples/text/plot_document_classification_20newsgroups.html
- [Ref] scikit-learn, metrics and scoring: https://scikit-learn.org/stable/modules/model_evaluation.html
- [Ref] scikit-learn, cross-validation, including group-aware splits: https://scikit-learn.org/stable/modules/cross_validation.html
- [Ref] Wikipedia, precision and recall: https://en.wikipedia.org/wiki/Precision_and_recall
- [Ref] Wikipedia, confusion matrix: https://en.wikipedia.org/wiki/Confusion_matrix
- [Deep] fast.ai, Practical Deep Learning for Coders (free, code first): https://course.fast.ai/
- [Deep] Andrew Ng, Machine Learning Specialization (Coursera, subscription): https://www.coursera.org/specializations/machine-learning-introduction
- [Deep] Andrej Karpathy, A Recipe for Training Neural Networks (how to debug training): https://karpathy.github.io/2019/04/25/recipe/
- [Deep] Book: Chip Huyen, Designing Machine Learning Systems (O'Reilly, 2022); notes and resources: https://github.com/chiphuyen/dmls-book

### 🤍 Module 3: evaluation

- [Start] Hamel Husain, Your AI Product Needs Evals: https://hamel.dev/blog/posts/evals/
- [Start] Hamel Husain and Shreya Shankar, AI Evals FAQ: https://hamel.dev/blog/posts/evals-faq/
- [Start] Anthropic, A statistical approach to model evaluations (error bars, paired comparison, power): https://www.anthropic.com/research/statistical-approach-to-model-evals
- [Start] Eugene Yan, Task-Specific LLM Evals that Do and Don't Work (classification metrics for LLMs): https://eugeneyan.com/writing/evals/
- [Deep] Evan Miller, Adding Error Bars to Evals (the paper: clustered errors, paired differences, sample size): https://arxiv.org/abs/2411.00640
- [Deep] Applied LLMs, What We've Learned From A Year of Building with LLMs: https://applied-llms.org/
- [Deep] Book: Chip Huyen, AI Engineering (O'Reilly, 2025), chapters 3 and 4 on evaluation; resources: https://github.com/chiphuyen/aie-book
- [Ref] Claude docs, define success criteria and build evaluations: https://docs.claude.com/en/docs/test-and-evaluate/develop-tests
- [Ref] Wikipedia, bootstrapping: https://en.wikipedia.org/wiki/Bootstrapping_(statistics)
- [Ref] Wikipedia, McNemar's test: https://en.wikipedia.org/wiki/McNemar%27s_test
- [Ref] Wikipedia, equivalence test (the idea behind non-inferiority): https://en.wikipedia.org/wiki/Equivalence_test
- [Ref] promptfoo, compare prompts and models on your own test set: https://www.promptfoo.dev/docs/intro/
- [Ref] EleutherAI lm-evaluation-harness, standard benchmarks: https://github.com/EleutherAI/lm-evaluation-harness
- [Ref] Hugging Face lighteval: https://github.com/huggingface/lighteval
- [Ref] Inspect, an eval framework from the UK AI Security Institute: https://inspect.aisi.org.uk/

### 🤍 Module 4: data and labeling

- [Start] MIT, Introduction to Data-Centric AI (free lectures and labs; start with label errors and dataset creation): https://dcai.csail.mit.edu/
- [Start] Label Studio quick start: https://labelstud.io/guide/quick_start
- [Ref] Label Studio docs: https://labelstud.io/guide/
- [Ref] Label Studio templates, including text classification: https://labelstud.io/templates/
- [Ref] scikit-learn, cohen_kappa_score: https://scikit-learn.org/stable/modules/generated/sklearn.metrics.cohen_kappa_score.html
- [Ref] Wikipedia, Cohen's kappa: https://en.wikipedia.org/wiki/Cohen%27s_kappa
- [Ref] Wikipedia, Krippendorff's alpha: https://en.wikipedia.org/wiki/Krippendorff%27s_alpha
- [Ref] scikit-learn, GroupShuffleSplit (split by author): https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.GroupShuffleSplit.html
- [Ref] Hugging Face Datasets: https://huggingface.co/docs/datasets/index
- [Ref] cleanlab docs, finding label errors: https://docs.cleanlab.ai/
- [Deep] Confident Learning, the paper behind cleanlab: https://arxiv.org/abs/1911.00068
- [Deep] Datasheets for Datasets: https://arxiv.org/abs/1803.09010
- [Deep] Distilling Step-by-Step, small models learning from a big teacher: https://arxiv.org/abs/2305.02301

### 🤍 Module 5: serving

- [Start] Maarten Grootendorst, A Visual Guide to Quantization: https://newsletter.maartengrootendorst.com/p/a-visual-guide-to-quantization
- [Start] Ollama, Modelfile reference: https://docs.ollama.com/modelfile
- [Ref] Ollama API: https://docs.ollama.com/api
- [Ref] Ollama, OpenAI compatibility: https://docs.ollama.com/api/openai-compatibility
- [Ref] Ollama, structured outputs: https://docs.ollama.com/capabilities/structured-outputs
- [Ref] Ollama, thinking on and off: https://docs.ollama.com/capabilities/thinking
- [Ref] Ollama, importing a model (GGUF and adapters): https://docs.ollama.com/import
- [Ref] Ollama FAQ (concurrency, keep-alive, memory): https://docs.ollama.com/faq
- [Ref] llama.cpp: https://github.com/ggml-org/llama.cpp
- [Ref] llama.cpp, quantization types and how to quantize: https://github.com/ggml-org/llama.cpp/blob/master/tools/quantize/README.md
- [Ref] Hugging Face, GGUF: https://huggingface.co/docs/hub/gguf
- [Ref] Hugging Face Transformers, quantization overview: https://huggingface.co/docs/transformers/main/quantization/overview
- [Deep] vLLM docs: https://docs.vllm.ai/en/latest/
- [Deep] PagedAttention, the paper behind vLLM: https://arxiv.org/abs/2309.06180

### 🤍 Module 6: fine-tuning

- [Start] Hugging Face LLM Course, supervised fine-tuning chapter: https://huggingface.co/learn/llm-course/chapter11/1
- [Start] Unsloth, fine-tuning LLMs guide (beginner friendly): https://docs.unsloth.ai/get-started/fine-tuning-llms-guide
- [Start] Unsloth, free Colab notebooks (Qwen3 and others): https://docs.unsloth.ai/get-started/unsloth-notebooks
- [Start] Thinking Machines, LoRA Without Regret (when LoRA matches a full fine-tune, which settings matter): https://thinkingmachines.ai/blog/lora/
- [Start] Sebastian Raschka, Practical Tips for Finetuning LLMs Using LoRA: https://magazine.sebastianraschka.com/p/practical-tips-for-finetuning-llms
- [Ref] TRL, SFTTrainer: https://huggingface.co/docs/trl/sft_trainer
- [Ref] PEFT docs: https://huggingface.co/docs/peft
- [Ref] PEFT, LoRA explained: https://huggingface.co/docs/peft/conceptual_guides/lora
- [Ref] Hugging Face blog, 4-bit loading and QLoRA with bitsandbytes: https://huggingface.co/blog/4bit-transformers-bitsandbytes
- [Ref] Hugging Face Transformers, training efficiently on one GPU: https://huggingface.co/docs/transformers/perf_train_gpu_one
- [Ref] Unsloth docs: https://docs.unsloth.ai/
- [Ref] Hugging Face Jobs, pay-per-second cloud GPUs: https://huggingface.co/docs/huggingface_hub/guides/jobs
- [Ref] Hugging Face Jobs pricing: https://huggingface.co/docs/hub/jobs-pricing
- [Ref] Hugging Face skills for coding agents (fine-tune and evaluate from Claude Code): https://github.com/huggingface/skills
- [Deep] LoRA paper: https://arxiv.org/abs/2106.09685
- [Deep] QLoRA paper: https://arxiv.org/abs/2305.14314
- [Deep] Google, Deep Learning Tuning Playbook (how to tune settings systematically): https://github.com/google-research/tuning_playbook
- [Deep] Hugging Face smol-course, hands-on with small models: https://github.com/huggingface/smol-course
- [Deep] DeepLearning.AI, Finetuning Large Language Models (short course): https://www.deeplearning.ai/short-courses/finetuning-large-language-models/

### 🤍 Module 7: MLOps

- [Start] MLflow, experiment tracking: https://mlflow.org/docs/latest/ml/tracking/
- [Start] MLflow, model registry and aliases: https://mlflow.org/docs/latest/ml/model-registry/
- [Start] Google Cloud, MLOps: continuous delivery and automation pipelines in machine learning: https://cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning
- [Start] Evidently, what is data drift: https://www.evidentlyai.com/ml-in-production/data-drift
- [Start] Evidently, what is concept drift: https://www.evidentlyai.com/ml-in-production/concept-drift
- [Ref] MLflow docs: https://mlflow.org/docs/latest/
- [Ref] Evidently docs: https://docs.evidentlyai.com/
- [Ref] Wikipedia, Jensen-Shannon divergence: https://en.wikipedia.org/wiki/Jensen%E2%80%93Shannon_divergence
- [Ref] Eugene Yan, applied-ml (how companies run ML in production): https://github.com/eugeneyan/applied-ml
- [Deep] Sculley et al., Hidden Technical Debt in Machine Learning Systems (NeurIPS 2015): https://papers.nips.cc/paper/2015/hash/86df7dcfd896fcaf2674f757a2463eba-Abstract.html
- [Deep] Shankar et al., Operationalizing Machine Learning: An Interview Study: https://arxiv.org/abs/2209.09125
- [Deep] Model Cards for Model Reporting: https://arxiv.org/abs/1810.03993
- [Deep] Made With ML, a free MLOps course: https://madewithml.com/
- [Deep] Full Stack Deep Learning 2022, lectures on deploying and monitoring: https://fullstackdeeplearning.com/course/2022/

### 🤍 Module 8: GPUs and the cluster

- [Start] EleutherAI, Transformer Math 101 (memory and compute math): https://blog.eleuther.ai/transformer-math/
- [Start] Horace He, Making Deep Learning Go Brrrr From First Principles (compute, bandwidth, overhead): https://horace.io/brrr_intro.html
- [Deep] Stas Bekman, Machine Learning Engineering open book (written for infra engineers: hardware, storage, debugging): https://github.com/stas00/ml-engineering
- [Deep] Hugging Face, The Ultra-Scale Playbook (multi-GPU training, for later): https://huggingface.co/spaces/nanotron/ultrascale-playbook
- [Ref] Tim Dettmers, which GPU for deep learning: https://timdettmers.com/2023/01/30/which-gpu-for-deep-learning/
- [Ref] Kubernetes, schedule GPUs: https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/
- [Ref] Kubernetes, Jobs: https://kubernetes.io/docs/concepts/workloads/controllers/job/
- [Ref] NVIDIA device plugin for Kubernetes: https://github.com/NVIDIA/k8s-device-plugin
- [Ref] NVIDIA DCGM exporter, GPU metrics for Prometheus: https://github.com/NVIDIA/dcgm-exporter
- [Ref] Karpenter, NodePools: https://karpenter.sh/docs/concepts/nodepools/
- [Ref] Karpenter, disruption (scale-down): https://karpenter.sh/docs/concepts/disruption/
- [Ref] Amazon EKS, accelerated AMIs for GPU nodes: https://docs.aws.amazon.com/eks/latest/userguide/ml-eks-optimized-ami.html
- [Ref] Amazon EC2 G6 (L4 GPUs): https://aws.amazon.com/ec2/instance-types/g6/
- [Ref] Amazon EC2 G6e (L40S GPUs): https://aws.amazon.com/ec2/instance-types/g6e/

### 🤍 Module 9: rights and licenses

- [Ref] Apache License 2.0: https://www.apache.org/licenses/LICENSE-2.0
- [Ref] Llama 3.1 Community License, an example of a custom model license: https://www.llama.com/llama3_1/license/
- [Ref] Gemini API additional terms, an example of limits on building competing models: https://aistudio.google.com/docs/terms
- [Ref] Open Source Initiative, the Open Source AI Definition: https://opensource.org/ai/open-source-ai-definition
