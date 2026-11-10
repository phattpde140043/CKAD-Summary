# 4. Application Environment, Configuration and Security — 25%

> CKAD domain 4/5 · Source: [CNCF CKAD curriculum](https://github.com/cncf/curriculum/tree/master/ckad)

## 4.1 Resources That Extend Kubernetes (CRD, Operators)

- [ ] CustomResourceDefinition → custom resources; naming `plural.group/version`
- [ ] Discovering extensions: `kubectl api-resources`, `kubectl api-versions`
- [ ] CRD schema via `openAPIV3Schema` (structural schema); creating and patching custom resources
- [ ] Operator pattern: custom controller automating an application's lifecycle (concept)

## 4.2 Authentication, Authorization and Admission Control

- [ ] Request flow: authentication → authorization → admission control
- [ ] RBAC objects: `Role`, `ClusterRole`, `RoleBinding`, `ClusterRoleBinding`
- [ ] Rules: verbs (`get`, `list`, `watch`, `create`, `update`, `patch`, `delete`), `resources`, `apiGroups`
- [ ] Binding subjects: users, groups, ServiceAccounts
- [ ] Verification: `kubectl auth can-i`, impersonation with `--as`

## 4.3 Requests, Limits, Quotas

- [ ] Namespace governance: `ResourceQuota` for compute (requests/limits) and object counts
- [ ] `LimitRange`: default / min / max per container or Pod
- [ ] Quota scoping and enforcement behavior when exceeded

## 4.4 ConfigMaps

- [ ] Create: `kubectl create configmap --from-literal | --from-file | --from-env-file`; declarative manifest
- [ ] Consume: individual env vars (`env` + `valueFrom.configMapKeyRef`), bulk env (`envFrom`), volume mounts
- [ ] Passing values into container args via `$(ENV_VAR)`
- [ ] Update behavior: volume-mounted ConfigMaps refresh (with delay); env vars do not update
- [ ] `immutable: true`; `data` vs `binaryData`

## 4.5 Define Resource Requirements

- [ ] Units: CPU cores and milli-cores (`500m`); memory `Mi` / `Gi`
- [ ] `requests` drive scheduling fit; `limits` cap runtime usage
- [ ] Exceeding limits: CPU throttled, memory → `OOMKilled`
- [ ] QoS classes: `Guaranteed`, `Burstable`, `BestEffort`; eviction order under node pressure

## 4.6 Create and Consume Secrets

- [ ] Types: `Opaque` (default), `kubernetes.io/dockerconfigjson`, `kubernetes.io/tls`
- [ ] Create: `kubectl create secret ... --from-literal | --from-file`; `data` (base64) vs `stringData` (plaintext)
- [ ] Consume: `env` + `secretKeyRef`, `envFrom`, volume mounts
- [ ] `imagePullSecrets` on Pods and ServiceAccounts
- [ ] `immutable: true`; TLS Secrets for Ingress → [topic 5](05-services-and-networking.md)

## 4.7 ServiceAccounts

- [ ] Every namespace has a `default` SA; assign per Pod with `spec.serviceAccountName`
- [ ] `automountServiceAccountToken`: Pod-level and SA-level override
- [ ] RBAC for apps: bind Roles to ServiceAccounts
- [ ] Projected `serviceAccountToken` volume; token expiry (concept)
- [ ] `imagePullSecrets` attached to a ServiceAccount

## 4.8 Application Security (SecurityContexts, Capabilities)

- [ ] Pod-level `securityContext`: `runAsUser`, `runAsGroup`, `runAsNonRoot`, `fsGroup`, `supplementalGroups`, `seccompProfile`
- [ ] Container-level: `privileged`, `allowPrivilegeEscalation`, `readOnlyRootFilesystem`
- [ ] Linux capabilities: `add` / `drop` (e.g. drop `ALL`, add `NET_BIND_SERVICE`)
- [ ] Precedence: container-level settings override Pod-level defaults
