# Deep dive — ConfigMap & Secret nâng cao

> Cấp độ mục tiêu: **Middle**. Toàn bộ nội dung *(bổ sung từ nghiên cứu
> ngoài video)*; tạo/mount/env từ ConfigMap và Secret đã có *(nhắc lại
> Bài 12)*.

## ① Concept

Ba tính chất nâng cao của ConfigMap/Secret đều xoay quanh **vòng đời dữ
liệu**:

- **immutable** — đánh dấu object **chỉ đọc vĩnh viễn**. Kubelet ngừng
  watch object này → giảm tải đáng kể cho API server và kubelet với
  cluster nhiều ConfigMap/Secret lớn. Muốn thay đổi chỉ có một đường:
  **delete + create lại** object mới.
- **`stringData`** (Secret) — **cổng ghi, không đọc**: khai báo giá trị
  chuỗi thô, API server base64-encode sang `data` khi lưu; mọi lần đọc
  về chỉ thấy `data`. Khi trùng key ở cả hai field, `stringData` thắng
  khi tạo/cập nhật.
- **`type: tls`** (Secret) — schema bắt buộc đúng 2 key `tls.crt` (cert)
  và `tls.key` (private key); dùng cho Ingress TLS và các TLS workload.

**Ghi chú phiên bản:** immutable ConfigMap/Secret GA **v1.21**; `type: tls`
và `stringData` có từ lâu — đề thi v1.37 → luôn khả dụng.

Quy tắc đặt tên key của ConfigMap/Secret: chỉ `[a-zA-Z0-9._-]` — key có
`/` hay khoảng trắng bị từ chối tại admission — vì khi mount thành file,
key trở thành tên file.

## ② Configuration (YAML)

Secret TLS + immutable:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: web-tls
type: tls
stringData:                # ghi chuỗi thô, không cần base64 tay
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    ...
  tls.key: |
    -----BEGIN PRIVATE KEY-----
    ...
# immutable: true          # bật khi dữ liệu đã ổn định
```

ConfigMap immutable:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
immutable: true
```

`immutable` là **cửa một chiều**: bật (false → true) trên object đang
chạy bằng edit/patch vẫn được phép; nhưng đã true thì không tắt lại
được, và mọi thay đổi `data` từ lúc đó bị từ chối.

## ③ Command (kubectl)

```bash
# tạo Secret TLS từ file cert/key
kubectl create secret tls web-tls --cert=tls.crt --key=tls.key

# stringData luôn bị convert thành data base64 trong output
kubectl get secret web-tls -o jsonpath='{.data.tls\.crt}' | base64 -d

# immutable đã bật: sửa data bị từ chối
kubectl edit configmap app-config
# error: ConfigMap "app-config" is invalid: data: Forbidden: field is
# immutable when `immutable` is set
```

Duy nhất một cách thay nội dung:

```bash
kubectl delete configmap app-config && kubectl apply -f app-config.yaml
```

## ④ Production scenarios

**Quy trình rotate cert (Ingress TLS):** Secret `type: tls` mount vào
Ingress — khi cert hết hạn, công cụ tự động như cert-manager update Secret
trực tiếp. Nếu Secret đang `immutable: true` → pipeline phải đi đường
**delete + recreate** với tên mới hoặc cùng tên; Ingress không cần tạo
lại, chỉ reload khi controller phát hiện Secret mới.

**Bẫy immutable + pod:** pod dùng immutable ConfigMap qua **env** không
bao giờ nhận giá trị mới (env set lúc khởi động container — đúng với cả
ConfigMap thường); qua **volume** thì kể cả object thường cũng chỉ được
refresh định kỳ bởi kubelet. immutable chặn luôn cơ chế này → thay đổi =
delete + recreate object **và restart pod**. Đây là đánh đổi có chủ đích:
đổi tính năng "hot reload" lấy hiệu năng cluster.

**Khi nào bật immutable:** dữ liệu cấu hình tĩnh (flag, config ít đổi,
cert dài hạn); cluster > hàng nghìn node; API server đang gánh tải
watch. Secrets chia sẻ rộng rãi (image pull secret, CA bundle) là ứng
viên tốt. Cấu hình đổi hằng ngày → để thường.

Vượt phạm vi CKAD nhưng đáng biết *(bổ sung từ nghiên cứu ngoài video)*:
dòng sản phẩm **SealedSecrets**/**External Secrets Operator** giải bài
toán "lưu Secret an toàn trong Git" — base64 của Secret chỉ là encoding,
**không phải mã hóa hay bảo mật** *(nhắc lại Bài 12)*.

## References

- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [kube-apiserver và immutable objects](https://kubernetes.io/docs/concepts/configuration/secret/#secret-immutable)
