# Writing Guidelines — Tiêu chuẩn biên soạn CKAD-Summary

Quy ước biên soạn cho toàn bộ tài liệu trong repo — dùng làm tiêu chuẩn khi viết module mới,
sửa module cũ, hoặc giao cho AI agent biên soạn. Rút ra từ phong cách các module đã hoàn
thành (Module 1–2) sau bước đánh giá và chuẩn hóa.

**Câu định vị phong cách:** tài liệu kỹ thuật có cấu trúc, diễn đạt trực tiếp, ưu tiên giải
thích bản chất và cơ chế hoạt động, dùng ví dụ thực tế kết nối lý thuyết với thực hành.

**Nguyên tắc biên tập:** mỗi đoạn văn phải giúp người đọc hiểu thêm một khái niệm, một cơ
chế, một mối quan hệ hoặc một cách áp dụng. Không viết chỉ để nhắc lại thông tin đã biết.

## 1. Ngôn ngữ

- Viết tiếng Việt; **thuật ngữ kỹ thuật giữ nguyên tiếng Anh** và in đậm lần đầu xuất hiện:
  Pod, Deployment, image, registry, namespace...
- Giải thích bản chất bằng tiếng Việt; dùng tiếng Anh khi đó là tên chính thức của khái niệm,
  thành phần, lệnh hoặc property.
- **Đi thẳng vào vấn đề** — không mở đầu bằng câu chung chung. Câu ngắn, **mỗi câu một ý**.
- Tách câu khi một câu chứa ≥3 tầng thông tin (vd: định nghĩa + hệ quả + lý do học gộp chung).
- Mũi tên `→` chỉ dùng cho **chuỗi quan hệ ngắn (tối đa 3 mắt xích)**. Cơ chế cần nhiều hơn
  2 câu giải thích → viết thành đoạn văn.
- **Hạn chế từ tuyệt đối hóa** ("không bao giờ", "hoàn toàn", "chỉ") — dùng khi đúng kỹ thuật
  và kèm lý do ngay sau đó.
- Khẩu ngữ gần gũi ("không phải khoa học tên lửa") chỉ dùng trong **ghi chú kinh nghiệm /
  cảnh báo** — không dùng trong định nghĩa hay phần giải thích cơ chế.
- Bôi đậm từ khóa cần ghi nhớ; code format cho lệnh, tên file, property.

## 2. Cấu trúc bài học

| Thành phần | Quy ước |
|---|---|
| H1 | `# Bài N — Tên bài (tiếng Anh theo course)` |
| Intro | 2–4 dòng: vị trí bài trong kỳ thi, nội dung chính |
| `## Lộ trình bài học` | Danh sách đánh số theo thứ tự video |
| `## Phần N` | Theo thứ tự video — không sắp xếp lại theo chủ đề |
| `### Demo` | Mỗi demo một subsection: lệnh + kết quả mong đợi |
| Bảng | So sánh các khái niệm/lệnh theo cùng tiêu chí |
| Blockquote | `> **Ghi nhớ thi CKAD**` cho mẹo làm bài thi |

- Trong mỗi Phần, trình tự ưu tiên: **Hiện tượng → Nguyên nhân → Cơ chế → Cách kiểm chứng →
  Kết luận**. Đây là checklist tư duy, không phải template cứng — phần nào không áp dụng
  được thì bỏ.
- Giải thích trước, thực hành sau: khái niệm và cơ chế đặt trước demo lệnh.

## 3. Quy ước ghi chú nguồn và phạm vi

Marker bắt buộc — nội dung không đến trực tiếp từ video phải được đánh dấu:

| Marker | Dùng khi |
|---|---|
| `*(bổ sung từ nghiên cứu ngoài video)*` | Kiến thức người biên soạn thêm vào ngoài video |
| `*(Transcript ghi "X" — chuẩn hóa thành "Y".)*` | Sửa lỗi transcribe — đặt cuối section chứa lỗi |
| `**Không nằm trong đề CKAD**` | Phân định phạm vi cần học sâu |
| `*(nhìn trước — chi tiết ở module/bài sau)*` | Forward reference |
| `*(nhắc lại Bài N Phần M)*` | Backward reference |

- Transcript thường duplicated 2–5 lần kèm nhiễu UI → lọc bỏ âm thầm, chỉ xử lý một lần.
- Forward/backward reference dùng **có chủ đích** — không ghi chú khi mối liên hệ đã hiển nhiên.

## 4. Bảng, bullet, đoạn văn

- **Bảng**: so sánh nhiều đối tượng theo cùng tiêu chí, hoặc bảng tra cứu lệnh (syntax +
  chức năng ngắn gọn).
- **Bullet**: các luận điểm độc lập cùng cấp.
- **Đoạn văn**: giải thích cơ chế liên tục cần ≥3 câu.
- Bảng tổng hợp cuối bài chỉ để **tra cứu nhanh** — phần giải thích chi tiết nằm trong Phần
  tương ứng, không nhét diễn giải dài vào bảng.
- Không biến mọi nội dung thành bảng chỉ vì bảng nhìn gọn.

## 5. Chính xác kỹ thuật

Nguyên tắc quan trọng nhất: **giải thích đơn giản nhưng không sai bản chất**. Một câu dễ
hiểu nhưng làm méo mô hình kỹ thuật là lỗi nghiêm trọng hơn câu hơi dài nhưng đúng.

Kiểm tra bắt buộc với các mô hình dễ sai:

- **Phân bậc rõ**: container runtime (runc, containerd, CRI-O) ≠ container engine/tooling
  (Docker, podman) ≠ container orchestrator (Kubernetes). Không đặt Kubernetes vào nhóm
  container engine.
- **Câu tuyệt đối về isolation** phải nêu namespaces chỉ là một cơ chế — isolation đầy đủ
  còn cần cgroups, capabilities.
- **Giá trị API** ghi chính xác theo docs (vd `imagePullPolicy` chỉ nhận `Always` /
  `IfNotPresent` / `Never`).

## 6. Labs (`labs.md`)

- Mỗi lab: `### Lab N.M — Tên` (N = số bài toàn cục, M = số thứ tự trong bài).
- Dạng checklist `- [ ]` các bước; mỗi bước kèm lệnh cụ thể.
- Kết mỗi lab bằng dòng **Verify:** tiêu chí kiểm chứng kết quả (trạng thái pod, output lệnh).
- Có thể thêm dòng **Ghi nhớ** cho mẹo dễ quên.

## 7. README module

- Mỗi bài một bullet: `- [Bài N — Tên](NN-file.md): <danh sách chủ đề chính> + end-of-lesson lab`.
- Section `## Tiến độ`: checkbox từng bài — tick khi hoàn tất, bỏ marker "đang xử lý".

## 8. Quy ước khác

- Không hardcode secret thật — password trong demo là giá trị giả (vd `password`) và được
  chú thích demo-only.
- Không lặp lại nội dung đã giải thích đầy đủ ở nơi khác — lặp lại chỉ khi phục vụ củng cố
  (giải thích lại ngắn gọn) hoặc tra cứu (bảng tổng hợp).
- Demo ưu tiên `kubectl apply -f` (idempotent); lệnh verify là read-only, chạy lại an toàn.
