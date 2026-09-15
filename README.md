# Agent evals against real dependencies

A shopping agent, a labelled eval suite for it, and the wiring to score that
suite against a running cluster instead of a fixture — without deploying the
agent or standing an environment up.

The interesting part is not the score. It is that `npm run eval` and
`mirrord exec … -- npm run eval` are the same command, and only one of them
describes your shop.

## What's here

| | |
| --- | --- |
| `apps/shop/chat-service` | the shopping agent — a tool-calling loop — and its eval suite |
| `apps/shop/inventory-service`, `order-service` | the dependencies the agent reads |
| `apps/shop/metal-mart-frontend` | a storefront with a support chat, so you can watch the agent work |
| `apps/visualization-shop` | a live picture of the cluster, including mirrord sessions |
| `overlays/` | deploy it: `base`, plus `local` or `gke` |
| `.mirrord/agent-evals.json` | twenty lines; the whole integration |

The eval suite has [its own README](apps/shop/chat-service/eval/README.md)
covering the dataset, the scoring modes and the code-based judge. Read that one
for the substance.

## What you need

- a Kubernetes cluster you can reach — kind and minikube are fine
- the [mirrord Operator](https://metalbear.com/mirrord/docs/overview/teams/)
  installed on it, and the [mirrord
  CLI](https://metalbear.com/mirrord/docs/overview/quick-start/) locally
- an Anthropic API key
- Node 20+

All container images are published publicly, so nothing here needs building.

## Getting it running

**1. Deploy the shop.**

```bash
kubectl apply -k overlays/local
kubectl rollout status -n shop deployment/chat-service
```

That brings up Postgres, Kafka and RabbitMQ in `infra`, the shop services and
the visualization in `shop`, and seeds a catalogue.

**2. Point the evals at it.**

```bash
cp .env.example .env     # then edit
export EVAL_KUBE_CONTEXT=$(kubectl config current-context)
export ANTHROPIC_API_KEY=sk-ant-...
```

**3. Run them.**

```bash
cd apps/shop/chat-service
npm ci
mirrord exec --config-file ../../../.mirrord/agent-evals.json -- npm run eval
```

The eval process runs on your machine but inherits `chat-service`'s environment
and network, so it reads the real catalogue and real stock. Nothing is deployed
and no environment is created.

**A full run is not free.** 66 cases, each several model calls with thinking on,
comes to a few dollars against a frontier model. While you are finding your way
around, run a cheap subset instead — evenly spread across the case classes, so
the score still means something:

```bash
mirrord exec --config-file ../../../.mirrord/agent-evals.json -- npm run eval -- --limit 12
```

`SHOPPING_AGENT_MODEL` picks the model if you would rather score a smaller one.

Drop the `mirrord exec` prefix and the identical command scores against a frozen
catalogue in `eval/fixtures/` instead — the runner has no flag for this and
never inspects mirrord. `src/agent/deps.ts` reads the environment: service URLs
present means live, absent means the fixture.

## Configuration

Everything environment-specific is an environment variable or one marked line.

| Variable | Default | What it is |
| --- | --- | --- |
| `EVAL_KUBE_CONTEXT` | current context | which cluster to target |
| `EVAL_NAMESPACE` | `shop` | namespace the target runs in |
| `EVAL_TARGET` | `chat-service` | deployment whose environment to borrow |
| `ANTHROPIC_API_KEY` | — | used by the eval process |

The storefront's own chat runs the agent *inside* the cluster, so it needs its
own credential. Skip this and the shop still works — the chat just hands over to
a human instead of answering:

```bash
kubectl create secret generic shopping-agent -n shop \
  --from-literal=anthropic-api-key=sk-ant-...
```

### Seeing the storefront

```bash
kubectl port-forward -n shop svc/metal-mart-frontend 3000:80
open http://localhost:3000/shop
```

The visualization is at `/visualization-shop` on the same port-forward.

### HTTPS on a domain you own

`overlays/gke` puts a Google-managed certificate on a global load balancer.
It needs three things that are yours, each marked `CHANGE ME` in
`overlays/gke/ingress.yaml`: a domain, a reserved **global** static IP, and GKE
itself — `ManagedCertificate` and `BackendConfig` are Google types.

```bash
kubectl apply -k overlays/gke
```

## In CI

`.github/workflows/ci-agent-evals.yml` runs the suite on a cold runner with no
cluster of its own, gated on a threshold. It needs three secrets:
`EVAL_KUBECONFIG_BASE64`, `ANTHROPIC_API_KEY`, and `MIRRORD_CI_API_KEY`.

## This is a demo, not a template for production

The databases here ship with a well-known password (`postgres`/`postgres`) in
plain text, there is no network policy between services, and the seed data is
made up. It is built to be stood up, pointed at, and thrown away. Do not put
anything real in it, and do not lift the manifests into a cluster that matters
without changing all of that first.

What *is* worth borrowing is the shape: the eval suite, the code-based judge,
and the environment seam that lets one command score against either a fixture or
a live cluster.

## Notes

The catalogue is MetalBear merchandise and the eval cases are generated from
whatever catalogue they are pointed at, so swapping in your own products means
regenerating the dataset — `npm run eval:refresh` does that from a live cluster.

A run borrows `chat-service`'s environment and network and takes no traffic from
it (`incoming: "off"`). It writes nothing, patches nothing, and restarts nothing
— the only trace it leaves is one short-lived session pod. So a shared cluster
can carry many runs at once, and people can keep using the shop while they go.
