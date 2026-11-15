# Labs — Module 5: Application Environment, Configuration and Security

Tổng hợp toàn bộ ví dụ demo và lab exercise từ các bài học trong module, chuyển thành checklist
thực hành kèm tiêu chí verify. Mục tiêu: làm tới khi không cần nhìn tài liệu.

## Bài 12

### Lab 12.1 — Deployment thiếu biến và set env (demo Phần 2)

- [ ] Tạo deployment và quan sát lỗi:

```bash
kubectl create deploy mydb --image=mariadb
kubectl get pods
kubectl describe pod mydb-<suffix>
kubectl logs mydb-<suffix>
```

**Verify:** CrashLoopBackOff, exit code 1; logs cho thấy ứng dụng đòi
variable.

- [ ] Cấp biến: `kubectl set env deploy/mydb MARIADB_ROOT_PASSWORD=password`
- [ ] `kubectl get all`

**Verify:** pod mydb Running.

- [ ] Xuất manifest để tái sử dụng: `kubectl get deploy mydb -o yaml >
      mydb.yaml`; dọn 4 dòng đầu + 2 dòng cuối của `metadata` và cắt từ
      `status:` trở xuống
- [ ] Dọn dẹp: `kubectl delete deploy mydb`

**Ghi nhớ:** MariaDB thiếu `MARIADB_ROOT_PASSWORD` thoát với exit code 1 —
nhận diện qua `describe`, phân biệt với lỗi image/pull.

### Lab 12.2 — ConfigMap biến + prefix (demo Phần 3)

- [ ] Tạo deployment 3 replicas + ConfigMap:

```bash
kubectl create deploy mydb --image=mariadb --replicas=3
kubectl create cm mydbvars --from-literal root_password=password
kubectl get cm mydbvars -o yaml
```

- [ ] Nối vào deployment:
      `kubectl set env deploy/mydb --from=configmap/mydbvars --prefix=mariadb_`
- [ ] `kubectl get deploy mydb -o yaml | grep env -A5`

**Verify:** biến `MARIADB_ROOT_PASSWORD` qua `configMapKeyRef` name
`mydbvars`, key `root_password`; pods Running.

- [ ] Sinh YAML thay vì tạo resource:
      `kubectl create cm mycm --from-literal key=value --dry-run=client -o yaml
      > mycm.yaml`
- [ ] Dọn dẹp: `kubectl delete deploy mydb; kubectl delete cm mydbvars mycm`

**Ghi nhớ:** key trong ConfigMap đặt gì cũng được — `--prefix` ghép tên lúc
dùng và kubectl tự viết hoa.

### Lab 12.3 — ConfigMap làm volume mount vào nginx (demo Phần 4)

- [ ] Tạo config file và ConfigMap:

```bash
echo hello-world > index.html
kubectl create cm myindex --from-file=index.html
kubectl describe cm myindex
```

- [ ] Tạo deployment: `kubectl create deploy myweb --image=nginx`
- [ ] `kubectl edit deploy myweb` — thêm `volumes` (ConfigMap myindex, tên
      nội bộ cmvol) ở `spec.template.spec` + `volumeMounts`
      (`mountPath: /usr/share/nginx/html`) trong container (YAML trong Phần 4)
- [ ] Kiểm tra nội dung mount: `kubectl exec deploy/myweb --
      cat /usr/share/nginx/html/index.html`

**Verify:** lệnh exec trả về `hello-world`.

- [ ] Dọn dẹp: `kubectl delete deploy myweb; kubectl delete cm myindex;
      rm index.html`

**Ghi nhớ:** hai nửa của mount nối nhau bằng **tên volume** (cmvol), không
phải tên ConfigMap.

### Lab 12.4 — Secret generic và Base64 (demo Phần 5–6)

- [ ] Tạo secret và soi giá trị (secret này dùng tiếp Lab 12.5):

```bash
kubectl create secret generic dbpw --from-literal root_password=password
kubectl describe secret dbpw
kubectl get secret dbpw -o yaml
echo <chuỗi-base64> | base64 -d
```

