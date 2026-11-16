# Deep dive — kubectl debug & Ephemeral Containers

> Cấp độ mục tiêu: **Middle → Senior**. Toàn bộ nội dung *(bổ sung từ nghiên
> cứu ngoài video)*; `kubectl exec` đã có *(nhắc lại Bài 16 Phần debugging
> bằng Linux skills)*.

## ① Concept

**Ephemeral container** là container tạm gắn thêm vào pod **đang chạy** để
debug. Ra đời vì hai tình huống mà `kubectl exec` bó tay: image không có
shell (distroless), và container đã crash.

Đặc tính bắt buộc nhớ:

- Gắn vào `pod.spec.ephemeralContainers`; **không thể sửa/xóa sau khi thêm**
  (pod spec bất biến với ephemeral) và **không restart** — thoát là nằm đó.
- **Không có** ports, probes, resources (luôn QoS BestEffort) — đúng nghĩa
  "công cụ thăm dò, không phải workload".
- Chia sẻ **network stack của pod** (cùng IP, cùng localhost).

**Ghi chú phiên bản:** GA **v1.25** (beta v1.23). Đề thi v1.37 → luôn khả dụng.

## ② Configuration (YAML) & ba chế độ

Chế độ 1 — **gắn trực tiếp vào pod đang chạy** (pod gốc giữ nguyên):

```bash
kubectl debug -it mypod --image=busybox:1.28 --target=app
```

`--target=app` bật **process namespace sharing** với container `app`:
thấy process của container đích trong `ps aux`, và đọc filesystem của
chúng qua `/proc/<PID>/root` — vì ephemeral container có filesystem riêng.

Chế độ 2 — **copy pod rồi sửa** (pod gốc không bị động chạm):

```bash
kubectl debug mypod -it --copy-to=mypod-debug \
  --set-image=app=busybox:1.28 --share-processes
```

- `--copy-to` tạo pod mới là bản sao; `--set-image` đổi image container
  chỉ định — cách "thay shell vào pod distroless".
- `--share-processes` set `shareProcessNamespace: true` trên bản sao.
- `--replace` xóa pod gốc sau khi bản sao tạo xong — chỉ dùng kèm
  `--copy-to`, khi muốn pod mang đúng tên cũ.

Chế độ 3 — **debug node**: tạo pod đặc biệt trên node, filesystem của node
mount tại `/host`:

```bash
kubectl debug node/mynode -it --image=busybox:1.28
chroot /host        # vào môi trường thật của node
```

## ③ Command (kubectl)

```bash
kubectl get pod mypod -o jsonpath='{.spec.ephemeralContainers[*].name}'
# xem ephemeral container đã gắn

kubectl describe pod mypod     # ephemeral container hiện ở cuối spec
kubectl delete pod mypod       # cách duy nhất để gỡ ephemeral container
```

## ④ Production scenarios

**Pod distroless không shell** — quy trình 2 bước:
1. `kubectl exec -it mypod -- sh` → fail (`exec: "sh": executable file
   not found`) → xác nhận không debug trực tiếp được.
2. `kubectl debug -it mypod --image=busybox:1.28 --target=app` → `ps aux`
   thấy process của app; `ls /proc/<PID>/root` đọc file của container đích.

**Container crash liên tục (CrashLoopBackOff):** copy pod và đổi command
thành `sleep infinity` — giữ nguyên môi trường mà pod sống đủ lâu để exec
vào kiểm tra: `kubectl debug mypod --copy-to=mypod-dbg --set-image=...`.

**Node không SSH được:** `kubectl debug node/...` + `chroot /host` — đọc
journal, kiểm tra container runtime, xem mount của node không cần SSH key.

**Lưu ý vận hành:** ephemeral container ghi vĩnh viễn vào pod spec — để lại
dấu vết audit (tốt); nhưng Deployment tạo lại pod thì ephemeral mất theo —
không thay thế được logging cố hữu.

## References

- [Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pods/)
- [Ephemeral Containers](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/)
