# Deep dive — QoS Classes

> Cấp độ mục tiêu: **Middle**. Toàn bộ nội dung *(bổ sung từ nghiên cứu
> ngoài video)*; requests/limits từng container đã có *(nhắc lại Bài 15
> Phần 5)*.

## ① Concept

**QoS (Quality of Service) class** là nhãn Kubernetes **tự suy ra** từ
cách khai báo requests/limits của pod — không khai báo trực tiếp. Nó quyết
định **pod nào bị hy sinh trước** khi node thiếu memory (eviction), và là
cơ chế "bảo hiểm" cho workload quan trọng.

| Class | Điều kiện (xét MỌI container trong pod) | Vị trí khi node thiếu memory |
|---|---|---|
| **Guaranteed** | requests == limits cho **cả CPU và memory** | Được bảo vệ cao nhất |
| **Burstable** | Có khai báo requests/limits nhưng không thỏa Guaranteed | Đứng giữa — bị đuổi khi dùng vượt requests |
| **BestEffort** | Không khai báo gì | Bị đuổi **đầu tiên** |

Quy tắc phỏng đoán nhanh: set bằng nhau cả hai tài nguyên → Guaranteed;
set một phần → Burstable; bỏ trống → BestEffort. Class là thuộc tính **của
pod**, tính từ tổng thể mọi container.

## ② Configuration (YAML)

```yaml
# Guaranteed — requests và limits KHỚP nhau
resources:
  requests: {cpu: 250m, memory: 256Mi}
  limits:   {cpu: 250m, memory: 256Mi}

# Burstable — có limits nhưng requests thấp hơn
resources:
  requests: {cpu: 100m, memory: 128Mi}
  limits:   {cpu: 500m, memory: 512Mi}

# BestEffort — không có block resources
# (không khai báo gì thêm)
```

Sai một con số (requests 256Mi, limits 257Mi) → rơi từ Guaranteed xuống
Burstable — kiểm tra bằng lệnh ở phần ③, đừng tin mắt.

## ③ Command (kubectl)

```bash
kubectl get pod <pod> -o jsonpath='{.status.qosClass}'
# Guaranteed | Burstable | BestEffort

kubectl get pods -o custom-columns='\
NAME:.metadata.name,QOS:.status.qosClass'
```

## ④ Production scenarios

**Phân biệt OOMKilled và Evicted** — hai cái chết khác nhau hay nhầm:

| | OOMKilled | Evicted |
|---|---|---|
| Ai ra tay | Kernel (OOM killer) khi container chạm **cgroup limit** | Kubelet khi **node** thiếu memory/disk |
| Trạng thái thấy | `RESTARTS` tăng, Reason `OOMKilled` | Pod status `Evicted`, pod biến mất |
| Phục hồi | Container restart tại chỗ (theo restartPolicy) | Pod **không** tự hồi phục — Deployment tạo pod mới |
| Ngăn thế nào | Tăng memory limit / sửa leak | Set requests sát nhu cầu + QoS cao |

**Chọn class theo workload:**
- Database, API gateway thanh toán → **Guaranteed** (requests == limits)
  — không bị giật khi neighbor bùng nổ.
- Batch/CronJob, worker → **Burstable** requests thấp — tận dụng tài
  nguyên dư, chấp nhận rủi ro bị đuổi.
- Tool debug tạm, canary → **BestEffort** — chết trước ai khác là đúng
  thiết kế.

**Đánh đổi Senior:** Guaranteed bắt buộc CPU limits → container bị
**throttle** (CFS) khi chạm limit, kể cả node đang rảnh — tệ cho service
nhạy latency. Nhiều team chọn requests cao + không CPU limits (Burstable)
cho API quan trọng: đánh đổi protection lấy throughput. Không có lựa chọn
miễn phí — hiểu workload rồi chọn *(bổ sung từ nghiên cứu ngoài video)*.

**Không nằm trong đề CKAD**: CPU Manager `static`
policy pin nguyên CPU core cho container Guaranteed dùng CPU nguyên số —
chỉ là điểm neo kiến thức khi đọc tiếp.

## References

- [Pod Quality of Service Classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)
- [Node Pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/) *(cách kubelet xếp hạng pod để evict)*
