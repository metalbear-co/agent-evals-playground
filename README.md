# Agent evals against real dependencies

A shopping agent, 66 labelled test cases, and a runner that scores it against
either a JSON fixture or a live Kubernetes cluster.

There is no flag for picking which. The runner reads `INVENTORY_SERVICE_URL` and
`ORDER_SERVICE_URL` from its environment. Unset, the agent falls back to a
catalogue snapshot checked into the repo. Set, it calls the real services.

```bash
npm run eval                            # scores against the fixture
mirrord exec ... -- npm run eval        # scores against your cluster
```

The second command inherits its environment from a running deployment, so those
URLs arrive already set and resolve inside the cluster. The agent itself still
runs locally. Nothing is deployed and no environment is created.

## What's here

| | |
| --- | --- |
| `apps/shop/chat-service` | the shopping agent (a tool-calling loop) and its eval suite |
| `apps/shop/inventory-service`, `order-service` | the dependencies the agent reads |
| `apps/shop/metal-mart-frontend` | a storefront with a support chat, so you can watch the agent work |
| `apps/visualization-shop` | a live picture of the cluster, including mirrord sessions |
| `overlays/` | deploy it: `base`, plus `local` or `gke` |
| `.mirrord/agent-evals.json` | the mirrord config, 20 lines |

[apps/shop/chat-service/eval/README.md](apps/shop/chat-service/eval/README.md)
covers the dataset, the scoring modes and the judge in detail.

## What you need

- a Kubernetes cluster you can reach. kind and minikube are fine
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

**2. Give the agent a credential.**

```bash
kubectl create secret generic shopping-agent -n shop \
  --from-literal=anthropic-api-key=sk-ant-...
kubectl rollout restart -n shop deployment/chat-service
```

The key lives in the cluster, not on your machine. A run picks it up from the
target along with the service URLs, so you do not need a copy locally to score
the suite.

**3. Point the evals at the cluster.**

```bash
export EVAL_KUBE_CONTEXT=$(kubectl config current-context)
```

**4. Run them.**

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
around, run a subset instead. `--limit` spreads evenly across the case classes,
so a partial score still means something:

```bash
mirrord exec --config-file ../../../.mirrord/agent-evals.json -- npm run eval -- --limit 12
```

`SHOPPING_AGENT_MODEL` picks the model if you would rather score a smaller one.

The switch between fixture and cluster lives in `src/agent/deps.ts`, not in the
runner. Nothing in the eval code knows mirrord exists.

## Configuration

Everything environment-specific is an environment variable or one marked line.

| Variable | Default | What it is |
| --- | --- | --- |
| `EVAL_KUBE_CONTEXT` | current context | which cluster to target |
| `EVAL_NAMESPACE` | `shop` | namespace the target runs in |
| `EVAL_TARGET` | `chat-service` | deployment whose environment to borrow |
| `ANTHROPIC_API_KEY` | from the target | only needed for a fixture run |

A run under mirrord takes the key from the target's environment, so the last row
is usually nothing you set. Export it locally only to score against the fixture,
where there is no cluster to take it from.

Skipping the `shopping-agent` secret leaves the shop working. The storefront chat
hands over to a human instead of answering, and the evals have no credential.

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
itself, since `ManagedCertificate` and `BackendConfig` are Google types.

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
made up. Do not put anything real in it, and do not copy these manifests into a
cluster that matters without fixing all of that first.

The parts worth copying are the eval suite, the code-based judge, and reading
dependency choice from the environment instead of passing it as a flag.

## Notes

The catalogue is MetalBear merchandise and the eval cases are generated from
whatever catalogue they are pointed at, so swapping in your own products means
regenerating the dataset. `npm run eval:refresh` does that from a live cluster.

A run borrows `chat-service`'s environment and network and takes no traffic from
it (`incoming: "off"`). It writes nothing, patches nothing and restarts nothing.
The only trace is one short-lived session pod, so several runs can share a
cluster while people keep using the shop.