**Verify:** describe chỉ hiện `root_password: 8 bytes`; `-o yaml` lộ chuỗi
Base64, decode ra `password`.

**Ghi nhớ:** encoded ≠ encrypted — `base64 -d` đọc được ngay; `describe` an
toàn hơn `-o yaml`.

### Lab 12.5 — Deployment dùng secret qua set env (demo Phần 6)

Prereq: secret `dbpw` từ Lab 12.4.

- [ ] Nối secret vào deployment:

```bash
kubectl create deploy mynewdb --image=mariadb
kubectl set env deploy/mynewdb --from=secret/dbpw --prefix=mysql_
kubectl get deploy mynewdb -o yaml | grep env -A5
```

**Verify:** biến `MYSQL_ROOT_PASSWORD` qua `secretKeyRef` name `dbpw`, key
`root_password`; pod Running.

- [ ] Dọn dẹp: `kubectl delete deploy mynewdb; kubectl delete secret dbpw`

**Ghi nhớ:** cú pháp giống hệt ConfigMap — chỉ đổi `configmap/` thành
`secret/`.

### Lab 12.6 — Using ConfigMaps and Secrets (end-of-lesson lab)

- [ ] `kubectl create deploy secretlab --image=mariadb`
- [ ] `kubectl create secret generic supersecret --from-literal
      root_password=password`
- [ ] `kubectl set env deploy/secretlab --from=secret/supersecret
      --prefix=mariadb_`
- [ ] `kubectl get all -l app=secretlab`

**Verify:** pod secretlab Running — thiếu `MARIADB_ROOT_PASSWORD` MariaDB
không thể start.

- [ ] Dọn dẹp: `kubectl delete deploy secretlab; kubectl delete secret
      supersecret`

**Ghi nhớ:** pod chạy được tự thân là chứng minh biến đã tới ứng dụng —
không cần exec vào pod kiểm tra.

## Bài 13

### Lab 13.1 — Ingress-based blue-green (demo Phần 2)

Prereq: ingress controller đang chạy (Bài 11); course Git repo đã clone
(Lab 1.0).

- [ ] Chạy bản blue (trong thư mục blue customization của
      `kustomize/blue-green`):

```bash
kubectl apply -k .
kubectl get deploy,pods,svc,cm,ing
```

**Verify:** blue-deploy 1/1 READY; thấy đủ deployment + service + cm +
ingress (dùng danh sách phẩy vì `kubectl get all` thiếu cm/ing).

- [ ] Thêm `myapp.local` vào `/etc/hosts` (trỏ IP ingress); `curl
      myapp.local` → **blue**
- [ ] Tạo bản green (thư mục green customization): `kubectl apply -k .` —
      users vẫn đang ở blue vì chưa có ingress nào trỏ green; test bản mới
- [ ] Switch: `kubectl edit ing myapp` — đổi service name blue-svc →
      green-svc; `curl myapp.local`

**Verify:** curl trả về **green**.

- [ ] Tắt bản cũ: `kubectl scale deploy blue-deploy --replicas=0`
- [ ] Dọn dẹp: delete deployment, service, cm, ingress của cả hai bên

**Ghi nhớ:** bản mới test song song với bản cũ đang chạy; switch chỉ là đổi
service name trong ingress — connections không đứt.

### Lab 13.2 — Service-based blue-green (demo Phần 2)

- [ ] Bản blue với service đặt tên riêng:

```bash
kubectl create deploy blue-nginx --image=nginx:1.14 --replicas=3
kubectl expose deploy blue-nginx --port=80 --name=bg-nginx
```

- [ ] Sinh manifest green: `kubectl get deploy blue-nginx -o yaml >
      green-nginx.yaml`; dọn metadata + status (như Bài 12 Phần 2);
      substitute blue → green toàn file; đổi image version
- [ ] Tạo bản green: `kubectl create -f green-nginx.yaml`

**Verify:** `kubectl get pods` — pods blue-nginx và green-nginx cùng chạy.

- [ ] Switch: `kubectl delete svc bg-nginx; kubectl expose deploy
      green-nginx --port=80 --name=bg-nginx`
