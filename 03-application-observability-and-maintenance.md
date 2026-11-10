# 3. Application Observability and Maintenance — 15%

> CKAD domain 3/5 · Source: [CNCF CKAD curriculum](https://github.com/cncf/curriculum/tree/master/ckad)

## 3.1 Understand API Deprecations

- [ ] Deprecation policy: API versions are removed after a deprecation window across minor releases
- [ ] Preferred vs stored version of a resource (concept)
- [ ] Discovering current apiVersions: `kubectl api-resources`, `kubectl explain <resource>`
- [ ] Migrating manifests off deprecated APIs (Kubernetes API migration guide)

## 3.2 Implement Probes and Health Checks

- [ ] Liveness probe: restarts unhealthy containers
- [ ] Readiness probe: gates traffic — removes the Pod from Service endpoints
- [ ] Startup probe: grace period for slow-start apps; defers liveness/readiness
- [ ] Handlers: `httpGet`, `tcpSocket`, `exec`, `grpc`
- [ ] Tuning knobs: `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, `failureThreshold`, `successThreshold`

## 3.3 Monitor Applications with Built-in CLI Tools

- [ ] `kubectl top pod | node` (`--containers`, `--sort-by`) — requires metrics-server
- [ ] `kubectl get --watch`, output formats and field selection
- [ ] `kubectl describe`: conditions and events in one view
- [ ] `kubectl get events --sort-by=.lastTimestamp`, `--field-selector`

## 3.4 Utilize Container Logs

- [ ] `kubectl logs` flags: `-f`, `--tail`, `--since`, `--timestamps`, `-p` (previous instance)
- [ ] Multi-container Pods: `-c <container>`; init container and sidecar logs
- [ ] Cluster-level logging flow: node agent shipping logs to a backend (concept)

## 3.5 Debugging in Kubernetes

- [ ] Pod status meanings: `Pending`, `ImagePullBackOff`, `CrashLoopBackOff`, `RunContainerError`, `OOMKilled`, `Completed`
- [ ] First checks: describe events, logs, image name/tag, resource limits, probe failures
- [ ] `kubectl exec` into running containers
- [ ] `kubectl debug`: ephemeral containers, Pod-copy debugging, node debugging profiles
- [ ] Service/endpoints connectivity debugging → [topic 5](05-services-and-networking.md)
