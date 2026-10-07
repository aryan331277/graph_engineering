# Adaptive Graph Harness (AGH)

still a work in progress.....integrating with current systems tooo

**An execution optimizer for agentic workloads — not another agent framework.**

Central question: **how much computation can we avoid while still producing the same-quality result?**

```
task → planner → DAG → optimizer (dag + routing + pruning) → scheduler
     → fresh-context verify → deterministic reduce → synthesize → receipt
```

Every run emits a machine-readable **execution receipt** (`runs/*.json`) stating exactly
what computation happened and what was avoided.

## The three optimizers (the thesis in one picture)

```
Restructure the computation → allocate the right computation → avoid computation.
```

1. **DAG optimization** (`harness/optimization/dag.py`) — fake-edge detection via
   node contracts (`requires ∩ provides`), transitive reduction, critical-path
   analysis, DAG validity checks. Naive chains become parallel graphs.
2. **Cost-aware model routing** (`routing.py`, `cost_model.py`, `registry.py`) —
   per node, minimize `α·cost + β·latency + γ·quality_loss` s.t. quality ≥ threshold.
   Model registry is configurable (no hardcoded tiers); mock tiers map to Groq
   `openai/gpt-oss-20b` (cheap) / `openai/gpt-oss-120b` (strong).
3. **Expected-value skip/pruning** (`pruning.py`) — every optional node scores
   `EV = expected quality gain / expected execution cost` (difficulty, user-intent
   relevance, verification weight, budget pressure). Below threshold → SKIP with
   recorded reason. Adaptive runtime (`harness/execution/adaptive.py` + scheduler
   wave hook) re-applies this at runtime: wave-0 SKIP, per-wave TERMINATE/EXPAND,
   REROUTE-to-strong on timeout, EXPAND = real second evidence pass keeping best.

Plus: fresh-context verifier, deterministic (code, not LLM) reducer with
Jaccard-0.85 semantic dedup, model-keyed file cache, token/cost budget gates from
`configs/budgets.yaml`, retries/timeouts/failure propagation, per-node traces.

## Results (measured, live on Groq unless noted)

Task A: `Compare vLLM, SGLang and TensorRT-LLM for LLM inference`
(`openai/gpt-oss-20b` cheap / `openai/gpt-oss-120b` strong, max_parallel=2):

| strategy   | tokens | cost     | latency | quality* | exec |
|------------|--------|----------|---------|----------|------|
| optimized  | 5998   | $0.00125 | 4.7s    | 0.857    | 7/7  |
| adaptive   | 6243   | $0.00130 | 4.8s    | 0.629    | 7/7  |
| static     | 7717   | $0.00169 | 6.8s    | 0.586    | 7/7  |
| sequential | 7197   | $0.00313 | 12.3s   | 0.657    | 7/7  |
| parallel   | 8063   | $0.00375 | 27.8s†  | 0.657    | 7/7  |

\*content-grounded score (finding coverage + evidence grounding − failure penalties),
or LLM rubric judge when `--judge` fills it. †parallel hit Groq 429s; latency includes backoff.

**Head-to-head deltas, recomputed from receipts (Task A):**

vs sequential (7,197 toks / $0.00313 / 12.3s / q0.657):

| strategy  | tokens | cost   | latency | quality            |
|-----------|--------|--------|---------|--------------------|
| optimized | −16.7% | −59.9% | −62.1%  | 0.657 → 0.857 (+30%) |
| adaptive  | −13.3% | −58.4% | −61.1%  | 0.657 → 0.629 (−4%)  |
| static    | +7.2%  | −46.0% | −45.2%  | 0.657 → 0.586 (−11%) |
| parallel  | +12.0% | +19.9% | +125.8% | tied               |

vs static — the fair baseline (hand-designed DAG, no optimizer):

| strategy  | tokens | cost   | latency | quality            |
|-----------|--------|--------|---------|--------------------|
| optimized | −22.3% | −25.7% | −30.8%  | 0.586 → 0.857 (+46%) |
| adaptive  | −19.1% | −22.9% | −29.0%  | 0.586 → 0.629 (+7%)  |

The honest headline is the second table: **~20–30% less compute than a competent
hand-built DAG**, not 60% vs a deliberately dumb chain. Optimizer also cut the graph
5 edges → 2 (fake-edge + transitive removal).

**Cost savings, live (Groq usage-priced):** sequential $0.00313 → optimized **$0.00125
(−59.9%)**, adaptive $0.00130 (−58.4%); vs hand DAG $0.00169 → −25.7%/−22.9%.
Naive parallel is the *most* expensive ($0.00375) — parallelism without routing burns
cash. Savings come from two engines: fewer tokens overall *plus* routing pushing
extraction/classification to `gpt-oss-20b` ($0.00005/1k in) while reasoning stays on
120b. Known leak (in ablation): on easy tasks the router overbuys strong models —
mock fixed-tier routing was cheaper ($0.0038 vs $0.0049) with tied quality.

Task B: `Survey open-source LLM serving stacks` (adaptive + `--judge`, live):
- `run_20260905_232111`: 5039 toks, $0.00119, 3.9s, **LLM judge 0.767**
  (`coverage=8 factuality=7 coherence=8`, `openai/gpt-oss-120b`).
