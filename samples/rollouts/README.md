# Argo Rollouts samples

Standalone examples for the progressive-delivery lecture. **Argo CD does not
deploy this folder.** The ApplicationSets only match `apps/*/overlays/*` and
`services/*/overlays/*`. You apply these by hand into a scratch namespace.
They use `argoproj/rollouts-demo`, which serves a coloured page on `:8080`
(`/color` returns the colour). The `bad-*` tags return errors.

ParcelPigeon's real Rollouts are in `apps/gateway/base` (canary) and
`apps/web/base` (blue-green).

| File | Shows |
|---|---|
| `01-canary-basic.yaml` | canary steps: `setWeight`, `pause: {}`, timed `pause`, weighted by pod count |
| `02-canary-nginx.yaml` | canary with `trafficRouting.nginx`: 5% of requests, independent of pod count |
| `03-bluegreen.yaml` | blue-green: preview Service, manual promotion, pre- and post-promotion analysis |
| `04-analysis-templates.yaml` | `web`, `job` and `prometheus` providers, plus background analysis with an inline analysis step |
| `05-workloadref.yaml` | turning an existing `Deployment` into a Rollout without rewriting it |

## Run one

The controller must already be running (`clusters/platform/`). You also need the
kubectl plugin: `brew install argoproj/tap/kubectl-argo-rollouts`.

```bash
kubectl create namespace rollouts-lab
kubectl apply -n rollouts-lab -f samples/rollouts/01-canary-basic.yaml

# second terminal
kubectl argo rollouts get rollout demo -n rollouts-lab --watch

kubectl argo rollouts set image demo demo=argoproj/rollouts-demo:yellow -n rollouts-lab
kubectl argo rollouts promote demo -n rollouts-lab         # past the pause: {}
kubectl argo rollouts promote demo -n rollouts-lab --full  # skip all remaining steps
kubectl argo rollouts abort demo -n rollouts-lab           # back to stable
kubectl argo rollouts retry rollout demo -n rollouts-lab   # try the aborted release again
kubectl argo rollouts dashboard                            # UI on http://localhost:3100
```

`03` and the background-analysis Rollout in `04` use the `http-ok` and
`smoke-job` templates, so apply `04` first. To watch an analysis fail, use
`kubectl argo rollouts set image demo-bga demo=argoproj/rollouts-demo:bad-red -n rollouts-lab`.
The `success-rate` (Prometheus) template is only a reference until Prometheus
runs in the cluster.

These are imperative `set image` demos, which is fine here because nothing in
Git owns `rollouts-lab`. On ParcelPigeon itself the image comes from Git, and
Argo CD's selfHeal would revert a `set image`.

Clean up with `kubectl delete namespace rollouts-lab`.

## Exercises (from the slides)

1. In `01`, change the steps to 10% → 30% → 60% with one-minute pauses. With 5
   replicas, what weight do you actually get?
2. In `03`, set `autoPromotionEnabled: true`. What happens to the
   pre-promotion analysis?
3. Make the background analysis in `04` stop a release that uses a `bad-*` image.
