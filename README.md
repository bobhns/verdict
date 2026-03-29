# Verdict

**Local and cloud model benchmarking that answers: which model should I use?**

```
verdict run

Models: qwen2.5:7b, llama3.2:3b, sonnet
Judge:  haiku
Cases:  25 across 3 pack(s)

Model                Score  Acc   Comp  Conc  Latency    Cost       Win%
──────────────────────────────────────────────────────────────────────────
[1] qwen2.5:7b       8.7    8.5   8.9   8.2   1.2s       free       68%
[2] sonnet            8.4    9.1   8.4   8.1   3.5s       $0.0234    58%
[3] llama3.2:3b       7.1    7.2   7.1   7.0   0.8s       free       35%

Cost-quality frontier
qwen2.5:7b matches sonnet within 0.3pts. Use the free model.

Winner: qwen2.5:7b (8.7/10, 18 wins)
```

---

## Why Verdict?

### The Problem

You're choosing between models based on:
- Generic benchmarks (MMLU, HellaSwag) that don't match your work
- Vibes and anecdotes ("Model X feels better")
- Expensive trial and error in production

### The Solution

**Verdict runs YOUR tasks against ANY models and tells you which wins.**

- Test local vs cloud on tasks that matter to you
- Catch quality regressions (2-bit quantization broke JSON? You'll know)
- Make data-driven decisions (not vibes)
- One config file, runs anywhere, zero cloud dependency

---

## Quick Start

### Install

```bash
npm install -g verdict
# or run directly
npx verdict init
```

### Initialize

```bash
verdict init
```

Creates:
- `verdict.yaml` - Your config (models, judge, settings)
- `eval-packs/` - Test cases for your domain

### Discover Models

```bash
verdict models discover
```

Finds installed Ollama/MLX models, gives you YAML to paste into config.

### Run Evals

```bash
verdict run

# Run specific pack
verdict run --pack code-generation

# Test specific models
verdict run --models "qwen2.5:7b,sonnet"

# Dry run (preview without API calls)
verdict run --dry-run
```

---

## What You Get

### 1. Leaderboard

Ranked models with scores across accuracy, completeness, conciseness, latency, cost, and win rate:

```
Model                Score  Acc   Comp  Conc  Latency    Cost       Win%
──────────────────────────────────────────────────────────────────────────
[1] qwen2.5:7b       8.7    8.5   8.9   8.2   1.2s       free       68%
[2] sonnet            8.4    9.1   8.4   8.1   3.5s       $0.0234    58%
[3] llama3.2:3b       7.1    7.2   7.1   7.0   0.8s       free       35%
```

### 2. Cost-Quality Frontier

Is paying for cloud worth it?

```
Cost-quality frontier
qwen2.5:7b matches sonnet within 0.3pts. Use the free model.
```

### 3. Per-Case Detail

See every prompt and per-model score with judge reasoning:

```
[case-001] "Write a function to parse CSV..."
    qwen2.5:7b             ||||||||||  9.2  Handles edge cases, clean code
    llama3.2:3b            |||||||...  7.0  Works but missing error handling
    sonnet                 |||||||||.  9.1  Correct with good structure
```

### 4. Baseline Regression Detection

Quantized a model? See what broke:

```
Baseline comparison (vs "production-v1")
────────────────────────────────────────────────────────
↑ qwen2.5:7b               8.5 → 8.7  +0.20 (+2.4%)
↓ qwen2.5:2bit             7.2 → 4.1  -3.10 (-43.1%)  REGRESSION

REGRESSION ALERT: One or more models dropped > 0.5pts vs baseline
```

---

## Supported Models

### Local Inference

| Provider | Status | Auto-Discovery | Notes |
|----------|--------|----------------|-------|
| **Ollama** | Full | Yes | Any model, any host, MoE detection |
| **MLX** | Full | Yes | Apple Silicon optimized |
| **LM Studio** | Compatible | Coming | Works via localhost:1234 |
| **llama.cpp** | Compatible | Coming | Any OpenAI-compat server |

### Cloud Models

| Provider | Status | One-Liner Setup |
|----------|--------|-----------------|
| **OpenRouter** | Supported | One key = 200+ models |
| **OpenAI** | Supported | Direct integration |
| **Anthropic** | Supported | Via OpenRouter or proxy |
| **Groq** | Supported | Direct (ultra-fast) |
| **Mistral** | Supported | Direct |
| **Any OpenAI API** | Supported | `base_url` + `api_key` |

**The judge can be any model** - including a local one. No cloud required!

---

## Example: Comparing Local vs Cloud

### Your Task

You write TypeScript code daily. Should you use:
- qwen2.5:7b (local, free)
- claude-sonnet ($3/million tokens)

### Create Eval Pack

```yaml
# eval-packs/my-coding.yaml
name: Daily TypeScript Work
cases:
  - prompt: "Write a function to debounce API calls"
    judge_criteria: "Code quality, edge cases, TypeScript types"

  - prompt: "Refactor this into async/await"
    context: |
      function getData(callback) {
        fetch('/api').then(r => callback(r))
      }
    judge_criteria: "Clean code, error handling"

  - prompt: "Debug: Why is this useState not updating?"
    context: |
      const [items, setItems] = useState([]);
      items.push(newItem); // Bug here
    judge_criteria: "Finds bug, explains why, fixes it"
```

### Run It

```bash
verdict run --pack my-coding
```

Results include the leaderboard, per-case scores, cost-quality frontier, and a winner declaration — your scores will vary based on your tasks and models.

---

## Example: Quantization Testing

### Scenario

You quantized qwen2.5:7b to 2-bit. Did quality drop?

### Setup

```yaml
# verdict.yaml
models:
  - id: qwen-4bit
    provider: ollama
    model: qwen2.5:7b

  - id: qwen-2bit
    provider: ollama
    model: qwen2.5:2bit
```

### Run

```bash
verdict run --models "qwen-4bit,qwen-2bit"
```

Compare scores in the leaderboard output to see exactly where quality dropped. Use `verdict baseline save` and `verdict baseline compare` to track regressions over time.

---

## Configuration

### verdict.yaml

```yaml
models:
  # Local models (Ollama)
  - id: qwen-fast
    provider: ollama
    model: qwen2.5:7b
    base_url: http://localhost:11434  # optional

  # Cloud models (OpenRouter)
  - id: sonnet
    provider: openrouter
    model: anthropic/claude-sonnet-4
    api_key: ${OPENROUTER_KEY}

  # Cloud models (direct)
  - id: gpt4
    provider: openai
    model: gpt-4o
    api_key: ${OPENAI_KEY}

judge:
  model_id: qwen-fast  # Use local model as judge (free!)
  temperature: 0.3
  max_tokens: 500

settings:
  parallel_requests: 3  # Run 3 evals at once
  timeout_seconds: 30
  retry_on_failure: true
```

### Eval Pack

```yaml
# eval-packs/code-generation.yaml
name: Code Generation
judge_criteria: |
  Rate 1-10 based on:
  - Correctness
  - Code quality
  - Edge case handling
  - Explanation clarity

cases:
  - prompt: "Write a function to deep clone an object"
    expected_behavior: "Handles nested objects, arrays, null"

  - prompt: |
      Fix this bug:
      const data = [1,2,3];
      data.length = 0;
      console.log(data); // Why is this empty?
    judge_criteria: "Explains array.length mutation correctly"
```

---

## Advanced Features

### 1. Model Router

**Automatically choose the best model for each task based on eval history.**

```bash
verdict route "Debug this memory leak in React"
# → routing to qwen2.5:7b (8.7/10)
#   Best match for coding tasks; lowest latency
```

The router learns from eval results — which model wins for which task types — and auto-classifies incoming prompts to pick the best match.

### 2. Baseline Comparison

Track model improvements over time:

```bash
# Save current results as baseline
verdict baseline save v1.0

# Later, compare new run to baseline
verdict run
verdict baseline compare v1.0
```

### 3. Custom Judges

Use different judge models for different packs:

```yaml
# eval-packs/creative-writing.yaml
name: Creative Writing
judge:
  model_id: sonnet  # Use smart judge for creative tasks
  temperature: 0.7
```

### 4. Inference Optimization

```yaml
# verdict.yaml
models:
  - id: qwen-fast
    provider: ollama
    model: qwen2.5:7b
    parameters:
      num_gpu: 99  # Use all GPU layers
      num_ctx: 8192  # Larger context
      temperature: 0.1  # More deterministic
```

### 5. JSON Output for CI/CD

```bash
verdict run --json 2>/dev/null > results.json
```

All informational output goes to stderr; stdout gets structured JSON with full results, summaries, and synthesis.

### 6. OpenAI-Compatible Proxy

```bash
verdict serve --port 4000
```

Starts an HTTP proxy that routes requests to the best model using `model: "auto"` or task type hints like `model: "auto:reasoning"`.

---

## CLI Reference

```bash
# Setup
verdict init                        # Create verdict.yaml + eval-packs/
verdict validate                    # Check config for errors without running evals
verdict models                      # Ping all configured models
verdict models discover             # Find Ollama/MLX models

# Run evals
verdict run                         # Run all packs, all models
verdict run -p code-gen             # Run specific pack
verdict run -m "qwen,sonnet"        # Test specific models
verdict run --dry-run               # Preview (no API calls)
verdict run --resume                # Resume from checkpoint
verdict run --question "Which is best for code?"  # Ask synthesis question
verdict run --json                  # Output JSON to stdout (for CI/CD)
verdict run --category reasoning    # Filter cases by category

# Compare
verdict compare <run-a> <run-b>     # Compare two result JSON files

# Baselines
verdict baseline save v1.0          # Save current as baseline
verdict baseline list               # Show saved baselines
verdict baseline compare v1         # Compare to baseline

# Routing
verdict route "your prompt"         # Route to best model based on history

# Infrastructure
verdict serve --port 4000           # OpenAI-compatible HTTP proxy
verdict history                     # View eval history from database
verdict watch                       # Poll for new local models
verdict daemon start                # Start background job daemon
verdict daemon stop                 # Stop daemon
verdict daemon status               # Show daemon status
```

---

## Real-World Use Cases

### 1. Choosing Local vs Cloud

**Goal:** Stop paying for cloud API calls when a local model is good enough.

**Setup:**
- Add local model (e.g. qwen2.5:7b via Ollama)
- Add cloud model (e.g. sonnet via OpenRouter)
- Create eval pack from your actual daily tasks

**Process:** Run `verdict run`, compare scores and cost in the leaderboard. The cost-quality frontier tells you if the local model is close enough.

### 2. Regression Testing

**Goal:** Catch quality drops before users do.

```bash
# Save current production model as baseline
verdict baseline save production-v1

# After updating model:
verdict run
verdict baseline compare production-v1
```

If any model drops more than 0.5pts, verdict flags a regression alert.

### 3. Cost Optimization

**Goal:** Find cheapest model that meets your quality bar.

Configure multiple models at different price points, run your eval packs, and use the leaderboard to find the cheapest model with an acceptable score.

---

## Contributing

We welcome:
- Bug reports
- Feature ideas
- Eval pack templates
- Provider integrations

**Not currently accepting:**
- Major architecture changes (please discuss first)
- New dependencies without clear value
- Breaking API changes

See `CONTRIBUTING.md` for details.

---

## FAQ

### Do I need a cloud API key?

No! You can use local models (Ollama, MLX) as both test subjects AND judge.

### How is this different from MMLU/HellaSwag?

Those are generic academic benchmarks. Verdict tests YOUR tasks.

### Can I test proprietary models?

Yes! Any OpenAI-compatible API works (OpenRouter, Azure, custom endpoints).

### Does the judge need to be GPT-4?

No! Local models (qwen2.5:7b, mistral) make excellent judges.

### How long does a benchmark run take?

Depends on:
- Number of cases (10 = ~2 min, 100 = ~20 min)
- Model speed (local = fast, cloud = medium)
- Parallel requests (3 = default)

### Can I use this in CI/CD?

Yes! Use `--json` for machine-readable output:

```bash
verdict run --json 2>/dev/null | jq '.summary'
```

Combine with `verdict baseline compare` to detect regressions.

---

## License

MIT

---

## Links

- **GitHub:** https://github.com/hnshah/verdict
- **Issues:** https://github.com/hnshah/verdict/issues

---

## Roadmap

**Shipped:**
- Model router (auto-select best model per task)
- Baseline comparison and regression detection
- JSON output for CI/CD
- OpenAI-compatible proxy server

**Coming Soon:**
- LM Studio auto-discovery
- Multi-judge consensus (2+ judges vote)
- Tag-based filtering (`verdict run --tags "quick,sanity"`)
- Web UI dashboard
- CI/CD GitHub Action

**Considering:**
- Prompt optimization (auto-improve prompts via evals)
- Dataset generation (create eval packs from logs)
- Model fine-tuning integration (eval → retrain loop)

Vote on features: https://github.com/hnshah/verdict/discussions

---

**Built by developers tired of guessing which model to use.**

*Stop vibes-based model selection. Start making data-driven decisions.*

---

Developed with cloud and local AI assistance.
