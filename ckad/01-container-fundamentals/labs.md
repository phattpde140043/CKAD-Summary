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
