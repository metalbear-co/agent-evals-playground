# base

Every cluster-agnostic piece of the demo: namespaces, Postgres/Kafka/RabbitMQ in
`infra`, the shop services and the visualization in `shop`, and the catalogue
seed job.

Nothing here is tied to a cloud. Pick an overlay that layers a way in on top:

- `../gke` — Google-managed TLS certificate on a global load balancer. Needs a
  domain, a reserved global static IP, and GKE.
- `../local` — no ingress, no certificate. Reaches the storefront with
  `kubectl port-forward`. Works on kind, minikube, or any cluster at all.
