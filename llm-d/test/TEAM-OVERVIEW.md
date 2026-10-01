# llm-d Feature Validation — Team Overview

A reusable, scenario-based test harness for proving **llm-d** features work on customer OpenShift clusters — with measurable performance evidence, not just a smoke test.

**Location:** `rhoai3/llm-d/test/`  
**Detailed docs:** [README.md](README.md) · [TESTPLAN.md](TESTPLAN.md)

---

## What it is

An automated validation framework that:

1. Deploys a specific `LLMInferenceService` configuration for each llm-d feature
2. Generates realistic load with [GuideLLM](https://github.com/vllm-project/guidellm) (traffic patterns tuned per feature)
3. Collects benchmark metrics (TTFT, ITL, RPS, error rate)
4. Correlates results with **Prometheus/Grafana** dashboards (KV cache hits, per-pod routing, EPP health)

Each scenario is self-contained: manifests, benchmark job, pass criteria, and a results template. The orchestrator (`test-script.sh`) handles deploy → smoke test → benchmark → collect → cleanup.

```
test-script.sh  →  LLMInferenceService  →  vLLM + llm-d EPP
       ↓
  GuideLLM Job  →  Inference Gateway  →  benchmark artifacts + Grafana
```

---

## Why use it with customers

| Benefit | Detail |
|---|---|
| **Repeatable** | Same scenarios, same model, same pass criteria — comparable across clusters and engagements |
| **Evidence-based** | Benchmark numbers + Grafana panels prove routing, caching, and scheduling behavior |
| **Aligned with RHOAI** | Coverage maps to the [RHOAI 3.5 llm-d deployment guide](docs/Red_Hat_OpenShift_AI_Self-Managed-3.5-Deploy_models_using_Distributed_Inference_with_llm-d-en-US.pdf) |
| **Customer-ready output** | Per-scenario `RESULTS.md` with pass/fail, metrics tables, and Grafana screenshots |
| **Pick your scope** | Run one scenario for a focused demo, or the full suite for a comprehensive validation |

The goal is to show that llm-d features produce **measurable wins** — e.g. prefix-cache routing cuts TTFT by 5× on warm traffic, or 4-replica scaling doubles throughput vs baseline.

---

## Prerequisites

Before running tests on a customer cluster:

1. Base llm-d stack deployed via [`../llmd-script.sh`](../llmd-script.sh) (GPU operators, RHOAI, gateway, `demo-llm` namespace)
2. Monitoring overlay: `oc apply -k ../overlays/09-llm-d-monitoring/`
3. `ghcr-pull-secret` in `demo-llm` (for GuideLLM image)
4. GPU capacity: **2 GPUs** for most scenarios; **4 GPUs** for Test 04a
5. `oc` access with cluster admin or equivalent

Standard model across all scenarios: **`RedHatAI/Qwen3-8B-FP8-dynamic`** (1 GPU per replica, 32Gi memory limit).

---

## Quick start

```bash
cd rhoai3/llm-d/test

# Verify cluster, gateway, and monitoring
./test-script.sh setup

# Run one scenario (~5–10 min)
./test-script.sh run 00-baseline

# Run the full suite
./test-script.sh run-all
```

---

## Scenario coverage

| ID | Scenario | Feature validated | GPUs |
|---|---|---|---|
| **00** | Baseline | Default deployment (no custom llm-d config) | 2 |
| **01a** | Prefix-cache routing | `prefix-cache-scorer` — routes shared-prefix traffic to cached pod | 2 |
| **01b** | Precise prefix-cache | Token-level `precise-prefix-cache-scorer` vs hash-based 01a | 2 |
| **01c** | Queue + KV scheduling | Spillover to second replica when cache pod saturates | 2 |
| **01d** | Round-robin control | A/B: naive random routing vs 01a (quantifies cache benefit) | 2 |
| **02a** | Global cache indexing | EPP tracks cache state across many unique prefixes | 2 |
| **03a** | P/D separation | Prefill/decode disaggregation (Technology Preview) | 2 |
| **03b** | KV direct transfer | NIXL KV transfer under high concurrency | 2 |
| **03c** | Heterogeneous P/D | Mixed prompt/output sizes on P/D stack | 2 |
| **04a** | Data parallelism | 4-replica throughput scaling | 4 |

Additional scenarios (latency routing, batch API, WVA, WideEP, flow control) are planned — see [deferred/README.md](deferred/README.md).

---

## Customer deliverables

After each run, artifacts land in `<scenario>/results/`:

| Artifact | Purpose |
|---|---|
| `benchmark.csv` / `benchmark.json` | GuideLLM metrics (TTFT, ITL, RPS, error rate) |
| `run-metadata.txt` | Gateway URL, benchmark time window (for Grafana) |
| `RESULTS.md` | Pass/fail analysis with metrics and Grafana screenshots |
| `grafana-page*.png` | Dashboard screenshots (captured manually post-run) |

A consolidated report across all scenarios is available as [results.md](results.md) (example from a full suite run).

### Example pass criteria (01a — prefix cache routing)

- Warm-prefix TTFT at least **5× faster** than cold-start TTFT
- One pod receives **>70%** of cache hits
- GuideLLM error rate **< 1%**

Each scenario in [TESTPLAN.md](TESTPLAN.md) defines its own objective, expected Grafana signals, and pass/fail criteria.

---

## Recommended engagement flow

1. **Deploy** — Run `llmd-script.sh` on the customer cluster; apply monitoring overlay
2. **Baseline** — Run Test 00 to establish TTFT/ITL/RPS baseline
3. **Feature demos** — Pick scenarios matching the customer's use case:
   - Shared system prompts / RAG → **01a**, **01b**, **02a**
   - High concurrency / load balancing → **01c**, **04a**
   - Long-context prefill optimization → **03a–03c**
4. **A/B proof** — Run **01d** alongside **01a** to quantify prefix-cache routing benefit
5. **Report** — Generate `RESULTS.md` per scenario; share consolidated summary

Typical time: **~1–2 hours** for a focused 3–4 scenario demo; **~3–4 hours** for the full suite (including Grafana capture and report generation).

---

## Who should run it

- **Field / SA teams** — Customer PoCs and feature validation workshops
- **Support / QE** — Regression testing on new RHOAI/llm-d releases
- **Customers** — Self-service validation after following the RHOAI deployment guide

No custom code required. Scenarios are YAML manifests + a shell orchestrator. Results analysis can be assisted by an LLM using the per-scenario `prompt.txt` templates.

---

## Further reading

- [README.md](README.md) — Commands, repo layout, adding new scenarios
- [TESTPLAN.md](TESTPLAN.md) — Detailed steps, pass criteria, troubleshooting
- [../README.md](../README.md) — Cluster deployment (`llmd-script.sh`)
- [RHOAI 3.5 llm-d guide (PDF)](docs/Red_Hat_OpenShift_AI_Self-Managed-3.5-Deploy_models_using_Distributed_Inference_with_llm-d-en-US.pdf) — Official feature reference
