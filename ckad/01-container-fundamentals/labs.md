# Labs — Module 1: Container Fundamentals

Tổng hợp toàn bộ ví dụ demo và lab exercise từ các bài học trong module, chuyển thành checklist
thực hành kèm tiêu chí verify. Mục tiêu: làm tới khi không cần nhìn tài liệu.

## Bài 1 — Understanding and Using Containers

### Lab 1.0 — Chuẩn bị môi trường

*(Demo đầu Bài 1; chi tiết đầy đủ lab environment ở Bài 4)*

- [ ] `sudo apt install git` — có thể đã cài sẵn
- [ ] `git clone https://github.com/sandervanvugt/ckad.git` — nếu bị hỏi username/password nghĩa là gõ sai URL (thường sai ở phần `sandervanvugt`)
- [ ] Vào thư mục `ckad`, chạy `./minikube-docker-setup.sh` — cài minikube, kèm luôn docker
- [ ] `sudo reboot`
- [ ] `sudo systemctl status docker` — **Verify:** trạng thái `active (running)`

### Lab 1.1 — Chạy container nginx

- [ ] `docker run nginx` — fetch image từ Docker Hub về rồi chạy foreground; Ctrl+C thoát vì output tràn màn hình
- [ ] `docker run -d nginx` — chạy detached mode
- [ ] `docker ps` — **Verify:** thấy container nginx với đủ các cột CONTAINER ID, IMAGE, COMMAND (entry point), CREATED/STATUS, NAMES (tên tự sinh)

### Lab 1.2 — Hiện tượng container thoát ngay (busybox)

- [ ] `docker run busybox`
- [ ] `docker ps` — **Verify:** không thấy busybox (entry point là shell, không có standard out → thoát ngay)
- [ ] `docker ps -a` — **Verify:** busybox ở trạng thái `Exited (0)` — exit code 0 = lệnh mặc định kết thúc "bình thường"
- [ ] `docker run -it busybox` — **Verify:** prompt đổi thành hash (đây là shell của container)
- [ ] `whoami` — **Verify:** `root`
- [ ] Quan sát: file system bên trong là root fs của **container image**, không phải của host OS
- [ ] `exit` — thoát khỏi shell container

### Lab 1.3 — Khởi chạy theo engine khác (tùy chọn)

- [ ] `podman run docker.io/library/nginx` — podman bắt buộc dùng FQCN
- [ ] `kubectl run mynginx --image=docker.io/library/nginx` — chạy pod trên minikube *(nhìn trước — chi tiết ở module sau)*

### Lab 1.4 — Quản lý vòng đời container

- [ ] `docker run --name web nginx` — quên `-d` → chạy foreground → Ctrl+C
- [ ] `docker run -d --name web nginx` — **Verify:** báo lỗi tên `web` đã bị chiếm dụng
- [ ] `docker rm web` — xóa container cũ (đã dừng)
- [ ] `docker run -d --name web nginx` — **Verify:** trả về container ID
- [ ] `docker ps` — **Verify:** 2 container cùng chạy từ một image nginx (isolation cho phép)
- [ ] `docker images` — **Verify:** nginx chỉ chiếm 1 dòng (image chỉ fetch 1 lần)
- [ ] `docker stop web` → `docker ps -a` — **Verify:** web chuyển sang `Exited`
- [ ] `docker start web` — **Verify:** web chạy lại trong `docker ps`
- [ ] `docker inspect web` — quan sát Path, Args (lệnh thật sự chạy), các thuộc tính Linux
- [ ] Ghi nhớ: `docker stop` cho container vài giây đóng file đúng cách rồi mới tự kill nếu cần — **không dùng `docker kill` tay**

### Lab 1.5 — Container logs & troubleshoot

- [ ] `docker run --name mydb mariadb` — chạy foreground, container fail và trả prompt
- [ ] `docker ps` — **Verify:** không thấy mydb
- [ ] `docker ps -a` — **Verify:** mydb `Exited (1)` — exit code 1 = ứng dụng dừng không đúng cách
- [ ] `docker logs mydb` — **Verify:** thấy nguyên nhân — mariadb cần biến môi trường chưa được cung cấp
- [ ] Ghi nhớ quy trình: fail → `docker ps -a` (exit code) → `docker logs` (nguyên nhân)

### Lab 1.6 — Lab exercise cuối bài (từ khóa học)

Yêu cầu môi trường: Ubuntu LTS (server hoặc desktop) bản mới nhất hoặc liền trước — tác giả demo **24.04**, dùng **22.04** vẫn OK, cũ hơn không nên (dễ trục trặc).

