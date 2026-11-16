# Deep dive — Ingress nâng cao

> Cấp độ mục tiêu: **Middle**. Toàn bộ nội dung *(bổ sung từ nghiên cứu
> ngoài video)*; Ingress rule, pathType và cài controller đã có *(nhắc
> lại Bài 11)*.

## ① Concept

- **ingressClassName** — chọn controller nào xử lý Ingress này khi
  cluster chạy nhiều controller (thay annotation
  `kubernetes.io/ingress.class` cũ; field có từ v1.18). Quên set (và
  cluster không có IngressClass mặc định) → Ingress tạo OK nhưng
  **không controller nào nhận** — triệu chứng trùng Lab 11.6.
- **defaultBackend** — Service nhận mọi request không khớp **bất kỳ**
  rule nào (host lẫn path); không khai báo thì controller trả lỗi mặc
  định của nó. Thường trỏ về một pod 404 riêng.
- **TLS termination** — block `tls` khai `hosts` + `secretName` trỏ
  Secret `type: tls` *(nhắc lại Lab DD.7 — Module 5)*. Controller
  **terminate** TLS: client ↔ controller chạy HTTPS, controller ↔ pod
  chạy HTTP thường. Cert phải khớp host — sai host ứng dụng vẫn chạy,
  chỉ client cảnh báo.

## ② Configuration (YAML)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: error-page
      port:
        number: 80
  tls:
  - hosts:
    - shop.example.com
    secretName: shop-tls           # Secret type: tls
  rules:
  - host: shop.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: shop-web
            port:
              number: 80
```

Một Ingress khai được nhiều host — mỗi block `tls` gắn một Secret cho
nhóm hosts của nó.

## ③ Command (kubectl)

```bash
# sinh Ingress TLS bằng một lệnh duy nhất
kubectl create ingress shop --class=nginx \
  --rule="shop.example.com/=shop-web:80,tls=shop-tls"

kubectl get ing shop               # PORTS: 80, 443 khi có TLS
kubectl describe ing shop          # mục TLS + backends kèm pod IPs

# đọc hạn cert trực tiếp từ Secret (không cần vào controller)
kubectl get secret shop-tls -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | openssl x509 -noout -dates
```

## ④ Production scenarios

**Sự cố cert hết hạn:** browser báo `ERR_CERT_DATE_INVALID` nhưng
`kubectl get ing` vẫn "ổn" — Ingress không biết cert trong Secret còn
hạn hay hết. Chẩn đoán bằng lệnh decode Secret ở mục ③; sửa = tạo lại
Secret **cùng tên** với cert mới, controller watch Secret và reload —
Ingress giữ nguyên.

**Đổi host quên cert:** thêm rule host mới mà quên đưa host vào
`tls.hosts` → controller phục vụ host đó bằng cert khác, client cảnh
báo. Checklist: mỗi host trong `rules` phải có mặt trong một block
`tls`.

**defaultBackend 404 "thương hiệu":** trỏ về pod trả trang 404 riêng
thay vì lỗi thô của controller — đồng thời là dấu chứng request đã
tới controller (phân biệt với lỗi DNS/network từ phía client).

## References

- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Ingress — TLS](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls)
