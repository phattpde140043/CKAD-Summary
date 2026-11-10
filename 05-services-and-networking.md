# 5. Services and Networking — 20%

> CKAD domain 5/5 · Source: [CNCF CKAD curriculum](https://github.com/cncf/curriculum/tree/master/ckad)

## 5.1 NetworkPolicies

- [ ] Default is allow-all; a Pod selected by any NetworkPolicy becomes isolated for the covered direction(s)
- [ ] `podSelector` (+ optional `namespaceSelector`) selects target Pods; empty selector `{}` selects all
- [ ] `policyTypes`: `Ingress`, `Egress`, or both
- [ ] Ingress rules: `from` (`podSelector` / `namespaceSelector` / `ipBlock`) + `ports`
- [ ] Egress rules: `to` + `ports`; DNS egress on port 53 is a common requirement
- [ ] `ipBlock` with `except` CIDR exclusion
- [ ] Enforcement depends on the CNI plugin (concept)

## 5.2 Provide and Troubleshoot Access via Services

- [ ] Types: `ClusterIP` (default), `NodePort`, `LoadBalancer`, `ExternalName`; headless Service: `clusterIP: None`
- [ ] `port` vs `targetPort` vs `nodePort`; multi-port Services require named ports
- [ ] Selector-based membership; selector-less Service with manually managed `Endpoints`
- [ ] Service DNS: `<svc>.<namespace>.svc.cluster.local`; short names resolve within the same namespace
- [ ] Empty endpoints diagnosis: label/selector mismatch, failing readiness probe, wrong `targetPort`
- [ ] Connectivity checks: `kubectl get endpoints`, DNS lookup from a temp Pod, curl the Service IP

## 5.3 Expose Applications with Ingress

- [ ] Ingress resource: HTTP(S) routing by `host` and `path`; `pathType`: `Prefix`, `Exact`, `ImplementationSpecific`
- [ ] Requires an Ingress controller in the cluster (concept)
- [ ] `ingressClassName` (or the legacy `kubernetes.io/ingress.class` annotation)
- [ ] TLS termination: `tls` section + Secret of type `kubernetes.io/tls`
- [ ] `defaultBackend` for unmatched requests
- [ ] Routing patterns: path fan-out, name-based virtual hosting
