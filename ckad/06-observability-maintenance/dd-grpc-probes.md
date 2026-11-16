# Deep dive — gRPC Probes

> Cấp độ mục tiêu: **Middle**. Toàn bộ nội dung *(bổ sung từ nghiên cứu ngoài
> video)*; nền tảng probe liveness/readiness/startup đã có *(nhắc lại Bài 17)*.

## ① Concept

Kubernetes cung cấp **4 loại probe handler**: `exec`, `httpGet`, `tcpSocket`
và `grpc`. Bài 17 đã cover 3 loại đầu; file này bổ sung loại thứ tư.

**Vấn đề gRPC giải quyết:** ứng dụng gRPC không nói HTTP/1.1, nên `httpGet`
probe không gọi được endpoint của ứng dụng (trừ khi ứng dụng expose thêm
HTTP endpoint riêng). `tcpSocket` chỉ xác nhận **port mở TCP** — check
thành công kể cả khi process đã treo bên trong. Probe `grpc` check đúng
tầng ứng dụng bằng cách gọi chuẩn **gRPC Health Checking Protocol**
(`grpc.health.v1.Health/Check`) và yêu cầu trạng thái `SERVING`.

Probe `grpc` dùng được cho cả ba loại: liveness, readiness, startup.

**Ghi chú phiên bản:** GA từ **v1.27** (trước đó là feature gate
`GRPCProbes`, beta v1.24). Đề thi v1.37 → luôn khả dụng, không cần flag.

## ② Configuration (YAML)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: grpc-app
spec:
  containers:
  - name: app
    image: registry.example.com/grpc-app:1.0
    ports:
    - containerPort: 50051
    readinessProbe:
      grpc:
        port: 50051      # bắt buộc là SỐ — không nhận tên port
      initialDelaySeconds: 5
      periodSeconds: 10
```

## ③ Command (kubectl)

```bash
kubectl apply -f grpc-app.yaml
kubectl get pod grpc-app              # 1/1 Running khi readiness pass
kubectl describe pod grpc-app         # xem kết quả probe + Events
```

Probe fail hiện trong Events với reason `Unhealthy`:

```
Warning  Unhealthy  kubelet  Readiness probe failed: grpc probe failed:
connection refused
```

## ④ Production scenarios

**Giới hạn phải biết trước khi chọn `grpc`** *(bổ sung từ nghiên cứu ngoài
video)*:

| Giới hạn | Chi tiết |
|---|---|
| `port` chỉ nhận số | Không dùng được named port (`http-port`) |
| Không có TLS | kubelet kết nối plaintext — server bắt buộc TLS thì probe fail |
| Không kèm credentials | Không set được header/token; server yêu cầu auth thì probe fail |

Ứng dụng phải TLS hoặc phải auth → dùng lại `exec` với binary
**grpc_health_probe** (công cụ CLI chính thức của cộng đồng Kubernetes,
cài vào image hoặc mount qua init):

```yaml
livenessProbe:
  exec:
    command: ["/bin/grpc_health_probe", "-addr=:50051"]
```

**Khi nào dùng:** service nội bộ chuẩn gRPC (etcd client port 2379 là ví
dụ chính thức trong docs), microservices HTTP/2 — bất kỳ server nào
implement `grpc.health.v1` đều check được, kể cả khi không "gRPC thuần".
Kế thừa codebase cũ đã chạy `exec` + grpc_health_probe → chuyển dần sang
`grpc` để bỏ phụ thuộc binary trong image.

## References

- [Configure Liveness, Readiness and Startup Probes — gRPC](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-grpc-probe)
- [gRPC Health Checking Protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md)
- [grpc_health_probe tool](https://github.com/grpc-ecosystem/grpc-health-probe)
