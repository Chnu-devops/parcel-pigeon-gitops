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
├── dev/
│   ├── root.yaml                # bootstrap, applied by hand once
│   └── apps/parcelpigeon-dev.yaml   # ApplicationSet for dev
└── prod/
    ├── root.yaml
    └── apps/parcelpigeon-prod.yaml
.github/workflows/promote.yml    # opens the dev → prod promotion PR
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
minikube start --memory 6g --cpus 4
minikube addons enable ingress
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
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
```

Full walkthrough: `docs/environments-migration.md` in the app repo.