- [ ] Dọn dẹp: `kubectl delete deploy blue-nginx` khi green đã ổn; hết lab
      thì delete green-nginx + svc bg-nginx

**Ghi nhớ:** dấu `;` — xóa service cũ và tạo service mới **cùng tên** chạy
sát nhau, downtime tối thiểu; users vẫn dùng đúng tên service cũ.

### Lab 13.3 — Ingress-based canary (demo Phần 3)

Prereq: ingress controller đang chạy (Bài 11); file `canary.yaml` từ course
repo; thêm host vào `/etc/hosts`.

- [ ] Hai ConfigMap:

```bash
echo new-version > index.html
kubectl create cm new-version --from-file=index.html
echo old-version > index.html
kubectl create cm old-version --from-file=index.html
```

- [ ] Bản cũ: `kubectl apply -f canary.yaml`; `kubectl expose deploy
      old-nginx --port=80 --type=NodePort`
- [ ] Bản mới: đổi old → new trong canary.yaml; apply lại; `kubectl expose
      deploy new-nginx --port=80 --type=NodePort`
- [ ] Ingress bản cũ: `kubectl create ingress old-version
      --rule=myapp.info/=old-nginx:80`; `curl myapp.info` → old version
- [ ] Ingress canary: viết `new-ing.yaml` — cùng host myapp.info, trỏ
      new-nginx, annotations `nginx.ingress.kubernetes.io/canary: "true"` +
      `nginx.ingress.kubernetes.io/canary-weight: "10"`; `kubectl apply -f
      new-ing.yaml`
- [ ] `curl myapp.info` liên tục nhiều lần

**Verify:** đa số response là old version, thiểu số new version — không
đúng chính xác 10% (round-robin).

- [ ] Dọn dẹp: delete hai deployment, hai service, hai cm, hai ingress

**Ghi nhớ:** hai Ingress **cùng host** — ingress controller merge hai bộ
rules; annotations canary đặt trên Ingress của **bản mới**.

### Lab 13.4 — Service-based canary (demo Phần 3)

- [ ] Hai ConfigMap:

```bash
echo old-nginx > index.html
kubectl create cm old-version --from-file=index.html
echo new-nginx > index.html
kubectl create cm new-version --from-file=index.html
```

- [ ] Bản cũ: apply `canary.yaml` (phiên bản service-based) — deployment
      old-nginx, 3 replicas, pods mang thêm label `type: canary`
- [ ] Bản mới: đổi old → new trong canary.yaml; apply — new-nginx pods cũng
      mang `type: canary`
- [ ] Service chọn label chung:

```bash
kubectl expose deploy old-nginx --name=theapp --port=80 \
  --selector=type=canary --type=NodePort
kubectl get svc                    # lấy NodePort
curl <minikube-ip>:<nodeport>      # demo: 192.168.49.2:31372
```

**Verify:** response trộn old/new với đa số old (3 pod cũ + 1 pod mới ≈
75/25).

- [ ] Đổi tỷ lệ: `kubectl scale deploy old-nginx --replicas=4` → 80/20
- [ ] Dọn dẹp: delete hai deployment, svc theapp, hai cm

**Ghi nhớ:** service chọn pods qua `--selector=type=canary` — deployment
nào đứng trong lệnh expose không quan trọng; tỷ lệ traffic = tỷ lệ
replicas.

### Lab 13.5 — Deploying Applications the DevOps Way (end-of-lesson lab)

- [ ] Sinh manifest bản cũ + label chung cho cả deployment lẫn pod template:

```bash
kubectl create deploy old-bird --image=nginx:1.18 \
  --dry-run=client -o yaml > old.yaml
# edit old.yaml: thêm type: bird vào metadata.labels (deployment) và
# spec.template.metadata.labels (pod template) — kubectl label không tới
# pod template
kubectl apply -f old.yaml
```

- [ ] Bản mới: đổi old → new toàn file + image nginx:latest; apply lại
- [ ] Service chọn label chung:

```bash
kubectl expose deploy old-bird --name=bird --port=80 \
  --selector=type=bird --type=NodePort
```

