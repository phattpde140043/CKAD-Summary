# Bài 12 — ConfigMaps and Secrets

Bài đầu của Module 5 — miền **Application Environment, Configuration and
Security** (25% đề thi). Chủ đề xuyên suốt: **decouple** thông tin đặc thù
từng site (site-specific) khỏi application code chung — ConfigMaps cho biến
và configuration file, Secrets cho dữ liệu nhạy cảm, kết thúc bằng Docker
registry access secrets.

## Lộ trình bài học

1. Why decoupling is important
2. Providing variables to Kubernetes applications
3. Providing variables with ConfigMaps
4. Providing configuration files using ConfigMaps
5. Secrets
6. Configuring applications to use secrets
7. Secrets and authenticated registry access
8. End-of-lesson lab

## Phần 1 — Why decoupling is important

- Tình huống: một ứng dụng chạy ở site 1, site 2, site 3 — mỗi site cần ứng
  dụng **hành xử khác nhau**. Ba option:

| Option | Cách làm | Đánh giá |
|---|---|---|
| **1 — Bake vào image** | Đặt biến `key=value` ngay trong image | Mỗi site cần một image riêng — không flexible |
| **2 — Định nghĩa trong YAML** | Biến nằm trong `myapp.yaml` | Dùng chung một image, nhưng mỗi site một YAML — YAML đó chỉ chạy đúng một site |
| **3 — ConfigMap** | YAML ứng dụng nối tới resource **ConfigMap** (CM) riêng | Mỗi site một ConfigMap, application code giữ nguyên |

- Option 3 là cách đạt **true decoupling**: code ứng dụng chung, từng site chỉ
  việc lấy ConfigMap áp dụng cho mình.

## Phần 2 — Providing variables to Kubernetes applications

- Truyền biến lúc start ứng dụng không phù hợp môi trường fully automated —
  không ai đứng đó pass biến; **configuration as code** đòi hỏi biến nằm sẵn
  trong configuration files.
- `kubectl create deploy` **không có option truyền biến**. Pattern chuẩn: tạo
  deployment trước, rồi `kubectl set env` đặt biến sau.
- Standalone pod thì `kubectl run` nhận `--env key=value` — nhưng lẽ ra không
  nên chạy standalone pod.

> **Ghi nhớ thi CKAD**: muốn thêm biến vào deployment: `kubectl set env
> deploy/<tên> KEY=value` — `kubectl create deploy` không nhận biến trực tiếp.

### Demo — mydb thiếu biến

```bash
kubectl create deploy mydb --image=mariadb
kubectl get pods                     # pod vào error state
kubectl describe pod mydb-<suffix>   # CrashLoopBackOff, exit code 1
kubectl logs mydb-<suffix>           # ứng dụng đòi variable
kubectl set env deploy/mydb MARIADB_ROOT_PASSWORD=password
kubectl get all                      # ứng dụng chạy
```

- MariaDB khởi động thiếu `MARIADB_ROOT_PASSWORD` nên container thoát ngay —
  last state `Terminated`, reason `Error`, exit code 1 → CrashLoopBackOff
  *(nhắc lại Bài 5 Phần 7 — pod troubleshooting)*.

Xuất manifest để tái sử dụng:

```bash
kubectl get deploy mydb -o yaml > mydb.yaml
```

- Dọn file trước khi dùng: trong `metadata` bỏ 4 dòng đầu + 2 dòng cuối —
  runtime info tự thêm (creationTimestamp, generation, resourceVersion,
  uid…); cắt từ `status` trở xuống vì đó là trạng thái hiện tại của ứng dụng
  trong cluster, không thuộc định nghĩa resource.

## Phần 3 — Providing variables with ConfigMaps

- **ConfigMap** là API resource lưu **site-specific information** — cách
  Kubernetes thay cho configuration file trên môi trường Linux standalone.
- Hai công dụng: lưu **variables** và lưu **configuration files**, tổng tối
  đa **1 MB**. Cần nhiều hơn thì lưu trong pod volume và mount vào pod
  *(nhắc lại Bài 7 Phần 2 — pod volume storage)*.

### Tạo ConfigMap

```bash
kubectl create cm mycm --from-literal key=value        # gán trực tiếp một biến
kubectl create cm mycm --from-env-file=envfile         # file nhiều biến, mỗi dòng một biến
kubectl create cm mycm --from-literal key=value \
  --dry-run=client -o yaml > mycm.yaml                 # sinh YAML thay vì tạo resource
```

- Đưa vào ứng dụng: `kubectl set env deploy/mydeploy --from=configmap/mycm`.
- `--prefix` ghép tiền tố vào tên biến lúc dùng: ConfigMap chỉ cần chứa
  `root_password`, chạy với `--prefix=mariadb_` tạo thành
  `MARIADB_ROOT_PASSWORD` (kubectl tự viết hoa) — định nghĩa một phần tên biến
  trong ConfigMap, phần còn lại bổ sung khi dùng.

