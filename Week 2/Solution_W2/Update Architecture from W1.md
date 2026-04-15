## GÓP Ý SỬA/THÊM CHO KIẾN TRÚC DIAGRAM TUẦN 2 (ĐỒNG BỘ VỚI KỊCH BẢN THUYẾT TRÌNH)
Dựa trên hình 3-tier.png hiện tại ở [text](<../../Week 1>) và sự đồng bộ chặt chẽ với kịch bản thuyết trình Tuần 2, chúng ta sẽ có những bổ sung quan trọng như sau:

---

### 1. Phân tách Data Layer với các bucket S3 rõ ràng (Bắt buộc)

* **Vấn đề:** Hiện tại kiến trúc đang gộp chung và chưa thể hiện chiến lược quản lý dữ liệu (giảm Blast Radius và tối ưu Lifecycle).
* **Cách vẽ thêm:** Vẽ rõ một cụm **Amazon S3 Data Layer** thể hiện các bucket sau:
  - **Bucket 1 (Tùy chọn - nếu Front-end không dùng Amplify): Static Assets**. (Host web tĩnh).
  - **Bucket 2: user-media:** Lưu trực tiếp ảnh sản phẩm, avatar, review. Nối đường mũi tên từ Fargate-backend ra bucket này.
  - **Bucket 3: logs-audit:** Lưu audit từ CloudTrail và truy cập từ CloudFront/ALB. 
  - **Bucket 4: backup-archive:** Dùng để backup/snapshot database.

---

### 2. Thiết lập Mã hóa KMS làm cốt lõi (Bắt buộc - Không còn là "Điểm cộng")

* **Vấn đề:** Kịch bản đặc biệt nhấn mạnh dữ liệu phải bọc 2 lớp phân quyền (Quyền S3 + quyền KMS). Nếu bỏ qua, hệ thống sẽ rất rời rạc với lời thuyết trình.
* **Cách vẽ thêm:** Vẽ icon **AWS KMS** ngay hệ thống lưu trữ S3 và RDS. Ghi chú rõ hệ thống sử dụng các Customer Managed Key (CMK) chuyên biệt:
  - `key-media` bọc bucket `user-media`.
  - `key-logs` bọc bucket `logs-audit`.
  - `key-backup` bọc bucket `backup-archive` và RDS Snapshot.

---

### 3. Cụ thể hóa Định danh IAM Roles (Lỗi sẽ bị trừ điểm nặng nhất nếu thiếu)

* **Vấn đề:** Nếu chỉ vẽ chung chung một Role thì không phản ánh được nguyên tắc Least Privilege như kịch bản trình bày.
* **Cách vẽ thêm:** Gắn icon cái khiên có chìa khóa **AWS IAM Role** và note rõ các Role chuyên biệt đã chỉ định:
  - **ECS Task Role:** Chỉ có quyền đọc/ghi vào `user-media` bucket bằng `key-media`.
  - **Logging Role:** Chỉ được mã hóa log ghi vào `logs-audit`.
  - **Backup / DBA Role:** Thực thi quá trình backup và chỉ có Admin/DBA mới có lệnh **Decrypt** bằng `key-backup` để restore hệ thống.

---

### 4. Hệ thống Quan sát và Kiểm toán (Observability)

* **Vấn đề:** Trọng điểm vận hành tuần 2 là phân biệt được Logging hệ thống và Audit API.
* **Cách vẽ thêm:** Cần vẽ nhóm icon Giám sát độc lập lên diagram:
  - **Amazon CloudWatch:** Gắn kết nối thu thập log thời gian thực từ ALB/ECS/RDS (Runtime & Metric alert).
  - **AWS CloudTrail:** Nối bao quát toàn bộ khối AWS kéo thẳng vào bucket `logs-audit` để truy vết hành động, thay đổi cấu hình hạ tầng.

---

### 5. Khẳng định thông tin ổ cứng EBS gp3 cho Database

* **Vấn đề:** Nguyên lý dưới đáy RDS (Managed Service) vẫn là ổ đĩa block storage (EBS).
* **Cách vẽ thêm:** Ngay bên dưới cụm icon RDS (Master & Standby Database), vẽ icon ô vuông hoặc thêm note box: **Storage: EBS gp3**. 
  - Kèm giải thích: *Cân bằng IOPS/Performance và chi phí.*

---

### 6. Ranh giới Security Groups (Tường lửa)

* **Vấn đề:** Chứng minh tính thiết kế mạng "kín kẽ" nhiều lớp của mô hình 3-tier.
* **Cách vẽ thêm:** Hãy vẽ các đường viền nét đứt (Box đứt đoạn, hình ổ khóa) bao quanh từng cụm:
  - **ALB-SG:** Chỉ Allow Inbound Port 80/443 từ Internet.
  - **App-SG (bao quanh Fargate):** Chỉ Allow Inbound của App (vd 8080) hướng mũi tên TỪ ALB-SG.
  - **DB-SG (bao quanh RDS):** Chỉ Allow Inbound (3306) hướng mũi tên TỪ App-SG.
