# Bài 2 — Container Images

Image là **nền tảng của mỗi container**. Bài học này gồm: cách image được cấu tạo
(architecture), cách quản lý image, và cách **tự tạo image** cho ứng dụng riêng.

**Mục tiêu bài học:**

- Container image được cấu tạo từ những gì (layers)
- Tạo image bằng **Dockerfile**
- Tạo image bằng **docker commit** — tạo image chứa toàn bộ cấu hình hiện tại của một container đang chạy
- Lab cuối bài để thực hành các kỹ năng trên

## Phần 1 — Managing Container Images

**Vì sao cần biết tạo image:** khi làm việc với Kubernetes, chắc chắn sẽ muốn chạy cả **ứng dụng riêng** của mình → cần biết cách tạo container image.

**Container image architecture:**

- Application images thường dùng **common system images** làm nền tảng
- System images là các bản **Linux distribution được container hóa ở dạng generic** — ví dụ **BusyBox**, **Alpine**
- Muốn xem một image dùng gì làm nền tảng → xem trong **Dockerfile**, tại lệnh `FROM` (trỏ tới system image được dùng làm base) — sẽ tự viết Dockerfile ở phần sau

**Layers — cách image được cấu thành:**

- Các modification (thêm ứng dụng cụ thể lên base) được lưu trong **layer riêng biệt**
- Đây là lý do làm việc với container image **hiệu quả**:
  - Base image BusyBox là generic, được nhiều ứng dụng dùng chung
  - Chạy MariaDB "on top" BusyBox: MariaDB chỉ cần **include image có sẵn như một layer riêng** — không phải đóng gói lại toàn bộ từ đầu
  - Chạy app của bạn on top MariaDB → lại thêm một layer riêng nữa

```
   ┌──────────────────┐
   │    Your app      │  ← layer 3: ứng dụng riêng
   ├──────────────────┤
   │    MariaDB       │  ← layer 2: include base như một layer riêng
   ├──────────────────┤
   │  BusyBox (base)  │  ← layer 1: generic system image
   └──────────────────┘
```

- Container image **bản chất là layered** — ví dụ này 3 layer, thực tế có thể 10–15 layer tùy ứng dụng

**Quản lý image hiện có:**

- **Thủ tục chuẩn:** trước khi container start, image cần thiết được **pull từ registry về lưu local** trên máy chạy container *(liên hệ Bài 1 Phần 2: vòng đời build → push → registry → pull → run)*
- `docker pull` — **prefetch** image trước; tiện khi không muốn mỗi lần chạy lại phải lên online lấy image
- `docker images` — liệt kê toàn bộ image đang lưu trên máy
- **Image pull policy:** image mới có thể **tự động được pull** nếu có. Theo tác giả: default của Docker là **Always**; trong Kubernetes về sau còn có thể đặt chế độ **chỉ pull khi có version mới hơn** trên registry

> **Làm rõ pull policy** *(bổ sung từ nghiên cứu ngoài video — kiểm tra lại video nếu cần)*: Kubernetes `imagePullPolicy` chỉ nhận `Always` / `IfNotPresent` / `Never` — **không có giá trị "Newer"**; mặc định là `IfNotPresent` với tag cố định, `Always` với tag `:latest`. Giá trị `newer` là tùy chọn `--pull` của **podman**; còn **Docker** thực tế chỉ pull khi image chưa có local. Chi tiết khi học Pod ở module sau.
>
> `docker image prune` — xóa các image không còn dùng (**dangling images**); nên chạy thỉnh thoảng để giải phóng dung lượng

**Demo — quản lý image:**

| Lệnh | Kết quả |
|---|---|
| `docker images` | Liệt kê mọi image kèm **size** — busybox chỉ **4 MB**, một image nền tảng cực nhỏ |
| `docker pull alpine` | Prefetch **alpine** — một minimal image khác; hiện trong `docker images` nhưng "to" hẳn so với busybox |
| `docker image prune` | Hỏi xác nhận → xóa dangling images và báo dung lượng reclaimed. Máy demo mới dùng < 1 giờ nên chẳng dọn được gì; máy dùng lâu sẽ reclaim được nhiều |

> Đào sâu về layers (layer caching, `docker image history`, `save/load`): xem [docker-mastery.md](docker-mastery.md) *(tham khảo ngoài phạm vi CKAD)*.

## Phần 2 — Creating Container Images

**Vì sao cần tự tạo image:** container đã trở thành **chuẩn để phân phối ứng dụng** → cần biết cách tạo image. Việc này không khó — có nhiều phương pháp.

**3 phương pháp tạo custom image:**

| Phương pháp | Cách hoạt động | Nhận xét |
|---|---|---|
| `docker commit` | Tạo image bằng cách **lưu các thay đổi đã thực hiện trên một container đang chạy** | Dùng được, nhưng **tác giả không khuyến nghị** |
| **Dockerfile** / Containerfile | Build image từ các thành phần **định nghĩa sẵn trong file** | Phương pháp chính — dùng ở phần sau |
| **buildah** | Giải pháp **nâng cao**, build image theo kiểu scripted | Ngoài phạm vi khóa, chỉ cần biết tồn tại *(transcript chép nhầm "wanted to manage it" — đúng là "wanted to mention it")* |

**Terminology — Dockerfile vs Containerfile:**

- **Dockerfile**: dùng trong môi trường **Docker**
- **Containerfile**: cái tên **Red Hat** muốn dùng; phổ biến hơn trong môi trường **podman** (với podman thì Docker là "đối thủ")
- Dùng tên nào cũng được — cả hai đều OK

**Base image:**