Cấu trúc field trong pod manifest:

| Field | Ý nghĩa |
|---|---|
| `pod.spec.containers.env.name` | Tên biến |
| `pod.spec.containers.env.valueFrom.configMapKeyRef.name` | Tên ConfigMap |
| `pod.spec.containers.env.valueFrom.configMapKeyRef.key` | Key bên trong ConfigMap |

### Demo — ConfigMap mydbvars

```bash
kubectl create deploy mydb --image=mariadb --replicas=3
kubectl create cm mydbvars --from-literal root_password=password
kubectl get cm mydbvars -o yaml    # data: root_password: password
kubectl set env deploy/mydb --from=configmap/mydbvars --prefix=mariadb_
kubectl get deploy mydb -o yaml | grep env -A5
```

- `kubectl get cm mydbvars -o yaml`: section `data` chứa key `root_password`
  với value `password` — gọi là **key** vì chính là thứ `configMapKeyRef.key`
  sẽ trỏ tới sau này.
- `grep env -A5` cho thấy trong container: biến `MARIADB_ROOT_PASSWORD`, mục
  `valueFrom` → `configMapKeyRef` với `name: mydbvars` (ConfigMap nào) và
  `key: root_password` (key nào).
- Mấu chốt: **key trong ConfigMap đặt là gì cũng được** — điều quan trọng là
  tên biến; `configMapKeyRef` chỉ mô tả cách lấy value cho biến từ key của
  ConfigMap nào.

*(Transcript ghi "kubectl setenv" và typo demo `--from-ConfigMap` — chuẩn hóa
thành `kubectl set env` và `--from=configmap/...`.)*

## Phần 4 — Providing configuration files using ConfigMaps

- Trong section `data` của ConfigMap, mỗi **configuration file** được đại diện
  bởi một key riêng; pod trỏ tới key đó để lấy value — thủ tục giống hệt như
  với biến.
- Khác biệt: muốn dùng configuration file, ConfigMap phải gắn như **pod
  volume** rồi mount vào một thư mục trong container. ConfigMap volume đứng
  cạnh các volume trỏ tới storage thật (hostPath, NFS, Fibre Channel…) — cách
  khai báo hoàn toàn giống nhau.

### Demo — myindex mount vào nginx

```bash
echo hello-world > index.html                      # tạo file cấu hình trên máy
kubectl create cm myindex --from-file=index.html   # đưa file vào cluster
kubectl describe cm myindex                        # key: index.html + value
kubectl create deploy myweb --image=nginx
kubectl edit deploy myweb
```

- ConfigMap chứa được **nhiều file**, thậm chí cả thư mục — linh hoạt tùy ý.
- File cấu hình đang nằm trên máy cá nhân; tinh thần Kubernetes là đưa nó
  lên cluster (qua ConfigMap) thay vì giữ ở local.
- Sửa deployment: volume nằm ở `spec.template.spec` — deployment spec →
  pod template nhúng trong đó → pod spec — ngang cấp với `containers`:

```yaml
      volumes:
      - name: cmvol
        configMap:
          name: myindex
```

- Trong thuộc tính container, thêm `volumeMounts`:

```yaml
        volumeMounts:
        - mountPath: /usr/share/nginx/html   # document root của nginx
          name: cmvol                        # tên volume nội bộ
```

- Ghi chú thụt lề của giảng viên: lề con của `volumes` không cần khớp lề con
  của `containers` — chỉ cần các con của **cùng một parent** giữ vặn lề với
  nhau.
- Hai cái tên: volume trỏ tới ConfigMap **myindex** (tên ngoài), rồi đặt tên
  nội bộ cho volume là **cmvol** và mount `cmvol` — tách tên trong khỏi tên
  ngoài, ConfigMap có thay đổi thì không phải sửa khắp YAML.

> **Ghi nhớ thi CKAD**: mount ConfigMap luôn cần **hai nửa**: `volumes` ở pod
> spec (trỏ ConfigMap) + `volumeMounts` trong container; hai nửa nối nhau bằng
> tên volume, không phải tên ConfigMap. Tài liệu ConfigMap trên exam khá dài
> (gồm cả biến lẫn volume) — đọc trước để biết mục nào nằm ở đâu.

*(Transcript ghi "echo hello-world-great-then-index.html" — chuẩn hóa thành
`echo hello-world > index.html`.)*

## Phần 5 — Secrets

- **Secret** là bản **Base64 encoded** thay thế ConfigMap: nội dung không đọc
  trực tiếp được, nhưng **encoded ≠ encrypted** — decode rất dễ. "A secret is
  not really secret."
