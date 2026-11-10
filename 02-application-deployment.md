# 2. Application Deployment — 20%

> CKAD domain 2/5 · Source: [CNCF CKAD curriculum](https://github.com/cncf/curriculum/tree/master/ckad)

## 2.1 Deployment Strategies via Kubernetes Primitives

- [ ] Blue/green: two Deployments (old + new version); Service cut-over by switching the label selector
- [ ] Canary: small-scale new-version Deployment alongside stable; traffic split by replica ratio or selector
- [ ] Rollback plan per strategy: re-point selector, scale down the previous version

## 2.2 Deployments and Rolling Updates

- [ ] `strategy.rollingUpdate`: `maxSurge`, `maxUnavailable`; `Recreate` strategy
- [ ] `kubectl rollout status | history | undo | pause | resume`
- [ ] `revisionHistoryLimit`; documenting changes with the `kubernetes.io/change-cause` annotation
- [ ] `minReadySeconds` and `progressDeadlineSeconds`
- [ ] `kubectl set image` and editing live specs

## 2.3 Helm — Deploy Existing Packages

- [ ] Chart repositories: `helm repo add | list | update | search`
- [ ] Release lifecycle: `helm install | upgrade | rollback | uninstall`, `helm list`, `helm status`
- [ ] Value overrides: `--values` file, `--set` flags; defaults via `helm show values`
- [ ] Rendering checks: `helm template`, `helm install --dry-run --debug`
- [ ] Inspecting a release: `helm get manifest | values | notes`

## 2.4 Kustomize

- [ ] `kustomization.yaml` building blocks: `resources`, `images`, `replicas`, `labels`
- [ ] Patches: strategic merge patch and JSON6902 patch with target selection
- [ ] `configMapGenerator` and `secretGenerator`
- [ ] Base + overlays directory layout
- [ ] Apply with `kubectl apply -k`; preview rendered output