- [ ] `sudo apt install -y vim git` — git là bắt buộc; vim là editor tác giả thích dùng (không bắt buộc cho lab, cài editor nào cũng được). Ghi chú hữu ích: **vim có sẵn trên kỳ thi CKAD**
- [ ] `git clone https://github.com/sandervanvugt/ckad.git` — đã làm trong Lab 1.0 thì bỏ qua
- [ ] Từ thư mục ckad chạy `./minikube-docker-setup.sh` — script cài cả minikube lẫn docker (mới chỉ quan tâm docker); chờ 1–2 phút
- [ ] `docker search nginx` — **Verify:** danh sách mọi image có "nginx" trong tên hoặc mô tả
  - Official build nằm trên cùng với tên ngắn gọn `nginx`; muốn chạy image khác thì dùng **đúng tên đầy đủ như liệt kê** (ví dụ `ubuntu/nginx`)
- [ ] `docker run nginx` — image download về rồi chạy nginx ở **foreground** (muốn chạy nền cần `-d` — nhưng phần cuối lab yêu cầu stop nên khỏi cần)
- [ ] Dừng container bằng **Ctrl+C** — **Verify:** `docker ps` không còn container nginx

## Bài 2 — Container Images

### Lab 2.1 — Quản lý container image

- [ ] `docker images` — **Verify:** thấy toàn bộ image kèm size; để ý busybox chỉ ~4 MB (image nền tảng)
- [ ] `docker pull alpine` — prefetch image, không cần chạy container; **Verify:** alpine xuất hiện trong `docker images`
- [ ] `docker image prune` — xác nhận khi được hỏi; **Verify:** báo dung lượng reclaimed (máy mới setup sẽ ít, máy dùng lâu sẽ nhiều)

### Lab 2.2 — Build image từ Dockerfile

*(Dockerfile demo nằm trong subdirectory `Dockerfile` của repo khóa học — đã clone ở Lab 1.0/1.6)*

- [ ] Mở thư mục `Dockerfile`, đọc file Dockerfile và script `countdown` — để ý `FROM alpine`, `COPY countdown .`, `ENTRYPOINT ./countdown`; script dùng `/bin/sh` thay vì `/bin/bash` (môi trường tối giản)
- [ ] `docker build -t myapp .` (chạy từ trong thư mục đó) — **Verify:** build xong rất nhanh
- [ ] `docker images` — **Verify:** myapp xuất hiện ~8.83 MB
- [ ] `docker image inspect myapp` — xem metadata của image (như `docker inspect` nhưng cho image)
- [ ] `docker run myapp` — **Verify:** countdown 1 phút; muốn lâu hơn thì truyền số phút làm argument

### Lab 2.3 — Tạo image bằng docker commit

- [ ] `docker run --name customweb -it nginx sh` — mở shell trên container nginx (`sh` thay default command)
- [ ] `touch /tmp/testfile` — tạo file trong container, rồi `exit` (shell đóng → container tắt theo)
- [ ] `docker commit customweb nginx:custom` — **Verify:** `docker images` thấy image mới `nginx:custom`
- [ ] `docker run -it nginx:custom ls -l /tmp/testfile` — **Verify:** file `/tmp/testfile` tồn tại trong image mới (thay đổi đã được commit)

### Lab 2.4 — Lab exercise cuối bài (từ khóa học)

Nhiệm vụ: tạo image tên `lab2image` từ Dockerfile trong repo khóa học *(transcript ghi "lab2-image" ở phần nhiệm vụ nhưng lệnh demo dùng `lab2image` — chọn một tên và dùng nhất quán là được)*.

- [ ] `find . -name Dockerfile` — chạy từ thư mục gốc repo khóa học để tìm vị trí Dockerfile; **Verify:** thấy đường dẫn — chính là Dockerfile đã dùng ở Lab 2.2, dùng lại không sao
- [ ] `docker build -t lab2image .` — chạy từ trong thư mục chứa Dockerfile; **Verify:** `docker images` thấy `lab2image` — xong lab

## Bài 3 — Kubernetes là gì

### Lab 3.1 — Khám phá CNCF landscape (demo cncf.io)

