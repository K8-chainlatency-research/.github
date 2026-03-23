# K8-chainlatency-research

**Quantifying the Cold-Start Tax: Tail-Latency Amplification from Pod Scheduling in Kubernetes Microservice Chains**

*NSDI '26 Poster Track*

---

## The Problem

Modern cloud applications decompose into dozens of microservices orchestrated by Kubernetes. A single user request may traverse a chain of *D* services, each of which can independently experience a **cold start** — the multi-second delay when a new pod is scheduled, image-pulled, and initialized.

If each service cold-starts with probability *r*, the chance that **at least one** hop in a depth-*D* chain hits a cold start is:

```
P_chain = 1 - (1 - r)^D
```

At *r* = 5% and *D* = 7, that's **~30% of requests** encountering at least one cold start. While individual cold-start latencies are well studied in serverless contexts, their **multiplicative effect on tail latency** across service chains has not been systematically quantified — until now.

## What We Built

### ChainBench — Benchmark Harness

A configurable-depth microservice chain deployed on a local [kind](https://kind.sigs.k8s.io/) cluster with Istio service mesh and Jaeger distributed tracing. Each of the 7 Go services performs synthetic CPU work (5 ms) and network delay (2 ms), then forwards the request to the next hop. Cold-start delays are injected using a **calibrated log-normal distribution** (median = 2100 ms, σ = 0.4) fitted to real Kubernetes pod startup measurements and validated via KS-test (*p* > 0.15).

### Pre-Warming Controller — Mitigation

A custom Kubernetes controller that watches HPA scale-up events and CPU utilization via Prometheus. When scaling activity is detected, it **proactively creates warm pods** for downstream services in the call graph before traffic arrives. Idle pre-warmed pods are automatically reaped after a configurable TTL (default 60s) to avoid resource waste.

### Experiment Suite — 72 Controlled Runs

A fully scripted experiment harness covering 6 experiment types across 24 configurations, each repeated 3 times:

| Experiment | What it varies | Configs |
|-----------|---------------|---------|
| Baseline | QPS ∈ {50, 100, 200, 500}, no cold-starts | 4 |
| Rate sweep | Cold-start rate *r* ∈ {1%, 5%, 10%, 25%} | 4 |
| Depth sweep | Chain depth *D* ∈ {3, 5, 7} | 3 |
| Position sweep | Injection position *P* ∈ {1, 3, 5, 7} | 4 |
| Real cold-start | Actual pod deletion at *P* ∈ {2, 4, 6} | 3 |
| Mitigation | Pre-warming on/off × *r* ∈ {5%, 10%, 25%} | 6 |

### Analysis Pipeline — Figures & Statistics

Python pipeline that computes amplification factors, aggregates across repetitions with 95% confidence intervals, runs Mann-Whitney U significance tests, and generates 5 publication-quality figures (PDF + PNG, colorblind-friendly, USENIX-formatted).

## Repositories

| Repo | Language | Description |
|------|----------|-------------|
| [**chainbench**](https://github.com/K8-chainlatency-research/chainbench) | Go | 7-service instrumented microservice chain on kind + Istio + Jaeger |
| [**controller**](https://github.com/K8-chainlatency-research/controller) | Go | Predictive pre-warming Kubernetes controller with Prometheus integration |
| [**experiments**](https://github.com/K8-chainlatency-research/experiments) | Bash | 72-run experiment harness with wrk2 load generation and automated data collection |
| [**analysis**](https://github.com/K8-chainlatency-research/analysis) | Python | Amplification analysis, statistical tests, and 5 publication-quality figures |

## Key Results

```
Cold-start rate r=5%, chain depth D=7, QPS=100:

  Percentile    Baseline    With Cold-Starts    Amplification
  ─────────────────────────────────────────────────────────────
  p50            51 ms         54 ms              1.05×
  p99            58 ms        476 ms              8.2×
  p99.9          61 ms       2501 ms             41.0×

With pre-warming controller enabled:

  Percentile    Without       With Controller     Reduction
  ─────────────────────────────────────────────────────────────
  p99           8.2×          2.3×                72%
  p99.9        41.0×         18.6×                55%
```

**Four key findings:**
1. At 5% cold-start rate and depth 7, p99 is amplified **8.2×** and p99.9 is amplified **41×** — the impact is concentrated entirely in the tail
2. Increasing chain depth from 3 to 7 raises p99 amplification from **3.8×** to **8.2×**, consistent with the `1-(1-r)^D` compounding model
3. Cold-start **position** in the chain changes p99 by less than **9%** — any service can dominate the tail
4. The pre-warming controller reduces p99 amplification by **72%**, but residual p99.9 inflation (18.6×) shows complementary strategies (hedging, topology-aware routing) are still needed

## Reproduce

```bash
# 1. Deploy the benchmark cluster
cd chainbench
make cluster && make deploy && make observe

# 2. Verify the chain works
curl http://localhost:30080/chain | python3 -m json.tool
# Returns JSON with timing from all 7 services

# 3. Open observability UIs
# Jaeger:     http://localhost:16686  (distributed traces, 7 spans per request)
# Prometheus: http://localhost:9090   (request_duration_seconds, cold_start_total)
# Grafana:    http://localhost:3000   (pre-built dashboards)

# 4. Run all 72 experiments (~8.4 hours)
cd ../experiments
cp .env.example .env    # edit endpoints if needed
./run_all.sh

# 5. Generate figures from results (or use --mock for synthetic data)
cd ../analysis
pip install -r requirements.txt
python3 generate_figures.py --results-dir ../experiments/results
# Produces: figures/fig{1-5}_*.{pdf,png}

# 6. Quick test with mock data (no cluster needed)
python3 generate_figures.py --mock
```

## Tech Stack

- **Services:** Go 1.21+, OpenTelemetry, Prometheus client
- **Infrastructure:** kind, Istio, Jaeger, Prometheus, Grafana
- **Load generation:** wrk2 with custom Lua scripts
- **Analysis:** Python 3.8+, NumPy, Pandas, Matplotlib, SciPy, Seaborn
- **Controller:** controller-runtime, k8s.io/client-go

## Citation

```bibtex
@inproceedings{chainbench-nsdi26,
  title     = {Quantifying the Cold-Start Tax: Tail-Latency Amplification
               from Pod Scheduling in {Kubernetes} Microservice Chains},
  author    = {Sanket},
  booktitle = {Poster at the 23rd USENIX Symposium on Networked Systems
               Design and Implementation (NSDI '26)},
  year      = {2026}
}
```