**Verify:** `kubectl describe svc bird` — Endpoints có pod của cả old-bird
lẫn new-bird (2 endpoints).

- [ ] Tỷ lệ 90/10: `kubectl scale deploy old-bird --replicas=9`
- [ ] `curl <minikube-ip>:<nodeport>` → Welcome to nginx

**Verify:** `kubectl get pods` — 9 old-bird + 1 new-bird; describe svc bird
— 10 endpoints ("3 endpoints +7 more").

- [ ] Dọn dẹp: `kubectl delete deploy old-bird new-bird; kubectl delete svc
      bird`

**Ghi nhớ:** label phải nằm trên **pod template** — service selector chọn
pods, không chọn deployment; tỷ lệ canary = tỷ lệ replicas (9:1 = 90/10).

## Bài 14

### Lab 14.1 — kubectl config và đổi namespace (demo Phần 1)

- [ ] Xem cấu hình client: `kubectl config view` — nhận diện cluster,
      users, context
- [ ] Đổi default namespace của context hiện tại:

```bash
kubectl config set-context --current --namespace=kube-system
kubectl get pods
```

**Verify:** pods của kube-system hiện ra (default thường rỗng).

- [ ] Quay về: `kubectl config set-context --current --namespace=default`

**Verify:** `kubectl get pods` trả lại như trước lab.

**Ghi nhớ:** cách duy nhất kubectl mặc định cung cấp để đổi current
namespace — luôn set về default khi xong.

### Lab 14.2 — curl API qua kubectl proxy (demo Phần 2)

**Không nằm trong đề CKAD** — lab để hiểu cơ chế, không phải kỹ năng thi.

- [ ] Mở proxy chạy nền: `kubectl proxy --port=8001 &`
- [ ] `curl http://localhost:8001` — xem mọi API paths/groups
- [ ] Tạo pod dummy: `kubectl run proxypod --image=nginx`
- [ ] Thao tác với API objects:

```bash
curl http://localhost:8001/version
curl http://localhost:8001/api/v1/namespaces/default
curl http://localhost:8001/api/v1/namespaces/default/pods/proxypod
curl -X DELETE http://localhost:8001/api/v1/namespaces/default/pods/proxypod
```

**Verify:** lệnh thứ 3 trả JSON đầy đủ properties của proxypod; sau DELETE,
`kubectl get pods` không còn proxypod.

**Ghi nhớ:** `kubectl run <tên> --image=<image>` — không nhận resource type
(khác `kubectl create`); curl không cần TLS vì `kubectl proxy` giữ phần mã
hóa ở đoạn ra mạng.

### Lab 14.3 — Sửa deprecated apiVersion (demo Phần 3)

Prereq: course Git repo (file redis-deploy.yaml).

- [ ] Áp file cũ: `kubectl create -f redis-deploy.yaml`

**Verify:** lỗi "no matches for kind Deployment in version apps/v1beta1".

- [ ] Tra version hiện hành: `kubectl api-versions` — apps/v1
- [ ] Đổi `apiVersion` trong file thành `apps/v1`; `kubectl apply -f
      redis-deploy.yaml`

**Verify:** deployment redis chạy; nếu kêu unknown attributes thì đối
chiếu `kubectl explain deploy.spec --recursive`.

**Ghi nhớ:** lỗi "no matches for kind ... in version ..." = resource đã rời
API group/version đó — thông báo ghi rõ kind và version bị từ chối; sửa
`apiVersion` trước, apply thử sau.

### Lab 14.4 — CRD và custom resource (demo Phần 5)

Prereq: course Git repo (crdobject.yaml, crdbackup.yaml).

- [ ] Tạo CRD: `kubectl apply -f crdobject.yaml`
- [ ] Kiểm tra resource mới:

```bash
kubectl api-resources | grep backup
kubectl get crds
```

**Verify:** thấy resource backups (API group stable.example.com).

- [ ] Tạo custom resource: `kubectl apply -f crdbackup.yaml`
- [ ] Liệt kê: `kubectl get backups`

**Verify:** backup object xuất hiện trong cluster.