- Ba type cho ba tình huống:

| Type | Dùng cho |
|---|---|
| `generic` | Giá trị nhạy cảm thông thường (password) — làm được mọi thứ ConfigMap làm được |
| `tls` | Lưu TLS keys/certificate |
| `docker-registry` | Credentials truy cập registry — cơ chế chuẩn, dùng được với cả registry ngoài Docker |

- Secret giúp Kubernetes an toàn hơn một bậc vì value không phải nằm trong
  application manifest (ConfigMap cũng có tính chất này) — nhưng không hoàn
  toàn bảo mật vì chỉ encoded.
- Kubernetes nội bộ dùng secret rất nhiều (credentials hệ thống); `kubectl
  describe pod` cho thấy **mọi pod đều dùng ít nhất một secret**.

## Phần 6 — Configuring applications to use secrets

- Secret = ConfigMap dạng encoded → cách dùng giống hệt: chứa **biến** thì
  `kubectl set env`; chứa **file** thì mount vào pod.
- Khác duy nhất so với `kubectl create cm`: phải ghi rõ **type** ngay trong
  lệnh. Bốn kịch bản:

```bash
# TLS keys
kubectl create secret tls mytls --cert=<cert-file> --key=<key-file>
# password (generic)
kubectl create secret generic dbpw --from-literal root_password=password
# SSH private key (generic)
kubectl create secret generic mykey --from-file=ssh-privatekey=<file>
# file nhạy cảm, mount root-only (generic)
kubectl create secret generic my-secret-file --from-file=<file>
```

- Khi mount secret nên để ý `volumes.secret.defaultMode: 0400` — chỉ owner
  của file có quyền đọc.
- Secret đã mount **tự cập nhật** trong ứng dụng khi secret thay đổi — theo
  giảng viên là điểm ConfigMap không có, và cũng là lý do nên ưu tiên secret
  khi cần file content.

*(bổ sung từ nghiên cứu ngoài video)*: thực tế volume mount của ConfigMap cũng
được kubelet sync lại theo chu kỳ; thứ **không bao giờ** tự cập nhật là biến
môi trường — dù lấy từ ConfigMap hay Secret.

### Demo — secret dbpw và Base64

```bash
kubectl create secret generic dbpw --from-literal root_password=password
kubectl describe secret dbpw     # data: root_password — chỉ hiện "8 bytes"
kubectl get secret dbpw -o yaml  # data: root_password: cGFzc3dvcmQ=
echo cGFzc3dvcmQ= | base64 -d    # password
```

- `describe` chỉ hiện **kích thước** value (8 bytes) — không lộ nội dung;
  `get -o yaml` mới lộ chuỗi Base64, và `base64 -d` decode ra clear text
  ngay.
- Vì sao "không tệ như trông thấy": (1) secret nằm trong **etcd** — user
  thường không có quyền đọc etcd; (2) manifest ứng dụng chỉ chứa con trỏ tới
  secret, không chứa value; đọc được secret đòi quyền read trên secrets —
  thường chỉ kube admin có; môi trường thật đa số user bị giới hạn quyền.

### Demo — mynewdb dùng secret

```bash
kubectl create deploy mynewdb --image=mariadb
kubectl set env deploy/mynewdb --from=secret/dbpw --prefix=mysql_
kubectl get deploy mynewdb -o yaml | grep env -A5
```

- Output: biến `MYSQL_ROOT_PASSWORD` lấy value qua `secretKeyRef` name
  `dbpw`, key `root_password` — cấu trúc giống hệt `configMapKeyRef` ở
  Phần 3.

*(Transcript nói "root-password", "my-new-db", "maria-db" — chuẩn hóa thành
`root_password`, `mynewdb`, `mariadb`; prefix `mysql_` + key `root_password`
mới cho ra biến hợp lệ `MYSQL_ROOT_PASSWORD`.)*

## Phần 7 — Secrets and authenticated registry access

- Nhiều container registry đòi **authenticated access**; kể cả registry miễn
  phí như Docker Hub, authenticate rồi cũng có **nhiều quyền hơn**.
- Trên command line thì dễ: `docker login` + username/password. Trong
  Kubernetes không tiện vậy → dùng secret: lưu registry credentials vào secret
  type **`docker-registry`**.
- Hai cách tạo:
  1. Ghi đủ các giá trị ngay trên command line.
  2. Import từ file `config.json` do `docker login` / `podman login` sinh ra.
- Tạo secret xong phải **chỉ cho pod dùng** — qua `pod.spec.imagePullSecrets`.

### Demo — dockercreds và pod gettools

**Bước 1 — tạo secret** (username/password demo là fake — mục đích chỉ để hiểu
cơ chế):