- `run_20260905_232309` (`--no-cache`): **6 planned / 5 executed / 1 skipped** —
  `adaptive_actions = [SKIP pricing_analysis: EV 0.192 < threshold, intent 0.3, wave-0]`.
  Runtime scheduler (not static pre-prune) killed the pricing branch: 4958 toks,
  $0.00129, quality 0.9. Mock mirrors it deterministically (same action, 6/5/1).

**Ablation** (`agh ablate`, mock, cache-isolated — `analysis/tables/ablation.csv`):

| variant    | tokens | cost     | quality | exec |
|------------|--------|----------|---------|------|
| full       | 2271   | $0.00489 | 0.714   | 7    |
| no-dag     | 2749   | $0.00534 | 0.800   | 7    |
| no-routing | 2273   | $0.00383 | 0.714   | 7    |
| no-skip    | 2271   | $0.00489 | 0.714   | 7    |
| no-verify  | 2271   | $0.00489 | 0.714   | 7    |
| no-cache   | 2271   | $0.00489 | 0.714   | 7    |

Reading: DAG optimization saves **+478 tokens** vs full (and note the trade honestly —
no-dag scores 0.800 vs 0.714: the optimizer trades a little coverage for efficiency).
Fixed-tier routing is cheaper ($0.0038 vs $0.0049) with tied quality here — routing pays
for strong-model insurance. no-skip/verify/cache ≈ 0 on this task (0 skip candidates,
cold cache); cache value shows on repeats (**7 cache hits**, 2 semantic-dedupes).

**Frontier:** `analysis/plots/frontier.png` + `analysis/tables/frontier.csv`
(cost vs quality, live-only) via `agh compare --plot`.

## Why this is better (each claim traces to a number above)

- **Less compute, same-or-better quality** — optimized beats sequential on all four
  axes at once; nothing is claimed without the receipt.
- **Knows where gains come from** — ablation attributes tokens to DAG/routing/skip,
  including results that flatter the baseline (no-dag quality 0.8 is in the table).
- **Quality is measured on output, not proxied** — evidence coverage + grounding +
  optional LLM rubric; the old node-count mock score is gone.
- **Adapts at runtime, observably** — skip/terminate/expand/reroute actions land in
  the receipt with reasons; the Survey run is the proof trace.
- **Efficient-AI is wired, not slides** — cache hits, dedupes, budget skips are
  counters in receipts, including a caught-and-fixed cache-poisoning bug
  (quarantined in `runs/quarantined/`).

## Reproduce everything

```bash
pip install -e .
pytest -q                                   # 15/15
# mock (no key): full pipeline + ablations + frontier table
python -m cli.main run "Compare vLLM, SGLang and TensorRT-LLM for LLM inference" --mock --strategy optimized
python -m cli.main ablate                   # → analysis/tables/ablation.csv
python -m cli.main compare --plot           # → analysis/plots/frontier.png
# live (Groq): needs GROQ_API_KEY; models in configs/models.yaml
export GROQ_API_KEY=...
python -m cli.main run "Survey open-source LLM serving stacks" --no-mock --strategy adaptive --max-parallel 2 --judge --no-cache
```

CLI: `plan|optimize|run|execute|benchmark|compare|inspect|graph|cost|trace|reproduce|ablate`
(`--judge` fills the LLM-rubric column; `--no-cache` forces fresh compute.)

## Honest limits

- Judge flakes empty ~50% on gpt-oss → retries + strong→cheap fallback + evidence score
  always recorded; some live receipts carry evidence-only quality.
- Parallel baseline latency is inflated by Groq 429 backoff, not pure scheduling.
- Per-wave TERMINATE/EXPAND are implemented + unit/mock-e2e proven, but on this DAG
  shape static edge-removal unblocks nodes first, so wave-0 SKIP does the runtime work;
  benchmark conf hit 0.46 vs EXPAND threshold 0.45 twice — threshold left untuned.
- Frontier is 10 live runs on 2 tasks: a real signal, not a benchmark suite result.

## Gates

G1 executable graph ✓ · G2 scheduler ✓ · G3 receipts ✓ · G4 baselines ✓ ·
G5 DAG optimizer ✓ · G6 routing ✓ · G7 skip ✓ · G8 verification ✓ ·
G9 adaptive runtime ✓ (wave-0 SKIP + TERMINATE/EXPAND/REROUTE, live-proven SKIP) ·
G10 efficient-AI ✓ (cache + dedup + budgets wired and counted)

## Layout

```
harness/{core,planning,optimization,execution,verification,models,observability,cache}
workloads/{research,coding,reasoning}  benchmarks/{baselines,runners,evaluation}
configs/{models,budgets,workloads,experiments}  analysis/{plots,tables}
experiments/{LIVE_RESULTS_vLLM_20260905.md,GAPFIX_VERIFICATION.md,manifests}
cli/main.py  tests/{unit,integration,failure,determinism}
```

Proof traces: `experiments/LIVE_RESULTS_vLLM_20260905.md`,
`experiments/GAPFIX_VERIFICATION.md`, `runs/*.json`.
