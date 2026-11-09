# SRE Copilot: observability and release safety

SRE Copilot is an AI agent that diagnoses broken Kubernetes workloads: it retrieves
runbooks, calls read-only cluster tools, and returns a cited diagnosis. This folder shows
how the agent is **observed, costed and released safely** on the platform. The agent's
own source is not published here.

## How it works

![Architecture](images/post-1-architecture.svg)

![Agent workflow](images/post-2a-agent-workflow.svg)

![RAG pipeline](images/post-2b-rag-pipeline.svg)

## Observability pipeline

Grafana Alloy is the single collector. It scrapes metrics, tails pod logs, and receives
OTLP traces from the agent.

```
agent / executor / eval jobs ──OTLP──▶ Alloy ──▶ Tempo   (traces)
cluster + app metrics        ───────▶ Alloy ──▶ Mimir   (metrics)
pod logs + k8s events        ───────▶ Alloy ──▶ Loki    (logs)
                                                  └──▶ Grafana
```

| File | What it does |
|---|---|
| [`alloy/kustomization.yaml`](alloy/kustomization.yaml) | Alloy via the `k8s-monitoring` chart (pinned): cluster metrics, pod logs, events, ServiceMonitor/PodMonitor discovery, destinations Mimir + Loki |
| [`alloy/sre-copilot.alloy`](alloy/sre-copilot.alloy) | OTLP receiver → batch → Tempo, and a Loki stage that gives audit events (`log_type=audit`) their own `stream="audit"` with `trace_id` as structured metadata |
| [`dashboards/sre-copilot.json`](dashboards/sre-copilot.json) | Cost and agent metrics dashboard |
| [`dashboards/sre-cluster-live.json`](dashboards/sre-cluster-live.json) | Live "what is broken right now" dashboard |

![Observability](images/post-6-observability.svg)

## Cost per diagnosis

The dashboard separates **real LLM cost**, **shadow LLM cost** (what the same tokens
would cost on a priced API) and **infra cost** (cluster cost attributed to the agent namespace), then adds them into a total cost per diagnosis. It also tracks p50/p95
latency, steps per request, KEDA in-flight requests vs replicas, guardrail hits,
time to first token, tool calls and retrieval quality.

> **Simulated model.** This run used a stand-in model (`sim-model`) to exercise the
> full pipeline under load without a real LLM. So "Real LLM $" is $0.00, and the shadow
> cost is a calculation, not a bill. The 2,980 diagnoses and the latency and autoscaling
> behaviour came from the real agent and the real cluster; the model's answers did not
> come from a real LLM.

![Cost and agent metrics](images/grafana-sre-copilot-cost.png)

## Live cluster issues

The second dashboard reads cluster state and Loki logs. Here it caught a genuine
`OOMKilled` on `argocd-repo-server`.

![Live cluster issues](images/grafana-live-cluster-issues.png)

![Grafana post](images/post-24-grafana.svg)

## Safe releases: canary gated by evals

A new agent version is not promoted on a green health check. Argo Rollouts shifts 20%
of traffic, then runs analysis, pauses, goes to 50%, pauses, then 100%.
[`rollouts/analysis-template.yaml`](rollouts/analysis-template.yaml) runs the eval suite
as a Job against the canary: exit code 0 promotes, non-zero aborts and rolls back.
The registry and image tag are placeholders in this copy.

The screenshots show the history: earlier revisions that failed analysis (red) and
were rolled back, then revisions that passed. (AWS account ID redacted.)

![Rollout paused at step 2/6](images/rollouts-canary-paused.png)

![Rollout with a new canary revision](images/rollouts-canary-analysis.png)

![Progressive delivery](images/post-4-progressive-delivery.svg)

## Autoscaling, CI and guardrails

![Autoscaling](images/post-3-autoscale.svg)

![CI eval](images/post-5-ci-eval.svg)

![Guardrails](images/post-7-guardrails.svg)

## Tool posts

![Alloy](images/post-20-grafana-alloy.svg)

![Mimir](images/post-21-mimir.svg)

![Loki](images/post-22-loki.svg)

![Argo Rollouts](images/post-12-argo-rollouts.svg)
