# 1. Application Design and Build — 20%

> CKAD domain 1/5 · Source: [CNCF CKAD curriculum](https://github.com/cncf/curriculum/tree/master/ckad)

## 1.1 Define, Build and Modify Container Images

- [ ] Dockerfile instructions: `FROM`, `RUN`, `COPY`, `ADD`, `CMD`, `ENTRYPOINT`, `ENV`, `ARG`, `WORKDIR`, `EXPOSE`, `USER`
- [ ] `CMD` vs `ENTRYPOINT` interaction and overriding at runtime
- [ ] Image layers and layer caching; effect on build time and pull size
- [ ] Multi-stage builds
- [ ] Tagging, pushing and pulling images from registries
- [ ] Inspecting images: `docker build`, `docker image inspect`, `docker history`

## 1.2 Choose and Use the Right Workload Resource

- [ ] Pod spec essentials: containers, labels, `restartPolicy`, naming
- [ ] Deployment: replicas, selector must match template labels
- [ ] Deployment → ReplicaSet → Pod ownership relationship
- [ ] DaemonSet: one Pod per matching node (node-level agents)
- [ ] Job: `completions`, `parallelism`, `backoffLimit`, `activeDeadlineSeconds`
- [ ] CronJob: cron schedule syntax, `concurrencyPolicy`, `startingDeadlineSeconds`, `suspend`, `timeZone`
- [ ] Decision criteria: which workload resource fits a given workload type

## 1.3 Multi-Container Pod Design Patterns

- [ ] Sidecar: auxiliary container enhancing the main app (log shipper, proxy)
- [ ] Native sidecars: init container with `restartPolicy: Always` (Kubernetes 1.29+), starts before app containers
- [ ] Ambassador: per-Pod proxy to external dependencies
- [ ] Adapter: transforms the output of the main container
- [ ] Init containers: sequential startup order, bootstrap tasks
- [ ] Sharing state between containers: `emptyDir` volumes, localhost networking within a Pod

## 1.4 Utilize Persistent and Ephemeral Volumes

- [ ] `emptyDir` (Pod lifetime) vs `hostPath` (node filesystem)
- [ ] PersistentVolume (PV) → PersistentVolumeClaim (PVC) → StorageClass provisioning flow
- [ ] Access modes: `ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany`; reclaim policies
- [ ] Claim sizing and binding; `storageClassName`
- [ ] Mounting `configMap`, `secret` and `projected` volumes into Pods
- [ ] Generic ephemeral volumes; CSI drivers (concept)