- [ ] Dọn dẹp: `kubectl delete backups --all`; `kubectl delete crd
      backups.stable.example.com`

**Ghi nhớ:** CRD chỉ định nghĩa resource — muốn "có tác dụng thật" cần
application tiêu thụ nó (như ingress controller dùng Ingress resources).

### Lab 14.5 — Working With the API (end-of-lesson lab)

- [ ] Tìm mọi resource trên hệ thống đã được định nghĩa bằng custom resource
      definitions:

```bash
kubectl api-resources | grep -i custom
kubectl get crds
```

**Verify:** thấy resource customresourcedefinitions (shortname crd/crds);
`kubectl get crds` liệt kê mọi CRD hiện có trong cluster.

**Ghi nhớ:** `kubectl api-resources` liệt kê resource *types* (kể cả bản
thân CRD); `kubectl get crds` liệt kê *các định nghĩa CRD đang tồn tại* —
lệnh trực tiếp nhất cho câu hỏi này.

## Bài 15

### Lab 15.1 — kubectl auth can-i (demo Phần 1)

- [ ] Kiểm tra quyền của mình và của một user không tồn tại:

```bash
kubectl auth can-i get pods
kubectl auth can-i get pods --as=bob@example.com
```

**Verify:** câu đầu yes; câu sau no — user không tồn tại thì không có
authorization nào diễn ra.

**Ghi nhớ:** `--as=<user>` hoặc `--as=system:serviceaccount:<ns>:<sa>` test
quyền của một danh tính khác.

### Lab 15.2 — default service account và kube-system (demo Phần 2)

- [ ] Pod nào cũng mang service account — xem bất kỳ pod đang chạy:

```bash
kubectl describe pod <pod-bất-kỳ>        # gần đầu: Service Account: default
kubectl get sa                           # default namespace: chỉ có default
kubectl get sa -n kube-system            # nhiều SA của system components
kubectl describe pod coredns-<suffix> -n kube-system
```

**Verify:** pod thường dùng SA default; pod coredns dùng SA coredns.

**Ghi nhớ:** mỗi namespace sinh ra đã có sẵn SA default — đủ cho đa số pod;
CoreDNS cần SA riêng vì phải communicate với pods để cung cấp DNS.

### Lab 15.3 — RBAC: role + service account + role binding (demo Phần 3)

- [ ] Tạo đủ chuỗi RBAC trong namespace riêng:

```bash
kubectl create ns bellview
kubectl create role viewer --verb=get --verb=list --verb=watch \
  --resource=pods -n bellview
kubectl create sa viewer -n bellview
kubectl create rolebinding viewer --serviceaccount=bellview:viewer \
  --role=viewer -n bellview
```

- [ ] Gắn SA vào deployment:

```bash
kubectl create deploy viewginx --image=nginx --replicas=3 -n bellview
kubectl set serviceaccount deploy/viewginx viewer -n bellview
```

- [ ] Verify quyền của SA:

```bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:bellview:viewer -n bellview
kubectl auth can-i get pods \
  --as=system:serviceaccount:bellview:viewer -n default
```

**Verify:** bellview → yes; default → no (role có namespace scope).

- [ ] Dọn dẹp: `kubectl delete ns bellview`

**Ghi nhớ:** tên SA trong rolebinding viết dạng `<namespace>:<tên-sa>` —
được phép dùng SA từ namespace khác; test danh tính SA bằng
`--as=system:serviceaccount:<ns>:<sa>`.

### Lab 15.4 — ClusterRole/ClusterRoleBinding qua CoreDNS (demo Phần 3)

- [ ] Soi chuỗi quyền của CoreDNS:

```bash
kubectl describe sa coredns -n kube-system
kubectl describe clusterrolebinding system:coredns
kubectl describe clusterrole system:coredns
```

**Verify:** rolebinding ghi RoleRef kind ClusterRole + Subjects là SA
coredns; clusterrole liệt kê verbs trong PolicyRules.

**Ghi nhớ:** SA tự thân gần như trống — quyền nằm ở (cluster) role binding
nối nó với role; hệ thống cần access toàn cluster thì dùng ClusterRole +
ClusterRoleBinding.