```bash
kubectl create secret docker-registry dockercreds \
  --docker-username=<username> --docker-password=<password> \
  --docker-email=<email> --docker-server=<server>
kubectl get secret dockercreds -o yaml
```

- Data section có key `.dockerconfigjson` với chuỗi Base64 dài — `base64 -d`
  decode ra JSON chứa toàn bộ các giá trị vừa nhập.

**Bước 2 — sinh manifest pod dùng image private:**

```bash
kubectl run gettools --image=docker.io/sandervanvugt/gettools \
  --dry-run=client -o yaml > gettools.yaml
```

**Bước 3 — edit `gettools.yaml`**, trong pod spec thêm secret:

```yaml
  imagePullSecrets:
  - name: dockercreds
```

**Bước 4 — apply và gỡ lỗi:**

```bash
kubectl apply -f gettools.yaml
# error: ... cannot unmarshal object into Go struct field
# Pod.spec.imagePullSecrets of type []core.LocalObjectReference
kubectl explain pod.spec.imagepullsecrets
```

- `explain` cho thấy field type đứng trong **ngoặc vuông** — ký hiệu
  **list**: "an optional list of references to secrets in the same namespace".
- YAML đầu tiên viết `imagePullSecrets:` + `name:` thụt vào — đó là map,
  không phải list. Trong list **mỗi item bắt đầu bằng hyphen** — như
  `containers`, mỗi container một `-`. Thêm `-` trước `name: dockercreds`
  → `kubectl apply -f gettools.yaml` chạy được.

> **Ghi nhớ thi CKAD**: `imagePullSecrets` là **list** — mỗi secret một dòng
> `- name: <secret>`. Gặp lỗi apply "cannot unmarshal ... into ..." → mở
> `kubectl explain` field đó: ngoặc vuông `[]` nghĩa là chờ list.

*(Transcript ghi "imagepool/pipeless secrets", "Docker config.json" và
"PubMed login" — chuẩn hóa thành `imagePullSecrets`, `.dockerconfigjson` và
`podman login`.)*

## Phần 8 — End-of-lesson lab — Using ConfigMaps and Secrets

### Đề bài

Tạo deployment `secretlab` chạy image MariaDB. Tạo secret `supersecret` đặt
biến `root_password` thành giá trị `password`. Đảm bảo deployment `secretlab`
dùng secret đó để cung cấp biến `MARIADB_ROOT_PASSWORD` cho ứng dụng.

### Lời giải

```bash
kubectl create deploy secretlab --image=mariadb
kubectl create secret generic supersecret --from-literal root_password=password
kubectl set env --help | less        # tra lại --prefix nếu quên
kubectl set env deploy/secretlab --from=secret/supersecret --prefix=mariadb_
kubectl get all -l app=secretlab
```

- Pod **Running** là bằng chứng đủ — thiếu `MARIADB_ROOT_PASSWORD` thì
  MariaDB không thể start *(như demo Phần 2)*.
- Bẫy nhỏ trong demo: quên tên secret ngay sau `generic` — lệnh báo lỗi ngay
  và gợi ý; chèn `supersecret` vào là xong.

## Tổng hợp lệnh Bài 12

| Lệnh | Chức năng |
|---|---|
| `kubectl set env deploy/<tên> KEY=value` | Đặt biến vào deployment (create deploy không nhận biến) |
| `kubectl create cm <tên> --from-literal key=value` / `--from-env-file=<file>` | Tạo ConfigMap từ biến / file biến |
| `kubectl create cm <tên> --from-file=<file>` | Tạo ConfigMap chứa configuration file |
| `kubectl set env deploy/<tên> --from=configmap/<cm> [--prefix=tiền_tố_]` | Nối biến từ ConfigMap vào deployment |
| `kubectl create secret generic <tên> --from-literal key=value` | Tạo secret cho password v.v. — nhớ ghi type |
| `kubectl create secret tls <tên> --cert=<cert> --key=<key>` | Tạo secret chứa TLS keys |
| `kubectl create secret docker-registry <tên> --docker-username=... ...` | Tạo secret registry access |
| `kubectl set env deploy/<tên> --from=secret/<secret> [--prefix=tiền_tố_]` | Nối biến từ secret vào deployment |
| `kubectl get secret <tên> -o yaml` + `echo <base64> \| base64 -d` | Xem và decode giá trị secret |
| `kubectl explain pod.spec.imagepullsecrets` | Tra type field khi YAML apply lỗi |

> ConfigMaps và Secrets khép lại phần **cấu hình**: ứng dụng giờ nhận được
> biến, configuration file và credentials mà không phải sửa image hay nhét
> giá trị vào manifest. Bài 13 — **Deploying Applications the DevOps Way** —
> chuyển sang cách triển khai ứng dụng bằng GitOps thay lệnh chạy tay.
