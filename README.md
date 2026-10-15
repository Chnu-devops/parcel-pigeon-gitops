# parcel-pigeon-gitops

Deployment config for [ParcelPigeon](https://github.com/Chnu-devops/parcel-pigeon).
This is the **only** repo Argo CD reads: what's on `main` here is what runs in
every environment. Application code, Dockerfiles and CI live in the app repo.

```
apps/                            # our services — images built by app-repo CI
├── gateway/
│   ├── base/                    # the Helm chart — shared by all environments
│   │   ├── Chart.yaml
│   │   ├── values.yaml          # defaults
│   │   └── templates/
│   └── overlays/
│       ├── dev/values.yaml      # image tag (CI), debug logs, dev host
│       └── prod/values.yaml     # image tag (promotion PR), replicas: 2
├── web/  shipments-service/  tracking-service/
services/                        # datastores & infra — upstream images, no CI tags
├── postgres/                    # same base/ + overlays/ shape
├── redis/  rabbitmq/  mailhog/
lib/
└── parcelpigeon-common/         # library chart: shared label helpers
clusters/
├── platform/                    # cluster-wide controllers, applied once before the envs
│   ├── root.yaml
│   ├── apps/
│   │   ├── argo-rollouts.yaml          # Argo Rollouts (Helm chart)
│   │   ├── kube-prometheus-stack.yaml  # Prometheus, Alertmanager, Grafana (release `kps`)
│   │   ├── loki.yaml                   # log storage
│   │   ├── alloy.yaml                  # log collector → Loki
│   │   └── parcelpigeon-observability.yaml
│   └── observability/           # shared Grafana dashboard "ParcelPigeon RED"
├── dev/
│   ├── root.yaml                # bootstrap, applied by hand once
│   └── apps/parcelpigeon-dev.yaml   # ApplicationSet for dev
└── prod/
    ├── root.yaml
    └── apps/parcelpigeon-prod.yaml
.github/workflows/promote.yml    # opens the dev → prod promotion PR
samples/rollouts/                # Argo Rollouts examples, applied by hand — not synced
```

- **A component runs in an environment iff it has `overlays/<env>/`**, in
  either `apps/` or `services/`. Each env's ApplicationSet makes one
  Application per such folder: `gateway-dev`, `postgres-prod`, … labelled
  `env=<env>` and `tier=apps|services`.
- Values are merged `base/values.yaml` → `overlays/<env>/values.yaml`; the
  overlay wins, maps are merged key by key.
- Each environment gets its own namespace: `parcelpigeon-dev`,
  `parcelpigeon-prod`. Here "dev" and "prod" clusters are the same minikube;
  on real infrastructure only the `destination.server` in `clusters/prod/`
  would change.
- **gateway** and **web** are Argo Rollouts `Rollout`s, not `Deployment`s.
  gateway is a canary, checked by its `/readyz` analysis. web is blue-green.
  Dev releases on its own. Prod pauses until someone runs
  `kubectl argo rollouts promote`.
- **Monitoring** lives next to the code it watches: gateway, shipments-service
  and tracking-service each ship a `ServiceMonitor` (scrape `/metrics` on the
  Service's `http` port) and a `PrometheusRule` (`<Svc>HighErrorRate`,
  `<Svc>Down`), behind `monitoring.enabled`. The gateway canary is checked
  twice: `/readyz`, then the canary's 5xx share in Prometheus
  (`gateway-success-rate`). Set `config.ERROR_RATE: "0.3"` in a gateway
  overlay to make a bad release on purpose.

## How a change reaches prod

```
app repo push to main
  └─ CI builds ghcr.io/chnu-devops/<svc>:<sha>
     └─ commits apps/<svc>/overlays/dev tag   → Argo CD deploys to dev
        └─ promote.yml opens/updates PR "Promote dev → prod"
           └─ review + merge                  → Argo CD deploys to prod
```

Image tags in `apps/*/overlays/` are written by CI / the promotion PR — don't
hand-edit them. Everything else (config, replicas, resources, datastore
versions in `services/`, new components) is a normal PR.

## Bootstrap

```bash
minikube start --memory 8g --cpus 4   # the monitoring stack needs the extra 2g
minikube addons enable ingress
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -f clusters/platform/root.yaml   # first: Rollout and ServiceMonitor CRDs are needed by the apps
kubectl apply -f clusters/dev/root.yaml -f clusters/prod/root.yaml
kubectl -n argocd get applications -L env,tier
```

## Day 2

```bash
argocd app list -l env=prod
argocd app list -l tier=services
argocd app history gateway-prod
git log --oneline -- apps/gateway/overlays/prod/   # deploy log of one service in prod
git revert <commit> && git push                    # roll it back (via PR in prod)

# releases (brew install argoproj/tap/kubectl-argo-rollouts)
kubectl argo rollouts get rollout gateway -n parcelpigeon-prod --watch
kubectl argo rollouts promote gateway -n parcelpigeon-prod   # past the manual pause
kubectl argo rollouts promote web -n parcelpigeon-prod       # switch blue → green
kubectl argo rollouts abort gateway -n parcelpigeon-prod     # stop; then git revert the tag
kubectl argo rollouts dashboard                              # http://localhost:3100

# observability
kubectl -n monitoring port-forward svc/kps-grafana 3000:80                              # http://localhost:3000, user admin
kubectl -n monitoring get secret kps-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
kubectl -n monitoring port-forward svc/kps-kube-prometheus-stack-prometheus 9090        # Status → Targets
kubectl -n monitoring port-forward svc/kps-kube-prometheus-stack-alertmanager 9093      # firing alerts
kubectl -n parcelpigeon-dev get analysisrun                                              # canary checks and their results
```

Full walkthroughs in the app repo: `docs/environments-migration.md`, `docs/rollouts-migration.md`,
`docs/observability-migration.md`.
