# Deep dive — ResourceQuota & LimitRange

> Cấp độ mục tiêu: **Middle → Senior**. Toàn bộ nội dung *(bổ sung từ nghiên
> cứu ngoài video)*; requests/limits từng container đã có *(nhắc lại Bài 15
> Phần 5)*. Course gộp quota vào nhóm không thuộc đề; checklist CNCF (mục
> 4.3 của file `04-...md`) vẫn nêu ResourceQuota/LimitRange → học sâu.

## ① Concept

Hai resource quản lý tài nguyên ở **cấp namespace**, bổ trợ cho
requests/limits ở cấp container:

- **ResourceQuota** — giới hạn **tổng** namespace được dùng: tổng
  requests/limits của mọi pod, số lượng pod, và số lượng object
  (ConfigMap, Secret, Service, PVC...). Trả lời "namespace này được phép
  tiêu bao nhiêu?".
- **LimitRange** — chính sách cho **từng container/pod**: giá trị default
  được inject khi không khai báo, và min/max mà container được phép set.
  Trả lời "mỗi container trong namespace này phải xin dạng nào?".

**Cơ chế tương tác quan trọng nhất:** namespace có ResourceQuota về
compute (requests/limits) → pod **không khai báo resources sẽ bị từ chối
ngay khi tạo** (admission controller không có căn cứ để trừ quota). Đây
chính là lý do hai resource đi cặp: LimitRange cung cấp default để pod
"trần" vẫn qua được quota.

## ② Configuration (YAML)

ResourceQuota — giới hạn tổng:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "1"        # tổng requests mọi pod
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    pods: "10"               # object counts
    services: "5"
    configmaps: "20"
    persistentvolumeclaims: "5"
```

LimitRange — chính sách từng container:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: team-limits
  namespace: team-a
spec:
  limits:
  - type: Container
    default:                 # inject làm LIMIT nếu thiếu
      cpu: 500m
      memory: 512Mi
    defaultRequest:          # inject làm REQUEST nếu thiếu
      cpu: 250m
      memory: 256Mi
    min:
      cpu: 50m
    max:
      cpu: "1"
      memory: 1Gi
```

Hai hành vi vi phạm khác nhau:

| Tiếp xúc | Hành vi khi không khai báo | Hành vi khi vi phạm min/max |
|---|---|---|
| ResourceQuota (compute) | Pod bị **từ chối** (`must specify ...`) | Vượt tổng → **từ chối** (`exceeded quota`) |
| LimitRange | **Inject** default/defaultRequest | Ngoài min/max → **từ chối** tại admission |

## ③ Command (kubectl)

```bash
kubectl create namespace team-a
kubectl apply -f quota.yaml -f limitrange.yaml

kubectl run bare --image=nginx -n team-a   # trước khi có LimitRange:
# Error from server (Forbidden): pods "bare" is forbidden:
# ... must specify requests.cpu ...

kubectl describe quota -n team-a            # bảng Used vs Hard
kubectl describe limitrange -n team-a
```

Vượt tổng quota → lỗi dạng:

```
Error from server (Forbidden): pods "x" is forbidden: exceeded quota:
team-quota, requested: requests.cpu=500m, used: requests.cpu=800m,
limited: requests.cpu=1
```

Lệnh imperative nhanh: `kubectl create quota team-quota -n team-a
--hard=cpu=2,memory=2Gi,pods=10`.

## ④ Production scenarios

**Phân tích Senior — 3 lớp phòng thủ cho namespace multi-tenant:**
1. **LimitRange** đặt sàn/trần từng container + cấp default → chống một
   container nuốt trọn namespace, và cứu pod không khai báo.
2. **ResourceQuota** chặn tổng → team không chiếm hết cluster, capacity
   planning có căn cứ (quota = ngân sách).
3. **QoS** (suy ra từ requests/limits) quyết định pod nào bị eviction
   trước khi node thiếu memory → xem [dd-qos-classes](dd-qos-classes.md).

Đánh đổi: quota đặt sát → dev bị chặn giữa đêm khi deploy; đặt lỏng → mất
ý nghĩa. Thói quen tốt: quota theo budget của team, LimitRange theo profile
workload (web 250m/512Mi, job 100m/256Mi).

Pod ở trạng thái terminal (Succeeded/Failed) không còn trừ vào quota pod
count — quan trọng với namespace chạy nhiều Job *(bổ sung từ nghiên cứu
ngoài video)*.

## References

- [Resource Quota](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)