- [ ] Mở [cncf.io](https://www.cncf.io) → mục **Projects** — **Verify:** thấy danh sách projects kèm cột trạng thái
- [ ] Nhận diện 4 trạng thái: **graduated** (mature hoàn toàn) · **incubating** (đang được giới thiệu) · **sandbox** (mới) · **archived** (cũ)
- [ ] Tìm các projects nhắc trong bài: **etcd** (core database của K8s), **CoreDNS** (DNS), **Helm** (package manager), **CNI** (network interface) — **Verify:** tìm thấy từng project + ghi nhận trạng thái hiện tại

### Lab 3.2 — Lab cuối bài (từ khóa học)

Điều kiện: lab chạy thuận lợi trên Ubuntu VM đã clone repo + chạy Minikube script (Lab 1.0/1.6). Dùng môi trường Kubernetes khác → học Bài 4 trước rồi quay lại làm lab này.

- [ ] `minikube start` — khởi động Minikube; **Verify:** Kubernetes up and running
- [ ] `kubectl api-resources` — liệt kê toàn bộ API resources; danh sách dài, tràn màn hình
- [ ] `kubectl api-resources | less` — pipe qua pager để đọc *(transcript chép "LES utility" — đúng là lệnh `less`)*; **Verify:** thấy pods *(transcript chép "ports")*, services, deployments và nhiều resource khác
- [ ] `kubectl explain pod` — hiện các field trong resource specification; để ý field quan trọng nhất: `spec`
- [ ] `kubectl explain pod.spec` — **Verify:** thấy toàn bộ parameters của pod specification *(học kỹ ở các bài sau)*
- [ ] Browser mở [kubernetes.io/docs](https://kubernetes.io/docs/), gõ `pod` vào search bar — **Verify:** thấy danh sách docs về pod, click vào có description + YAML code *(chi tiết YAML ở bài kế tiếp)*
- [ ] Ghi nhớ cho kỳ thi: **kubernetes.io/docs được phép dùng trên exam** — tận dụng tối đa

## Bài 4 — Kubernetes Deployment Options & Lab Environment

### Lab 4.1 — All-in-one bằng kubeadm (tùy chọn, nâng cao)

*(Không thuộc đề thi CKAD — dành cho người đã có kinh nghiệm Kubernetes. Bỏ qua lab này không cản trở các bài sau nếu đã dùng Minikube.)*

- [ ] SSH vào Ubuntu LTS server VM; `git clone https://github.com/sandervanvugt/ckad` — đã có repo từ Lab 1.0 thì bỏ qua bước clone
- [ ] Trong repo chỉ dùng 2 script: `setup-container.sh` + `setup-kubetools.sh` — bỏ qua các script khác
- [ ] `sudo ./setup-container.sh` — **Verify:** `systemctl status containerd` → active (running)
- [ ] `sudo ./setup-kubetools.sh` — cài kubectl, kubeadm, kubelet từ Kubernetes repositories
- [ ] `sudo kubeadm init` — 5+ phút; đọc kỹ output: 3 lệnh setup client + yêu cầu deploy pod network + lệnh join worker node
- [ ] Paste 3 lệnh setup client (mkdir + copy `admin.conf` về `~/.kube/config` + chown) — **Verify:** `kubectl get all` trả về thông tin cluster
- [ ] `kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml` — cài pod network
- [ ] `kubectl get pods -n kube-system` — **Verify:** calico-node + coredns chuyển từ ContainerCreating sang **Running** (chờ nếu cần)
- [ ] `kubectl run testpod --image=nginx` — **Verify:** `kubectl get pods` = Pending (control node chưa nhận user pod)
- [ ] `kubectl edit node <tên-node>` — tìm taint, xóa 3 dòng taint, thoát editor — **Verify:** testpod Pending → ContainerCreating → **Running**
- [ ] Ghi nhớ: môi trường này phải tự cài ingress controller, storage provisioner… — tác giả: "for experienced people only"

### Lab 4.2 — Ứng dụng đầu tiên với kubectl

- [ ] `source <(kubectl completion bash)` — **Verify:** gõ `kubectl create ` rồi Tab 2 lần → hiện danh sách subcommand
- [ ] `kubectl -h | less` — để ý examples nằm ở **đầu** output (kubectl không có man pages)
- [ ] `kubectl create deployment -h | less` — xem examples + toàn bộ options
- [ ] `kubectl create deployment firstapp --image=nginx` — **Verify:** báo `deployment.apps/firstapp created`
- [ ] `kubectl get all` — **Verify:** thấy đủ 3 đối tượng — deployment, replicaset (do deployment quản lý), 1 pod

### Lab 4.3 — Lab cuối bài: dựng lab environment (từ khóa học)

Thứ tự ưu tiên của tác giả: (1) Ubuntu LTS **desktop** VM, hoặc **server** VM — Minikube không cần giao diện đồ họa; (2) Ubuntu trên Windows WSL; (3) cài thẳng trên Windows/macOS — không khuyến nghị; (4) cloud distribution — phương án cuối cùng.

Demo chạy trên Ubuntu server (đã SSH vào — không cần desktop):

- [ ] `git clone https://github.com/sandervanvugt/ckad` — đã làm ở Lab 1.0 thì bỏ qua
- [ ] Từ thư mục ckad chạy `./minikube-docker-setup.sh` — chính là script đã dùng ở Lab 1.0
- [ ] `minikube start --cni=calico` — dùng CNI calico để có network policy; xem thêm `minikube start -h` cho các option `--cni`, `--cpus`, `--memory`, `--disk-size`; chờ ~5 phút
- [ ] `kubectl get all` — **Verify:** có output (service/kubernetes…) = lab environment sẵn sàng — xong trọn Module 1
