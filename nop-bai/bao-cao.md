# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

<!--
HƯỚNG DẪN - đọc rồi XÓA TOÀN BỘ các khối chú thích này sau khi điền xong:

  - Giới hạn: KHÔNG QUÁ 1 TRANG A4, tương đương khoảng 450 - 550 từ nội dung.
  - Chỉ điền vào các chỗ ___ và các ô trong bảng. Không thêm mục mới.
  - Viết bằng câu hoàn chỉnh, không gạch đầu dòng cụt lủn.
  - Kiểm tra độ dài sau khi đã xóa hết chú thích:
        wc -w nop-bai/bao-cao.md
    và xem trước bản in bằng cách mở file trên GitHub rồi Ctrl+P / Cmd+P.
-->

| | |
|---|---|
| Họ và tên | Bùi Lê Gia Huy |
| MSSV | 2A202602607 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/blgihuy/K4-L3-DAY21-BuiLeGiaHuy-2A202602607-CI-CD-for-AI-Systems |
| Ngày nộp | 07/10/2026 |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.1 | 3 | 0.7109 | 0.8780 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.8460 |
| 3 | 100 | 0.2 | 3 | 0.7290 | 0.8840 |

**Bộ siêu tham số đã chọn:** `n_estimators=100`, `learning_rate=0.2`, `max_depth=3`.

**Lý do:** Bộ siêu tham số ở Lần 3 đạt F1-score cao nhất (0.7290), vượt xa ngưỡng chất lượng bắt buộc 0.65 và vượt trội hơn so với các lần chạy còn lại. Trong bài toán phân loại mất cân bằng, F1-score phản ánh chính xác khả năng nhận diện lớp thiểu số (thu nhập > 50K) thay vì bị phóng đại như accuracy. Đáng chú ý ở Lần 2, dù accuracy vẫn giữ mức cao 0.8460 do tỷ lệ lớp chiếm ưu thế kéo điểm lên, F1-score lại sụt giảm nghiêm trọng xuống 0.6051 (không đạt chuẩn), cho thấy accuracy che giấu tình trạng bỏ sót nhiều mẫu dương. Ngoài ra, quan sát thực nghiệm cho thấy Gradient Boosting đòi hỏi sự bù trừ giữa các vòng boosting: khi giữ `n_estimators=100`, việc tăng `learning_rate` từ 0.1 lên 0.2 giúp các cây sửa lỗi hiệu quả hơn trên tập holdout; ngược lại việc giảm đồng thời cả `learning_rate` lẫn số cây ở Lần 2 khiến mô hình bị underfitting rõ rệt.

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

Tập dữ liệu Adult có phân bố lớp mất cân bằng nghiêm trọng khi lớp thu nhập cao (thu nhập > 50K) chỉ chiếm 24,8% tổng số mẫu. Do đó, một mô hình tầm thường chỉ cần luôn đoán "thu nhập thấp" cho mọi trường hợp vẫn dễ dàng đạt accuracy lên tới 0,752 (75,2%), tạo ra ảo tưởng về hiệu quả trong khi hoàn toàn thất bại trong việc phát hiện đối tượng mục tiêu. F1-score của riêng lớp dương đóng vai trò trung bình điều hòa giữa Precision và Recall, đo lường chính xác khả năng mô hình vừa không bỏ sót đối tượng thu nhập cao vừa không dự đoán sai lệch. Ta tuyệt đối không sử dụng `average="weighted"` hoặc `average="macro"` vì phép tính trung bình sẽ bị lớp đa số (thu nhập <= 50K) chiếm 75,2% kéo điểm số tăng giả tạo, làm mất đi ý nghĩa giám sát nghiêm ngặt của Quality Gate.

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| Lệnh tạo Service Account key bị lỗi FAILED_PRECONDITION khiến `sa-key.json` rỗng | Chính sách bảo mật của GCP tổ chức chặn tạo private key (`disableServiceAccountKeyCreation`) | Tận dụng file Application Default Credentials (`application_default_credentials.json`) để xác thực DVC và nạp vào GitHub Secrets |
| DVC pull trên GitHub Actions báo lỗi 401 Unauthorized | Cấu hình `.dvc/config` tìm `sa-key.json` ở thư mục gốc repo thay vì `/tmp/sa-key.json` | Cập nhật workflow ghi secret ra `sa-key.json` tại thư mục làm việc và bổ sung biến `GOOGLE_CLOUD_PROJECT` |
| Dịch vụ `income-api` trên VM báo lỗi không unpickle được model | VM cài phiên bản `scikit-learn 1.7.2` mới nhất, xung đột với `scikit-learn 1.4.2` lúc huấn luyện | Hạ phiên bản thư viện trên máy ảo về đúng `scikit-learn==1.4.2` để tương thích hoàn toàn |

---

## 4. So Sánh Bước 2 và Bước 3 (bắt buộc, 2 - 3 câu)

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1`) | 0.7290 | 0.8840 |
| Bước 3 (thêm `train_batch2`) | 0.7330 | 0.8820 |

**Nhận xét:** Khi bổ sung 22.361 mẫu dữ liệu mới ở Bước 3 (tổng cộng 44.722 mẫu huấn luyện), F1-score tăng nhẹ từ 0.7290 lên 0.7330 (+0.0040) trong khi accuracy giữ ổn định quanh 0.8820. Do tập dữ liệu mới có cùng nguồn gốc và phân phối với tập ban đầu nên mô hình không có sự đột biến lớn về hiệu năng, nhưng độ tin cậy được nâng cao. Quan trọng nhất, toàn bộ pipeline CI/CD đã tự động phát hiện thay đổi từ file con trỏ DVC, kích hoạt lại quy trình huấn luyện, vượt qua Quality Gate và triển khai thành công mô hình mới lên VM.

---

## 5. Phần Bonus Đã Thực Hiện (nếu có)

<!-- Xóa cả mục 5 nếu không làm bonus. Mỗi bonus tối đa 1 dòng. -->

- [ ] Bonus 1 - Tracking MLflow từ xa với DagsHub: ___
- [ ] Bonus 2 - Điều chỉnh ngưỡng quyết định: ___
- [ ] Bonus 3 - Báo cáo precision / recall tự động: ___
- [ ] Bonus 4 - Hoàn trả về phiên bản trước: ___
- [ ] Bonus 5 - Cảnh báo lệch lạc dữ liệu: ___