- Là image **tối giản** dùng làm nền tảng để build image riêng; thường là một **Linux distribution tối giản nhưng đầy đủ chức năng**
- Đã gặp 2 base image ở Phần 1: **Alpine** và **BusyBox**
- **Đừng kỳ vọng mọi tool đều có sẵn** — mục đích của container image là **chạy một ứng dụng**, không phải môi trường đa dụng. Ví dụ: để chạy ứng dụng chính thì cần gì công cụ `ps`? — sẽ thường gặp tình huống thiếu tool này khi làm việc bên trong container image

3 base image phổ biến nhất cần biết để lựa chọn:

| Base image | Đặc điểm |
|---|---|
| **BusyBox** | **Nhỏ nhất** trong các base image (~4 MB như demo Phần 1) |
| **Alpine** | Minimal image phổ biến, hơi to hơn BusyBox |
| **Red Hat UBI** (Universal Base Image) | Dựa trên **RHEL**; Red Hat dùng nó cho toàn bộ container image của họ — hướng tới nếu bạn "Red Hat minded" |

## Phần 3 — Tạo image bằng Dockerfile

**Dockerfile là gì:**

- File **mô tả cách đi tới custom container image** — thường do application developers cung cấp
- Nếu Dockerfile chỉ dùng các standard component có sẵn trên public registry → chỉ cần distribute Dockerfile, **ai cũng build được image** từ nó

**Dockerfile vs Containerfile (tiếp Phần 2):** không có khác biệt chức năng, nhưng file tên **Containerfile** trong **Docker** sẽ báo lỗi → muốn tên file dùng được ở cả Docker lẫn podman → dùng **Dockerfile**

> Viết Dockerfile **không nằm trong đề CKAD** — phần này chỉ phân tích một Dockerfile đơn giản để hiểu chuyện gì đang diễn ra ("thật ra không phải khoa học tên lửa").

**Lệnh build image:**

```
docker build -t myapp .
```

- `-t` (tag) — tên image muốn tạo
- `.` — Dockerfile nằm ở **current directory**; build từ nơi khác thì thay `.` bằng đường dẫn thư mục chứa Dockerfile

**Phân tích Dockerfile demo** (subdirectory `Dockerfile` trong repo khóa học):

| Instruction | Ý nghĩa |
|---|---|
| `FROM alpine` | System image làm base. Alpine chỉ ~8 MB — không chênh bao nhiêu so với 4 MB của BusyBox; nếu từng MB đều quan trọng thì cứ dùng BusyBox |
| `MAINTAINER` | Tên + contact details của người tạo Dockerfile — lịch sự với người dùng image |
| `VOLUME` + `WORKDIR` | Định nghĩa environment dùng khi build image để chạy ứng dụng |
| `COPY countdown .` | Copy file `countdown` từ current directory tới `.` — `.` ở đây chính là workdir |
| `ENTRYPOINT ./countdown` | Ứng dụng mặc định chạy khi container start. `countdown` là script nhận argument = **số phút đếm ngược**; mặc định 1 phút |

**Countdown script — bài học về môi trường tối giản:**

- Script dùng `/bin/sh` thay vì `/bin/bash` vì **compatibility**: container image là môi trường tối giản — `bash` có thể không có sẵn (liên hệ Phần 2: đừng kỳ vọng mọi tool đều có)
- Thói quen luôn để `#!/bin/bash` làm interpreter sẽ gặp vấn đề trong minimized environment

**Demo — build và chạy:**

| Lệnh | Kết quả |
|---|---|
| `docker build -t myapp .` | Build image từ Dockerfile của current directory, đặt tên myapp — xong rất nhanh |
| `docker images` | myapp xuất hiện với size **8.83 MB** |
| `docker image inspect myapp` | Như `docker inspect` (Bài 1 Phần 4) nhưng cho **image** — hiện contents của container image để phân tích |
| `docker run myapp` | Chạy image → countdown 1 phút (default argument) |
| `docker run myapp <số phút>` | Muốn countdown lâu hơn → truyền số phút làm argument |

> Danh sách đầy đủ Dockerfile instructions (16 lệnh, kèm best practices): xem [docker-mastery.md](docker-mastery.md) *(tham khảo ngoài phạm vi CKAD)*.

## Phần 4 — Tạo image bằng docker commit

**Ý tưởng:** thay vì Dockerfile, tạo image **từ một container đang chạy** — quy trình 3 bước:

1. Chạy container
2. Thực hiện mọi modification mong muốn
3. `docker commit` — lưu các thay đổi đó thành image mới

**Demo:**

| Bước | Lệnh | Kết quả |
|---|---|---|
| 1 | `docker run --name customweb -it nginx sh` | Mở shell trên container nginx — `sh` **thay thế default command** (entrypoint nginx) |
| 2 | `touch /tmp/testfile` | Tạo file test bên trong container *(transcript chép "search" — đúng là `touch`, lệnh tạo file)* |
| 3 | `exit` | Thoát shell → **container tắt theo**: vì default command đã bị thay bằng shell, shell đóng = container đóng |
| 4 | `docker commit customweb nginx:custom` | Lưu trạng thái hiện tại của container `customweb` thành image mới tên `nginx:custom` |
| 5 | `docker images` | **Verify:** xuất hiện image mới — name `nginx`, tag `custom` |
| 6 | `docker run -it nginx:custom ls -l /tmp/testfile` | Chạy image mới với lệnh `ls -l` làm command — không mở container dài hạn, chỉ liệt kê thuộc tính file để xác nhận `/tmp/testfile` tồn tại trong image |

- Các modification được lưu thành **layer riêng trên nền image gốc** (liên hệ kiến trúc layered ở Phần 1)
- Nhắc lại Phần 2: tác giả **không khuyến nghị** cách làm này — Dockerfile mới là phương pháp chuẩn
