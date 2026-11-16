# Deep dive — Deployment strategy & rollout

> Cấp độ mục tiêu: **Middle**. Toàn bộ nội dung *(bổ sung từ nghiên cứu
> ngoài video)*; rolling update và rollback cơ bản đã có *(nhắc lại
> Bài 8)*.

## ① Concept

**Hai chiến lược triển khai** (`spec.strategy.type`):

- **RollingUpdate** (mặc định) — thay dần pod cũ bằng pod mới, nhịp
  điều khiển bằng 2 field:
  - `maxUnavailable` — tối đa **thiếu** bao nhiêu pod so với replicas
    (số nguyên hoặc %);
  - `maxSurge` — tối đa **vượt** replicas bao nhiêu — nguồn dự trù
    cho pod mới.
  - Mặc định cả hai **25%**: 4 replicas → thiếu tối đa 1, vượt tối
    đa 1.
- **Recreate** — xóa **hết** pod cũ rồi mới lên pod mới: downtime
  thật, đổi lại không bao giờ chạy 2 phiên bản song song — dùng khi 2
  phiên bản không được cùng ghi một datastore (vd schema mới không
  tương thích ngược).

**Ba field kỷ luật rollout và lịch sử:**

- `revisionHistoryLimit` (mặc định 10) — giữ bao nhiêu ReplicaSet cũ;
  ReplicaSet 0 replicas **chính là** lịch sử rollback *(nhắc lại
  Lab 8.3)*.
- `minReadySeconds` (mặc định 0) — pod phải Ready **liên tục** chừng
  này giây mới tính là available; chặn kịch bản pod Started 2 giây
  rồi chết mà rollout vẫn tiến.
- `progressDeadlineSeconds` (mặc định 600) — rollout quá lâu không
  xong → condition `ProgressDeadlineExceeded`; CI/CD dựa vào đó để
  fail deploy thay vì treo vô hạn.

**change-cause:** chú thích cho từng revision trong `rollout history`
— là annotation `kubernetes.io/change-cause` trên Deployment, được
sao chép vào ReplicaSet **mỗi khi revision sinh ra**; annotate sau
khi update thì revision vừa rồi vẫn `<none>`. `--record` đã
deprecated — dùng annotate.

Blue-green/canary không có sẵn trong Deployment — làm bằng Service/
Ingress *(nhắc lại Bài 13)*.

## ② Configuration (YAML)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  annotations:
    kubernetes.io/change-cause: "nginx 1.27 — ticket 1234"
spec:
  replicas: 6
  revisionHistoryLimit: 5
  minReadySeconds: 30
  progressDeadlineSeconds: 300
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0        # zero-downtime
      maxSurge: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
```

Đổi sang Recreate — chú ý **xóa hẳn** block `rollingUpdate` (để lại
cùng `type: Recreate` là invalid):

```yaml
strategy:
  type: Recreate
```

## ③ Command (kubectl)

```bash
kubectl rollout status deployment/webapp
kubectl rollout history deployment/webapp
kubectl rollout history deployment/webapp --revision=2
kubectl rollout undo deployment/webapp                # về revision kế cuối
kubectl rollout undo deployment/webapp --to-revision=1
kubectl rollout restart deployment/webapp             # sinh revision mới

# đổi nhịp rolling không cần sửa file
kubectl patch deployment webapp -p \
  '{"spec":{"strategy":{"rollingUpdate":{"maxUnavailable":0,"maxSurge":2}}}}'

# ghi change-cause TRƯỚC update để revision mới nhận chú thích
kubectl annotate deployment webapp \
  kubernetes.io/change-cause="nginx 1.27" --overwrite
```

## ④ Production scenarios

**Deploy xong mới chết:** image khởi động ngon 10 giây rồi crash ngay
khi vào lưu lượng — rollout đã "hoàn tất" từ lâu. Hàng thủ: readiness
probe + `minReadySeconds` ≥ thời gian quan sát, và
`progressDeadlineSeconds` ngắn hơn độ kiên nhẫn của pipeline.

**Lịch sử rollback là ReplicaSet:** giảm `revisionHistoryLimit` tiết
kiệm object rác nhưng cắt luôn khả năng undo xa — cân đôi bên; pipeline
nên annotate change-cause với build number/git SHA để `rollout history`
đọc thành "đang chạy bản nào".

**Chọn zero-downtime:** `maxUnavailable: 0` + `maxSurge > 0` cần node
còn dư tài nguyên cho pod vượt; cluster chật → phải chấp nhận thiếu pod
tạm thời (maxUnavailable > 0).

## References

- [Deployment — Strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy)
- [Deployment — Rolling back](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment)
