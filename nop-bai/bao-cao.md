# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

| | |
|---|---|
| Họ và tên | Trần Đăng Nguyên |
| MSSV | 2A202601798 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/nguyendangtran2110-bit/Track2_Day21_2A202601798_TranDangNguyen |
| Ngày nộp | 21/08/2026 |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.1 | 3 | 0.7109 | 0.8780 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.8460 |
| 3 | 200 | 0.1 | 5 | 0.7149 | 0.8740 |

**Bộ siêu tham số đã chọn:** `n_estimators=200`, `learning_rate=0.1`, `max_depth=5`.

**Lý do:** Bộ siêu tham số ở lần chạy 3 được lựa chọn vì đạt chỉ số `f1_score` cao nhất (0.7149), vượt qua ngưỡng chất lượng 0.65 để kích hoạt pipeline triển khai. Lần chạy 2 (`n_estimators=50`, `learning_rate=0.05`, `max_depth=2`) bị underfitting với F1 chỉ đạt 0.6051 do cây quá nông và số vòng boosting chưa đủ bù cho tốc độ học nhỏ. Đáng chú ý, lần chạy 1 có `accuracy` cao nhất (0.8780) nhưng `f1_score` (0.7109) lại thấp hơn lần chạy 3. Điều này phản ánh rõ bài toán mất cân bằng lớp: accuracy bị kéo lên cao bởi việc đoán đúng lớp đa số (thu nhập thấp), trong khi F1 phản ánh chính xác khả năng nhận diện lớp thiểu số quan trọng (thu nhập > 50K). Khi giảm `learning_rate`, ta bắt buộc phải tăng `n_estimators` và tăng `max_depth` để mô hình học được ranh giới phân loại phức tạp.

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

Tập dữ liệu Adult Census Income có sự mất cân bằng lớp rõ rệt khi tỷ lệ người có thu nhập trên 50K USD chỉ chiếm 24,8% (tỷ lệ lớp 75/25). Trong điều kiện này, một mô hình tầm thường luôn dự đoán nhãn "thu nhập thấp" cho mọi mẫu thử vẫn dễ dàng đạt độ chính xác (Accuracy) lên tới 75,2%, nhưng thực tế mô hình đó hoàn toàn vô dụng vì không phát hiện được bất kỳ trường hợp thu nhập cao nào (F1 bằng 0). Chỉ số F1 của lớp dương (target = 1) là trung bình điều hòa giữa Precision và Recall, đo lường chính xác năng lực nhận diện đúng người có thu nhập cao mà không bị áp đảo bởi số lượng lớn của lớp đa số. Khi đánh giá, ta tuyệt đối không dùng `average="weighted"` hay `average="macro"` vì các trọng số này sẽ bị lớp đa số kéo điểm lên cao giả tạo, làm mất đi ý nghĩa giám sát nghiêm ngặt của Quality Gate trong môi trường sản xuất.

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| Lỗi build dependency khi cài thư viện trên Python 3.13 | `scikit-learn` 1.4.2 chưa có binary wheel tương thích cho Python 3.13 trên Windows | Tạo lại môi trường ảo bằng Python 3.12 để cài đặt mượt mà các gói phụ thuộc |
| Lỗi cấp quyền Service Account trên Cloud Storage | Lệnh phân quyền thực thi quá nhanh khi tài khoản IAM vừa tạo chưa kịp đồng bộ toàn cầu | Cấp quyền `roles/storage.objectAdmin` trực tiếp trên bucket sau khi tài khoản đã sẵn sàng |
| Lỗi không nạp được model trên VM và Health check timeout | VM ban đầu cài `scikit-learn` 1.7.2 gây lệch phiên bản unpickle và Uvicorn cần 8-10s để tải model | Cài đúng `scikit-learn==1.4.2` trên VM và thêm vòng lặp retry loop 30s cho bước kiểm tra sức khỏe |

---

## 4. So Sánh Bước 2 và Bước 3

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1`) | 0.7149 | 0.8740 |
| Bước 3 (thêm `train_batch2`) | 0.7354 | 0.8820 |

**Nhận xét:** Khi bổ sung thêm 22.361 mẫu dữ liệu mới ở Bước 3 (tổng cộng 44.722 mẫu), điểm F1 tăng từ 0.7149 lên 0.7354 và Accuracy tăng từ 0.8740 lên 0.8820. Việc bổ sung lượng dữ liệu lớn giúp mô hình GradientBoosting bao quát tốt hơn các trường hợp thiểu số phức tạp. Tuy nhiên, giá trị cốt lõi ở Bước 3 là việc kiểm chứng toàn bộ chu trình Huấn luyện liên tục (Continuous Training) vận hành tự động: chỉ với một thao tác push dữ liệu DVC, pipeline CI/CD đã tự động huấn luyện lại, vượt qua Quality Gate và cập nhật mô hình mới lên server mà không cần bất kỳ can thiệp thủ công nào.