### Lab 15.5 — Security context: runAsUser/runAsGroup/fsGroup (demo Phần 4)

Prereq: file security-context-demo-2.yaml (course Git repo / K8s docs).

- [ ] Apply pod có securityContext ở cả pod lẫn container level, rồi vào
      shell pod:

```bash
kubectl apply -f security-context-demo-2.yaml
kubectl exec -it security-context-demo -- sh
```

- [ ] Trong shell của pod:

```bash
ps                                   # processes chạy với user ID 1000
cd /data && ls -l                    # demo: owner root, group 2000
cd demo && echo hello > testfile && ls -l
```

**Verify:** process của user 1000; file mới owned user 1000 + group 2000
(fsGroup) — không phải primary group 3000 (runAsGroup).

**Ghi nhớ:** fsGroup đặt group mặc định cho file mới tạo trong container;
`allowPrivilegeEscalation: false` cấm sudo từ trong container.

- [ ] Dọn dẹp: `kubectl delete -f security-context-demo-2.yaml`

### Lab 15.6 — runAsNonRoot xung đột với image chạy root (demo Phần 4)

- [ ] Apply pod nginx-secure với `runAsNonRoot: true`:

```bash
kubectl apply -f security-context-demo.yaml
kubectl get pods
kubectl describe pod nginx-secure
```

**Verify:** status CreateContainerConfigError; event báo container có
runAsNonRoot nhưng image chạy root — xung đột.

**Ghi nhớ:** pod fail vì security context có thể hiện failing hoặc pending
— `kubectl describe` lấy events.

- [ ] Dọn dẹp: `kubectl delete -f security-context-demo.yaml`

### Lab 15.7 — Requests/limits: OOMKilled → Pending → chạy (demo Phần 5)

Prereq: file frontend-resources.yaml (course Git repo) — pod 2 containers:
mysql (request 64Mi/250m, limit 128Mi/500m) + wordpress tương tự.

- [ ] Chạy bản gốc:

```bash
kubectl apply -f frontend-resources.yaml
kubectl get pods
kubectl describe pod frontend
```

**Verify:** 1/2 container chạy; events cho thấy db container OOMKilled —
limit 128MB không đủ.

- [ ] Sửa request của db thành 64Gi, apply lại

**Verify:** pod Pending; events FailedScheduling — insufficient memory.

- [ ] Sửa về request 64Mi, limit 512Mi, apply lại

**Verify:** pod Running.

- [ ] Dọn dẹp: `kubectl delete -f frontend-resources.yaml`

**Ghi nhớ:** OOMKilled = ứng dụng cần nhiều hơn limit; Pending +
FailedScheduling = scheduler không tìm được node đủ request cho pod.

### Lab 15.8 — kubectl set resources trên deployment (demo Phần 5)

- [ ] Set requests + limits trong một lệnh:

```bash
kubectl set resources deploy/<tên> --requests memory=64Mi,cpu=250m \
  --limits memory=512Mi,cpu=500m
```

**Verify:** `kubectl get deploy <tên> -o yaml | grep -A4 resources:` thấy
cả requests và limits.

**Ghi nhớ:** mẹo thi — cần set trên pod riêng lẻ: tạo deployment, `kubectl
set`, rồi copy spec từ deployment YAML sang pod YAML.

### Lab 15.9 — Security (end-of-lesson lab)

- [ ] Namespace `secure` + SA `secure` + deployment `secure-deploy` dùng
      SA đó:

```bash
kubectl create ns secure
kubectl create sa secure -n secure
kubectl create deploy secure-deploy --image=nginx -n secure
kubectl set serviceaccount deploy/secure-deploy secure -n secure
kubectl describe -n secure pod secure-<suffix>
```

**Verify:** describe pod hiện `Service Account: secure`.

**Ghi nhớ:** chỉ cần gắn đúng SA cho deployment — không phải tạo role/
rolebinding (RBAC không thuộc đề CKAD).

- [ ] Dọn dẹp: `kubectl delete ns secure`
