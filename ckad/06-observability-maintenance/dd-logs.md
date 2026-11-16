# Deep dive — Container Logs

> Cấp độ mục tiêu: **Junior**. Toàn bộ nội dung *(bổ sung từ nghiên cứu
> ngoài video)*; `kubectl logs` cơ bản và `--previous` đã có *(nhắc lại
> Bài 16 Phần xem logs)*.

## ① Concept

Container ghi log ra **stdout/stderr**; container runtime trên node thu
lần và `kubectl logs` đọc qua kubelet. Hệ quả:

- Log **theo từng container** — không có "log của cả pod" trừ khi chỉ định
  rõ; pod nhiều container mà bỏ qua `-c` → kubectl báo lỗi kèm danh sách
  container để chọn.
- Container restart → log hiện tại là của **instance mới**; log lần crash
  trước nằm ở `--previous`.
- `kubectl logs` chỉ đọc log node hiện đang chạy pod đó — pod đã xóa thì
  mất log; gom log nhiều node là việc của cluster-level logging (node
  agent ship về backend — Bài 17 đã nêu khái niệm).

## ② Configuration

Không có YAML — log config nằm ở phía ứng dụng (ghi ra stdout/stderr,
không ghi file trong container) và rotation nằm ở container runtime.

## ③ Command (kubectl)

Bảng tra cứu đầy đủ các flags:

| Flag | Chức năng |
|---|---|
| `-f` / `--follow` | Stream log mới liên tục (Ctrl+C để thoát) |
| `--tail=N` | Chỉ N dòng cuối (mặc định: toàn bộ) |
| `--since=30m` | Log từ 30 phút trước đến nay |
| `--since-time=2026-11-16T00:00:00Z` | Như trên nhưng mốc RFC3339 |
| `--timestamps` | Prefix mỗi dòng timestamp — đối chiếu với events |
| `-c <tên>` / `--container` | Chọn container trong pod nhiều container |
| `-p` / `--previous` | Log của instance đã chết (crash/restart trước) |
| `-l app=web` | Lấy log pod theo label selector |
| `--all-containers` | Mọi container của pod (dùng kèm `-l`) |
| `--prefix` | Prefix `[pod/container]` mỗi dòng |

Lưu ý: `--since` và `--since-time` loại trừ nhau — chỉ dùng một trong hai.

```bash
kubectl logs -f deploy/web                     # follow deployment (pod đầu)
kubectl logs web-abc123 -c sidecar --tail=100  # 100 dòng cuối của sidecar
kubectl logs -l app=web --all-containers \
  --prefix --tail=20                           # mọi pod + container theo label
kubectl logs web-abc123 -p --timestamps        # log instance crashed + mốc giờ
```

## ④ Production scenarios

**Crash Loop:** `kubectl get pod` thấy `CrashLoopBackOff` → lấy log lần
chết gần nhất: `kubectl logs <pod> -p`. Log hiện tại thường chỉ có vài
dòng startup vì container mới lập tức chết.

**Correlate với events:** chạy `kubectl logs <pod> --timestamps` song song
`kubectl get events --sort-by=.metadata.creationTimestamp` — thấy "probe
fail lúc 10:00:03" khớp với "OOMKilled lúc 10:00:02" → hướng nguyên nhân.

**Nhiều replica:** log rải trên nhiều pod → `-l app=web --prefix` gom
một lần; hoặc dựa hệ thống logging tập trung khi có.

## References

- [kubectl logs reference](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#logs)
- [Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
