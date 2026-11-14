# Docker Mastery — Từ Fresher đến Senior

> Tài liệu tham khảo **ngoài phạm vi đề CKAD** *(tổng hợp từ nghiên cứu ngoài video)*.
> Phục vụ mục tiêu Senior Backend / DevOps Engineer. Với CKAD chỉ cần phần lệnh cơ bản
> đã tổng hợp trong [Bài 1](01-understanding-and-using-containers.md#tổng-hợp-lệnh-docker-trong-bài-1).

Đừng học Docker theo kiểu thuộc lòng câu lệnh — cần thành thạo 3 năng lực:

1. **Docker Operations** — quản lý container, image, network, volume và Docker Engine
2. **Docker Troubleshooting** — điều tra và xử lý vấn đề thực tế trong production
3. **Docker Architecture & Optimization** — hiểu sâu cơ chế hoạt động, tối ưu hiệu năng, bảo mật, thiết kế Dockerfile/Compose

---

# PHẦN I. Các lệnh Docker cần thành thạo

## 1. Container Management

Nhóm lệnh dùng hàng ngày.

| Command | Chức năng |
|---|---|
| `docker run` | Tạo và chạy container |
| `docker create` | Tạo container nhưng chưa chạy |
| `docker start` | Khởi chạy container đã tồn tại |
| `docker stop` | Dừng container một cách graceful |
| `docker kill` | Gửi signal để dừng container |
| `docker restart` | Khởi động lại container |
| `docker pause` | Tạm dừng tiến trình container |
| `docker unpause` | Tiếp tục container |
| `docker rm` | Xóa container |
| `docker ps` | Liệt kê container đang chạy |
| `docker ps -a` | Liệt kê tất cả container |
| `docker inspect` | Xem cấu hình và metadata chi tiết |
| `docker exec` | Thực thi lệnh bên trong container |
| `docker attach` | Attach vào luồng stdin/stdout của container |
| `docker logs` | Xem log container |
| `docker top` | Xem process đang chạy |
| `docker stats` | Theo dõi CPU, RAM, Network, I/O |
| `docker port` | Xem port mapping |
| `docker cp` | Copy file giữa host và container |
| `docker diff` | Xem thay đổi filesystem so với image ban đầu |
| `docker rename` | Đổi tên container |
| `docker wait` | Chờ container kết thúc và trả về exit code |

**Các lệnh cần đặc biệt thành thạo:**

```bash
# Chạy container với các cấu hình quan trọng
docker run -d \
  --name my-app \
  --restart unless-stopped \
  -p 8080:3000 \
  -e NODE_ENV=production \
  --memory=512m \
  --cpus=1.5 \
  nginx:latest

# Kiểm tra container
docker ps -a

# Xem log liên tục
docker logs -f --tail=100 my-app

# Xem log trong khoảng thời gian
docker logs --since=30m my-app

# Vào container
docker exec -it my-app sh

# Chạy lệnh không cần interactive shell
docker exec my-app env

# Kiểm tra cấu hình chi tiết
docker inspect my-app

# Theo dõi tài nguyên
docker stats

# Xem process
docker top my-app

# Copy file từ container ra host
docker cp my-app:/var/log/app.log ./app.log
```

**Senior cần hiểu sâu:**

- Sự khác nhau giữa `stop`, `kill`, `restart`
- SIGTERM và SIGKILL
- Graceful shutdown và thời gian chờ `--stop-timeout`
- PID 1 trong container
- Exit code 0, 1, 137, 143
- Restart policies: `no`, `always`, `unless-stopped`, `on-failure`

## 2. Image Management

| Command | Chức năng |
|---|---|
| `docker images` / `docker image ls` | Liệt kê images |
| `docker pull` | Download image |
| `docker push` | Upload image |
| `docker build` | Build image |
| `docker tag` | Gắn tag cho image |
| `docker rmi` | Xóa image |
| `docker image inspect` | Xem metadata |
| `docker image history` | Xem các layer trong image |
| `docker image prune` | Xóa image không sử dụng |
| `docker save` | Export image thành file archive |
| `docker load` | Import image từ archive |
| `docker export` | Export filesystem của container |
| `docker import` | Tạo image từ filesystem archive |
| `docker manifest inspect` | Xem manifest của image |

```bash
# Build image
docker build -t crm-api:v1.0 .

# Build với Dockerfile cụ thể
docker build -f Dockerfile.prod -t crm-api:prod .

# Build không sử dụng cache
docker build --no-cache -t crm-api:v1.0 .

# Build với build arguments
docker build --build-arg NODE_ENV=production -t crm-api:v1.0 .

# Kiểm tra image layers / metadata
docker image history crm-api:v1.0
docker image inspect crm-api:v1.0

# Push image lên Registry
docker tag crm-api:v1.0 registry.example.com/crm-api:v1.0
docker push registry.example.com/crm-api:v1.0

# Save / load image
docker save -o crm-api.tar crm-api:v1.0
docker load -i crm-api.tar
```

**Phân biệt `save/load` vs `export/import`** — câu hỏi phỏng vấn Senior phổ biến vì nhiều developer nhầm lẫn:

| Command | Nội dung |
|---|---|
| `docker save` | Lưu **image** cùng các layer và metadata |
| `docker load` | Khôi phục image từ archive |
| `docker export` | Xuất **filesystem của một container** |
| `docker import` | Tạo image mới từ filesystem archive |

## 3. Docker Network

Rất quan trọng với Backend Engineer và DevOps Engineer.

| Command | Chức năng |
|---|---|
| `docker network ls` | Liệt kê network |
| `docker network create` | Tạo network |
| `docker network inspect` | Xem cấu hình network |
| `docker network connect` | Kết nối container vào network |
| `docker network disconnect` | Ngắt kết nối |
| `docker network rm` | Xóa network |
| `docker network prune` | Xóa network không sử dụng |

```bash
# Tạo network riêng
docker network create app-network

# Tạo container thuộc network
docker run -d --name api --network app-network nginx

# Kiểm tra network
docker network inspect app-network

# Kết nối / ngắt kết nối container hiện có
docker network connect app-network redis
docker network disconnect app-network redis
```

**Senior cần hiểu:**

- Bridge / Host / None / Overlay network
- DNS service discovery giữa các container
- Container-to-container communication
- Port publishing và port mapping
- NAT và iptables/nftables
- Network isolation

**Lỗi thực tế điển hình:** container A truy cập container B bằng `localhost` nhưng không được. Nguyên nhân: `localhost` bên trong container A trỏ về **chính container A**. Nếu cùng Docker network, dùng **DNS name** của container/service.

## 4. Docker Volume & Storage

| Command | Chức năng |
|---|---|
| `docker volume ls` | Liệt kê volumes |
| `docker volume create` | Tạo volume |
| `docker volume inspect` | Xem thông tin volume |
| `docker volume rm` | Xóa volume |
| `docker volume prune` | Xóa volume không sử dụng |

```bash
# Tạo volume
docker volume create postgres-data

# Chạy PostgreSQL với persistent storage
docker run -d \
  --name postgres \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:16

# Kiểm tra volume
docker volume inspect postgres-data

# Mount thư mục host vào container
docker run -d -v /home/user/data:/app/data nginx
```

**Phân biệt 3 loại storage:**

| Loại | Đặc điểm |
|---|---|
| Named Volume | Docker quản lý vòng đời và vị trí lưu trữ |
| Bind Mount | Mount trực tiếp thư mục/file của host |
| tmpfs | Lưu dữ liệu tạm thời trong memory |

Cần hiểu thêm: persistence; file permissions và UID/GID; read-only mount; volume backup/restore; volume lifecycle; storage driver và copy-on-write.

## 5. Docker Compose

Nhóm lệnh rất đáng đầu tư với Backend Engineer.

| Command | Chức năng |
|---|---|
| `docker compose up` | Tạo và chạy services |
| `docker compose down` | Dừng và xóa resources do Compose tạo |
| `docker compose ps` | Liệt kê services |
| `docker compose logs` | Xem logs |
| `docker compose exec` | Thực thi lệnh trong service |
| `docker compose build` | Build images |
| `docker compose pull` / `push` | Pull / push images |
| `docker compose restart` / `stop` / `start` | Restart / dừng / khởi động services |
| `docker compose config` | Validate và render cấu hình |
| `docker compose top` / `stats` | Xem process / tài nguyên |
| `docker compose run` | Chạy one-off container |
| `docker compose rm` | Xóa stopped containers |
| `docker compose watch` | Theo dõi thay đổi và đồng bộ/rebuild theo cấu hình |

```bash
# Kiểm tra cấu hình
docker compose config

# Build và chạy toàn bộ services
docker compose up -d --build

# Chỉ chạy một service
docker compose up -d api

# Theo dõi log / vào container / restart
docker compose logs -f api
docker compose exec api sh
docker compose restart api

# Pull image mới
docker compose pull

# Dừng và xóa containers, networks
docker compose down

# Xóa cả named volumes do Compose quản lý
docker compose down -v
```

> ⚠️ `docker compose down -v` có thể xóa volume chứa database data — không dùng tùy tiện trên production.

## 6. Docker System & Cleanup

Thường dùng khi Docker Host bị đầy ổ cứng.

| Command | Chức năng |
|---|---|
| `docker info` | Thông tin Docker Engine |
| `docker version` | Phiên bản client/server |
| `docker system df` | Kiểm tra dung lượng Docker |
| `docker system events` | Theo dõi Docker events |
| `docker system prune` | Dọn dẹp resources |
| `docker system prune -a` | Dọn dẹp thêm unused images |
| `docker container prune` | Xóa stopped containers |
| `docker image prune` | Xóa unused images |
| `docker volume prune` | Xóa unused volumes |
| `docker network prune` | Xóa unused networks |

```bash
docker system df        # tổng quan dung lượng
docker system df -v     # chi tiết
docker system prune     # dọn stopped containers + unused networks
docker system prune -a  # dọn thêm toàn bộ unused images
docker events           # theo dõi events
```

> **Senior cần chú ý:** không phải resource nào không dùng tại thời điểm hiện tại cũng xóa an toàn được — đặc biệt volumes và images chuẩn bị cho deployment tiếp theo.

## 7. Docker Security & Registry

| Command | Chức năng |
|---|---|
| `docker login` / `logout` | Đăng nhập / đăng xuất Registry |
| `docker search` | Tìm kiếm Docker Hub |
| `docker scout` | Phân tích image và lỗ hổng bảo mật |
| `docker buildx` | Build images nâng cao, đa nền tảng |
| `docker context` | Quản lý Docker endpoints |
| `docker plugin` | Quản lý Docker plugins |

```bash
# Login private registry
docker login registry.example.com

# Build multi-platform image
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t registry.example.com/api:v1 \
  --push .

# Kiểm tra / chuyển context
docker context ls
docker context use default

# Kiểm tra image vulnerabilities
docker scout cves my-app:v1
```

**Kiến thức bảo mật quan trọng:**

- Không chạy container bằng root nếu không cần thiết (dùng `USER` trong Dockerfile)
- Không hardcode secrets trong Dockerfile
- Read-only filesystem khi phù hợp
- Giới hạn Linux capabilities
- Không mount Docker socket vào container tùy tiện
- Scan image vulnerabilities; dùng trusted base images

## 8. Docker Engine & Host Troubleshooting

Senior phải biết kiểm tra cả Docker Engine và Linux Host, không chỉ Docker CLI.

| Command | Chức năng |
|---|---|
| `systemctl status docker` | Kiểm tra Docker service |
| `systemctl restart docker` | Restart Docker Engine |
| `journalctl -u docker` | Xem Docker daemon logs |
| `journalctl -u docker -f` | Theo dõi daemon logs |
| `docker info` | Kiểm tra cấu hình Engine |
| `docker events` | Theo dõi sự kiện Docker |
| `df -h` | Kiểm tra dung lượng ổ đĩa |
| `free -h` | Kiểm tra RAM |
| `ps aux` | Kiểm tra process |
| `ss -tulpn` | Kiểm tra các port đang listen |
| `ip addr` / `ip route` | Kiểm tra network interfaces / routing |
| `lsns` | Kiểm tra Linux namespaces |
| `nsenter` | Truy cập namespace của process khác |
| `mount` | Kiểm tra filesystem mounts |

**Quy trình điều tra "Docker không hoạt động":**

```bash
systemctl status docker                        # 1. Docker Engine
journalctl -u docker -n 200 --no-pager         # 2. daemon logs
df -h                                          # 3. dung lượng ổ đĩa
docker system df -v                            # 4. Docker storage
ps aux | grep dockerd                          # 5. process
docker version                                 # 6. phiên bản client/server
docker info                                    # 7. cấu hình Engine
```

Trên Ubuntu, nếu Docker cài bằng **Snap** thay vì Docker Engine package thông thường, kiểm tra thêm:

```bash
snap services docker
snap logs docker
```

Hữu ích khi xử lý môi trường có nhiều phương thức cài Docker khác nhau.

## 9. Dockerfile Instructions

Không phải command chạy trong terminal mà là các instruction trong Dockerfile.

| Instruction | Mục đích |
|---|---|
| `FROM` | Chọn base image |
| `RUN` | Chạy lệnh trong quá trình build |
| `CMD` | Lệnh mặc định khi chạy container |
| `ENTRYPOINT` | Executable chính của container |
| `COPY` | Copy file vào image |
| `ADD` | Copy file, hỗ trợ thêm giải nén archive |
| `WORKDIR` | Thiết lập working directory |
| `ENV` | Thiết lập environment variables |
| `ARG` | Khai báo build-time variables |
| `EXPOSE` | Khai báo port mà ứng dụng sử dụng |
| `USER` | Chỉ định user chạy process |
| `VOLUME` | Khai báo mount point |
| `HEALTHCHECK` | Kiểm tra tình trạng container |
| `LABEL` | Gắn metadata |
| `SHELL` | Thay đổi shell dùng trong shell-form instructions |
| `STOPSIGNAL` | Chỉ định signal dùng để dừng container |

**Chủ đề Dockerfile Senior phải thực sự hiểu:** multi-stage builds; layer caching; BuildKit; build context; `.dockerignore`; ENTRYPOINT vs CMD; shell form vs exec form; ARG vs ENV; image size optimization; non-root execution; graceful shutdown.

---

# PHẦN II. Câu hỏi phỏng vấn — Junior Level

Không tập trung vào định nghĩa mà kiểm tra khả năng sử dụng Docker và xử lý vấn đề.

## Nhóm 1: Container Operations

**Q1.** Chạy container bằng `docker run -d` nhưng container lập tức chuyển sang trạng thái Exited. Điều tra thế nào?
*Interviewer muốn kiểm tra:* `docker ps -a` · `docker logs` · `docker inspect` · hiểu Exit Code · hiểu CMD và ENTRYPOINT.

**Q2.** Container chạy thành công nhưng không truy cập được ứng dụng qua `localhost:8080`. Kiểm tra những gì?
*Gợi ý:* port mapping · application binding address · container network · firewall · application logs.

**Q3.** Container liên tục restart. Xác định nguyên nhân thế nào?

**Q4.** Cần vào container đang chạy để kiểm tra file config. Dùng lệnh nào? Nếu container không có `bash` thì sao?

**Q5.** Container đang chạy bình thường; restart Docker Engine thì chuyện gì có thể xảy ra?
*Muốn biết:* hiểu restart policy, daemon behavior và trạng thái container.

## Nhóm 2: Image & Dockerfile

**Q6.** Dockerfile có 10 layers. Chỉ thay đổi một file source code — Docker có phải build lại toàn bộ image không? Vì sao?

**Q7.** Image Node.js 1.5GB — giảm xuống ~200MB thế nào?
*Gợi ý:* multi-stage build · base image phù hợp · `.dockerignore` · chỉ copy dependencies cần thiết · không đưa build tools vào runtime image.

**Q8.** Build trên máy local thành công nhưng chạy trên server lỗi `exec format error`. Nguyên nhân?
*Gợi ý:* CPU architecture (AMD64 vs ARM64).

**Q9.** Sửa một dòng trong Dockerfile nhưng Docker vẫn dùng cache cũ. Kiểm tra và xử lý thế nào?

**Q10.** Chuyển Docker Image sang server không có Internet?
*Gợi ý:* `docker save` và `docker load`.

## Nhóm 3: Network & Storage

**Q11.** Hai container cùng chạy trên một host. Cho container A kết nối container B mà không publish port ra host thế nào?

**Q12.** Container PostgreSQL bị xóa và tạo lại. Tại sao dữ liệu database vẫn còn hoặc bị mất?

**Q13.** Mount thư mục từ host vào container nhưng ứng dụng không ghi được file. Nguyên nhân?
*Gợi ý:* UID/GID · file permissions · read-only mount · SELinux/AppArmor tùy môi trường.

**Q14.** Hai container cùng dùng port 3000 **bên trong** container. Có conflict không?

**Q15.** Container A ping được container B bằng IP nhưng không resolve được tên container B. Kiểm tra gì?
*Gợi ý:* user-defined bridge network, Docker DNS và network attachment.

---

# PHẦN III. Câu hỏi phỏng vấn — Senior Level

Kiểm tra khả năng phân tích nguyên nhân gốc rễ, hiểu Linux và thiết kế giải pháp production — không đơn thuần nhớ câu lệnh.

## Nhóm 1: Container Runtime & Linux Internals

**Q1. Container không phải Virtual Machine — điều gì thực sự xảy ra khi chạy `docker run`?**
Cần giải thích: Docker CLI giao tiếp với Docker Daemon qua Docker API → Docker Engine quản lý container lifecycle → container runtime tạo container theo OCI specification → Linux **Namespaces** cung cấp isolation → **cgroups** quản lý và giới hạn tài nguyên → **Overlay filesystem** cung cấp writable layer.

**Q2.** Container bị OOMKilled dù `docker stats` cho thấy RAM sử dụng vẫn thấp. Điều tra thế nào?
Cần xem xét: memory limit của container · memory usage tại thời điểm lỗi · Linux OOM Killer · memory accounting · host memory pressure · kernel logs.

```bash
docker inspect <container>
docker stats
dmesg -T | grep -i -E 'oom|killed process'
journalctl -k
```

**Q3.** Vì sao process trong container có thể có PID 1? PID 1 có đặc điểm gì khác process thông thường?
Cần hiểu: PID namespace · signal handling · zombie processes · orphan processes · init process và process reaping.

**Q4.** Vì sao chạy ứng dụng bằng shell-form ENTRYPOINT có thể gây vấn đề khi Docker gửi SIGTERM?
Cần hiểu process tree, shell process, signal forwarding và exec form.

**Q5.** Container bị giới hạn CPU ở 1 core nhưng ứng dụng vẫn tạo 20 threads thì điều gì xảy ra?
Cần hiểu CPU quota, throttling, scheduling và cgroups.

**Q6.** Docker container dùng chung Linux Kernel với host. Nếu container bị compromise, những lớp bảo vệ nào hạn chế ảnh hưởng đến host?
Cần hiểu: namespaces · cgroups · Linux capabilities · seccomp · AppArmor/SELinux · user namespaces · Rootless Docker · kernel vulnerabilities và container escape.

## Nhóm 2: Dockerfile & Build Optimization

**Q7.** Có một monolithic Dockerfile dùng chung cho dev/staging/production. Thiết kế lại thế nào?
*Muốn nghe:* multi-stage build · build arguments · environment configuration · separate runtime image · không đóng gói secrets vào image · reproducible builds.

**Q8.** Vì sao thứ tự các instruction trong Dockerfile ảnh hưởng đến build time?

```dockerfile
# Chậm: đổi source code → cài lại dependencies
COPY . .
RUN npm install

# Nhanh: đổi source code → tận dụng lại layer npm ci
COPY package*.json ./
RUN npm ci
COPY . .
```

Cần giải thích cơ chế **layer caching**: thay đổi source code không nên khiến dependencies bị cài lại.

**Q9.** Làm thế nào đảm bảo image build hôm nay và build lại sau 3 tháng vẫn cho kết quả nhất quán?
Cần phân tích: pin base image bằng digest · lock dependencies · reproducible builds · build environment · supply chain security · SBOM · image provenance.

**Q10.** Phát hiện image chứa AWS credentials/API keys trong một layer cũ, dù layer cuối đã xóa file đó. Tại sao?
Cần hiểu **image layers bất biến** — xóa file ở layer sau không xóa dữ liệu ở layer trước. Giải pháp: build lại image không chứa secret · rotate/revoke credentials · dùng BuildKit secrets · kiểm tra image history và các layer.

## Nhóm 3: Networking & Distributed Systems

**Q11.** Ứng dụng kết nối database được bằng IP nhưng không kết nối được bằng service name. Debug từ đâu?
Cần phân tích: network driver · Docker DNS · container network membership · network aliases · Compose service discovery.

**Q12.** Hai container nằm trên hai Docker Hosts khác nhau — giao tiếp với nhau thế nào?
Cần phân biệt: bridge network chỉ trong một Docker Host · overlay network · routing · port publishing · service discovery · VPN hoặc external networking.

**Q13.** Một container truy cập Internet được, container khác thì không. Kiểm tra những thành phần nào trên Host?
Cần hiểu: routing table · DNS · NAT · iptables/nftables · Docker bridge · firewall · network interface.

**Q14.** Vì sao service truy cập được từ trong Docker network nhưng không truy cập được từ ngoài Host?
Phân tích: port publishing · binding address · firewall · network routing · reverse proxy.

## Nhóm 4: Production Troubleshooting

**Q15.** Docker Host có 100GB disk nhưng sau vài ngày đầy disk. Xác định Docker chiếm dung lượng ở đâu?
Cần điều tra: container writable layers · container logs · image layers · build cache · volumes · Overlay2 storage · Docker data root.

```bash
docker system df -v
docker ps -s
docker inspect <container>
du -sh /var/lib/docker/*
df -h
```

**Q16.** Container hoạt động bình thường 2 ngày rồi response time tăng rất cao. Điều tra thế nào?
Cần **phương pháp**, không chỉ một câu lệnh: (1) CPU và memory → (2) CPU throttling → (3) disk I/O → (4) network → (5) application logs → (6) process và thread → (7) connection pool → (8) memory leak.

**Q17.** Docker daemon bị crash — điều gì xảy ra với các container đang chạy?
Cần phân biệt: daemon restart · container runtime behavior · `live-restore` · container restart policies · container process survival.

**Q18.** Container không ghi được dữ liệu vào volume ở production nhưng local chạy bình thường. Kiểm tra gì?
*Gợi ý:* UID/GID khác nhau · host filesystem permissions · SELinux labels · read-only mounts · volume driver · filesystem compatibility.

**Q19.** 50 containers trên một Docker Host, một container liên tục dùng CPU 100%. Xác định container nào và process nào?

```bash
docker stats
docker top <container>
docker inspect <container>
```

Sau đó điều tra sâu hơn bằng `top`, `htop`, `ps`, `pidstat` và Linux cgroups.

**Q20.** Docker Host out of memory, nhiều container bị kill cùng lúc. Xử lý thế nào để hạn chế tái diễn?
*Mong đợi:* xác định OOM source · đánh giá memory limits · memory reservation và capacity planning · application memory profiling · monitoring và alerting · host-level memory protection.

---

# PHẦN IV. Các tình huống thực tế Senior thường gặp

Có giá trị cao hơn hỏi thuộc lòng câu lệnh.

## Scenario 1: Database mất dữ liệu sau deployment

*(Production incident)*

Hệ thống: Backend API + PostgreSQL đều chạy container; database lưu dữ liệu trong **container filesystem**. Developer chạy `docker compose down` rồi deploy lại → toàn bộ dữ liệu biến mất.

**Câu hỏi:**
1. Nguyên nhân thực sự là gì?
2. Khôi phục dữ liệu thế nào?
3. Thiết kế lại Docker Compose như thế nào để không lặp lại?
4. Triển khai backup và restore ra sao?

*Điểm đánh giá:* storage persistence, volume lifecycle, backup strategy, deployment safety.

## Scenario 2: Docker Image quá lớn

*(Performance optimization)*

Image 2.8GB, mỗi lần deploy lên Kubernetes mất 4–5 phút pull image. Nhiệm vụ: giảm thời gian deployment.

**Câu hỏi:**
1. Kiểm tra những gì trước khi sửa Dockerfile?
2. Xác định layer chiếm nhiều dung lượng nhất thế nào?
3. Có nên chuyển sang Alpine không?
4. Có nên dùng distroless image không?
5. Multi-stage build giải quyết vấn đề gì?
6. Tối ưu build cache trong CI/CD thế nào?
7. Đánh đổi giữa image size, security và debugging?

## Scenario 3: Container liên tục restart

Alert nhận được:

```
Container: crm-api
Status: Restarting
Exit Code: 137
Restart Count: 45
```

**Câu hỏi:**
1. Exit code 137 có ý nghĩa gì?
2. Có chắc chắn container bị OOMKilled không?
3. Xác định process bị kill thế nào?
4. Kiểm tra memory limit ở đâu?
5. Phân biệt application memory leak với memory limit quá thấp ra sao?
6. Giải pháp lâu dài?

> Điểm quan trọng: exit code 137 thường tương ứng SIGKILL nhưng **chưa đủ** để kết luận OOM — cần kiểm tra thêm `OOMKilled`, kernel logs và các sự kiện liên quan.

## Scenario 4: Container không kết nối được Database

Hai services `crm-api` và `postgres` đều running nhưng API báo:

```
ECONNREFUSED 127.0.0.1:5432
```

**Câu hỏi:**
1. Tại sao lại kết nối tới 127.0.0.1?
2. Làm thế nào để hai container giao tiếp?
3. Có cần publish port PostgreSQL ra host không?
4. Kiểm tra DNS resolution ra sao?
5. Nếu PostgreSQL khởi động chậm hơn API thì sao?
6. `depends_on` có đảm bảo database đã sẵn sàng nhận kết nối không?

*Điểm đánh giá:* Docker networking, service discovery, healthcheck, retry và dependency management.

## Scenario 5: Secret bị lộ trong Docker Image

Developer viết `ENV DATABASE_PASSWORD=secret123` rồi push image lên public registry.

**Câu hỏi:**
1. Xóa ENV ở layer cuối có đủ an toàn không?
2. Kiểm tra secret có nằm trong image thế nào?
3. Xử lý credentials đã bị lộ ra sao?
4. Dùng Docker BuildKit secrets thế nào?
5. Trên Kubernetes nên quản lý secrets ra sao?

## Scenario 6: Docker Daemon không khởi động

*(Infrastructure troubleshooting)*

Sau khi server reboot, Docker không hoạt động: `Cannot connect to the Docker daemon`.

**Câu hỏi:**
1. Kiểm tra daemon thế nào?
2. Kiểm tra Docker socket và permissions ra sao?
3. Phát hiện port hoặc socket conflict thế nào?
4. Kiểm tra storage driver thế nào?
5. Nếu `/var/lib/docker` đầy thì xử lý ra sao?
6. Nếu có hai Docker daemon cùng chạy thì điều tra thế nào?

## Scenario 7: Production bị ảnh hưởng bởi container khác

Một container batch job dùng gần 100% CPU, khiến API container response chậm.

**Câu hỏi:**
1. Xác định container gây tải thế nào?
2. Giới hạn CPU ra sao?
3. CPU shares và CPU quota khác nhau thế nào?
4. Memory limit tác động thế nào đến hiệu năng?
5. Đảm bảo workload quan trọng có đủ tài nguyên thế nào?

```bash
docker run -d \
  --name batch-worker \
  --cpus=1.0 \
  --memory=512m \
  my-worker:v1
```

---

# PHẦN V. Phân biệt tư duy Junior vs Senior

Cùng câu hỏi, cách trả lời thể hiện mức độ kinh nghiệm:

| Chủ đề | Junior thường trả lời | Senior cần phân tích |
|---|---|---|
| Container restart | Xem logs | Signal, PID 1, exit code, restart policy |
| Image quá lớn | Dùng image nhỏ hơn | Layer, build cache, multi-stage, supply chain |
| Database mất data | Dùng volume | Persistence, backup, recovery, lifecycle |
| Container không kết nối DB | Kiểm tra port | DNS, bridge, routing, firewall |
| Container OOM | Tăng RAM | cgroups, memory leak, OOM Killer, capacity |
| Docker không chạy | Restart Docker | Daemon, socket, storage driver, kernel |
| Build chậm | Dùng cache | Layer ordering, BuildKit, remote cache |
| Secret bị lộ | Xóa ENV | Layer history, credential rotation, secret management |
| Deploy lỗi | Restart container | Rollback, immutable image, deployment strategy |
| CPU tăng cao | Xem `docker stats` | CPU throttling, profiling, scheduling, resource contention |

## 15 câu hỏi khó để tự kiểm tra

1. Vì sao container có thể chạy tiếp khi Docker daemon bị restart? Điều kiện nào khiến nó không thể tiếp tục?
2. Docker OverlayFS hoạt động thế nào? Vì sao ghi vào container writable layer có thể chậm hơn volume?
3. Nếu container chạy với UID 0 thì có đồng nghĩa với root trên host không?
4. Docker CPU limit được thực thi bằng cơ chế nào của Linux?
5. Vì sao container bị OOMKilled dù host vẫn còn RAM?
6. Khi `docker compose down`, resource nào bị xóa và resource nào được giữ lại?
7. Docker DNS hoạt động thế nào giữa các container trong user-defined bridge?
8. Tại sao `localhost` trong container không trỏ đến host?
9. Vì sao `COPY . .` đặt trước `RUN npm ci` có thể khiến build chậm?
10. Làm thế nào đảm bảo Docker Image reproducibly build?
11. Image layer có thể chứa dữ liệu nhạy cảm đã bị xóa ở layer sau không?
12. Rootless Docker khác Rootful Docker thế nào?
13. Docker socket `/var/run/docker.sock` có rủi ro bảo mật gì?
14. Thiết kế chiến lược image tagging và rollback cho production thế nào?
15. Docker Host có 200 containers — thiết kế monitoring, logging và resource isolation ra sao?

## Checklist Docker Senior Readiness

**1. Docker Fundamentals & Operations**
- [ ] Thành thạo docker run, exec, inspect, logs, stats
- [ ] Hiểu container lifecycle và restart policies
- [ ] Hiểu image, container và image layers
- [ ] Hiểu CMD, ENTRYPOINT, PID 1 và signals

**2. Dockerfile & Image Optimization**
- [ ] Viết Dockerfile production-ready
- [ ] Multi-stage build và BuildKit
- [ ] Layer caching và build context
- [ ] Image tagging, registry, save/load
- [ ] Reproducible builds và image scanning

**3. Networking & Storage**
- [ ] Bridge, Host, None và Overlay network
- [ ] Docker DNS và service discovery
- [ ] Port mapping, NAT và firewall
- [ ] Named Volume, Bind Mount và tmpfs
- [ ] Backup, restore và permission management

**4. Production & Security**
- [ ] CPU, Memory limits và OOM troubleshooting
- [ ] Docker daemon và storage troubleshooting
- [ ] Linux namespaces, cgroups, capabilities
- [ ] Rootless Docker, seccomp, AppArmor
- [ ] Logging, monitoring và resource cleanup

**5. Docker Compose & CI/CD**
- [ ] Thiết kế multi-service Compose
- [ ] Healthcheck và dependency management
- [ ] Environment configuration và secrets
- [ ] Build/push image trong CI/CD
- [ ] Deployment, rollback và version management

## Thứ tự học khuyên dùng

Với nền tảng Backend → Docker → Kubernetes/AKS:

| Giai đoạn | Nội dung | Mục tiêu |
|---|---|---|
| 1 | Docker CLI, Container, Image | Thành thạo operations |
| 2 | Dockerfile, BuildKit, Compose | Tự thiết kế môi trường chạy ứng dụng |
| 3 | Network, Volume, Storage | Debug các vấn đề thực tế |
| 4 | Linux Internals, cgroups, namespaces | Hiểu sâu container runtime |
| 5 | Security, Monitoring, Troubleshooting | Đạt năng lực production |

**Chuỗi kiến thức từ source code đến production** — đừng chỉ học Docker CLI, hãy hiểu mối quan hệ:

```
Dockerfile & BuildKit          → build application image
        │
Container Registry             → store & distribute images
        │
Container Runtime              → containerd / CRI-O / runc
        │
Linux Kernel                   → namespaces / cgroups / capabilities
        │
Kubernetes / AKS               → orchestration & deployment
```

Hiểu trọn chuỗi này là hiểu từ lúc source code được đóng gói cho đến khi ứng dụng thực sự chạy trong production.
