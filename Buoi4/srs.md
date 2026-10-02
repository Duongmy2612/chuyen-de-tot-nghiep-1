# SRS – Smart CRM Mekong Mobile · L7: Chất lượng dữ liệu khách hàng

| Mục | Nội dung |
|---|---|
| Sinh viên | Dương Diễm My – MSSV 2374802010318 |
| Track | DA (Data Analytics) |
| Học phần | Chuyên đề Tốt nghiệp 1 – Trường ĐH Văn Lang |
| Luồng nghiệp vụ | L7 – Chất lượng dữ liệu khách hàng |
| Phiên bản | 1.0 (Buổi 4) |
| Tài liệu liên quan | `docs/usecase_L7.drawio`, `docs/data-requirement-spec.md` |

**Phạm vi một câu:** Nạp tệp khách hàng thô, chuẩn hóa số điện thoại và họ tên, phát hiện cặp hồ sơ nghi trùng bằng so khớp mờ để nhân viên Marketing xác nhận gộp hoặc giữ riêng, rồi đo và báo cáo chỉ số mức độ sạch dữ liệu theo từng đợt nạp.

---

## 1. Giới thiệu

### 1.1. Mục đích
Tài liệu đặc tả yêu cầu cho luồng L7 của hệ thống Smart CRM Mekong Mobile: phát hiện hồ sơ trùng, thiếu, sai định dạng trong dữ liệu khách hàng; chuẩn hóa dữ liệu; đo và báo cáo mức độ sạch theo thời gian. Luồng giải quyết vấn đề **V1** của case study (khách hàng nằm rải rác ở Excel, Zalo, sổ tay; ước tính 18–22% hồ sơ trùng).

### 1.2. Phạm vi

**Làm**
- Nạp dữ liệu khách hàng thô (CSV/Excel) vào bảng tạm `stg_customer`.
- Chuẩn hóa số điện thoại (QT-02) và họ tên.
- Phát hiện cặp nghi trùng bằng so khớp mờ, tính điểm tương đồng.
- Cho nhân viên Marketing xem, xác nhận gộp hoặc giữ riêng; cập nhật `customer_clean`.
- Cấu hình bộ quy tắc chất lượng dữ liệu (`dq_rule`), tính và lưu chỉ số mức độ sạch (`dq_result`) theo đợt nạp.
- Báo cáo mức độ sạch theo thời gian (dashboard).

**Không làm (ngoài phạm vi)**
- Không xử lý dữ liệu đơn hàng, bảo hành, tồn kho linh kiện; chỉ dữ liệu khách hàng.
- Không tự động gộp hồ sơ khi chưa có xác nhận của nhân viên.
- Không xây dựng mô hình dự đoán khách hàng rời bỏ (luồng L9).
- Không gửi tin khuyến mãi / chiến dịch marketing thực tế.

### 1.3. Bảng thuật ngữ
Mỗi khái niệm chỉ dùng **một tên** trong SRS, sơ đồ Use Case và wireframe.

| Thuật ngữ | Định nghĩa | Tên kỹ thuật |
|---|---|---|
| Hồ sơ khách hàng | Một bản ghi mô tả một khách hàng (họ tên, số điện thoại, email, địa chỉ). | `customer` |
| Tệp khách hàng thô | Tệp CSV/Excel chứa hồ sơ khách hàng chưa làm sạch (nguồn mẫu: `customers_raw.csv`). | – |
| Đợt nạp | Một lần nạp một tệp khách hàng thô vào hệ thống; có mã và thời điểm. | `load_batch` / `batch_id` |
| Bảng tạm | Nơi lưu dữ liệu thô sau khi nạp, trước khi làm sạch. | `stg_customer` |
| Hồ sơ đã làm sạch | Hồ sơ đã chuẩn hóa, số điện thoại hợp lệ và duy nhất. | `customer_clean` |
| Chuẩn hóa | Đưa số điện thoại về dạng `0xxxxxxxxx` và họ tên về dạng thống nhất. | `phone_norm`, `name_norm` |
| Cặp nghi trùng | Hai hồ sơ khách hàng có khả năng là một người. Mỗi hồ sơ trong cặp gọi là hồ sơ nghi trùng. | `dup_candidate` |
| Điểm tương đồng | Số từ 0–100 đo mức giống nhau giữa hai hồ sơ trong cặp nghi trùng. | `score` |
| Hồ sơ gốc | Hồ sơ được giữ lại sau khi gộp; hồ sơ còn lại bị ngừng sử dụng. | `survivor` |
| Gộp | Hợp nhất hai hồ sơ thành một (có xác nhận của nhân viên). | `MERGED` |
| Giữ riêng | Quyết định coi hai hồ sơ là hai khách hàng khác nhau. | `KEEP_SEPARATE` |
| Quy tắc chất lượng dữ liệu | Một tiêu chí đánh giá bản ghi, có trọng số và ngưỡng. | `dq_rule` |
| Chỉ số mức độ sạch | Điểm 0–100 tổng hợp từ các quy tắc chất lượng dữ liệu cho một đợt nạp. | `dq_result` |
| Ngừng sử dụng | Xóa mềm: đánh dấu không dùng, giữ lịch sử (QT-13). | `is_active = false` |

### 1.4. Tài liệu tham chiếu
- Case study Smart CRM – Mekong Mobile (Mục 2.3, 3, 5.2, 7-L7, 8, 9, 11).
- Bài thực hành 1 – Phiếu phạm vi đã duyệt (Buổi 2).

---

## 2. Mô tả tổng quan

### 2.1. Bối cảnh và vấn đề
Hiện có khoảng 65.000 hồ sơ khách hàng ghi nhận rời rạc. Một khách có thể tồn tại nhiều lần với nhiều số điện thoại (V1). Marketing (anh Khoa) không biết ai là khách tốt và gửi tin tràn lan (V5). Dữ liệu mẫu cố ý chứa lỗi: số điện thoại ba dạng (`0901234567`, `84901234567`, `0901 234 567`), ô trống, tên hoa/thường lẫn lộn, có dấu và không dấu, khoảng trắng thừa (Bảng 5.2).

### 2.2. Actor

| Actor | Loại | Vai trò trong L7 |
|---|---|---|
| Nhân viên Marketing | Người dùng chính | Nạp tệp, chạy phát hiện, xem và xác nhận cặp nghi trùng, xem báo cáo, xuất danh sách hồ sơ lỗi. |
| Quản lý cửa hàng | Người dùng | Cấu hình quy tắc chất lượng dữ liệu, xem báo cáo mức độ sạch, xuất danh sách hồ sơ lỗi để cửa hàng sửa tại nguồn. |
| Tệp nguồn Excel/CSV | Hệ thống ngoài (actor phụ) | Cung cấp dữ liệu khách hàng thô cho UC1. |

### 2.3. Luồng nghiệp vụ tổng thể (bắt đầu → kết thúc)
1. Nhân viên Marketing nạp tệp khách hàng thô → tạo một đợt nạp, ghi vào `stg_customer`.
2. Hệ thống chuẩn hóa số điện thoại và họ tên; gắn cờ hồ sơ thiếu hoặc sai định dạng.
3. Hồ sơ có số điện thoại hợp lệ và chưa tồn tại được đưa vào `customer_clean`. Hồ sơ trùng số điện thoại với hồ sơ có sẵn không được tạo mới mà thành cặp nghi trùng (QT-01).
4. Nhân viên Marketing chạy phát hiện: hệ thống so khớp mờ, tạo các cặp nghi trùng kèm điểm tương đồng.
5. Nhân viên xem danh sách và quyết định từng cặp: **Gộp** hoặc **Giữ riêng**; hệ thống cập nhật `customer_clean` và ghi lịch sử.
6. Hệ thống đánh giá đợt nạp theo `dq_rule`, lưu `dq_result`.
7. Quản lý cửa hàng / Marketing xem báo cáo mức độ sạch theo thời gian. *Kết thúc.*

### 2.4. Ràng buộc và giả định
- Stack: Python (pandas, rapidfuzz), PostgreSQL, dashboard Streamlit hoặc Power BI / Looker Studio.
- Làm việc trên mẫu con 5.000–8.000 dòng của `customers_raw.csv` (~65.000 dòng, dữ liệu mô phỏng).
- Hệ thống dạng prototype chạy cục bộ; không tích hợp hệ thống thật của doanh nghiệp.
- Giả định: tệp nguồn có tối thiểu hai cột `full_name`, `phone`; các cột `email`, `address` có thể thiếu.
- Hồ sơ không có số điện thoại hợp lệ **không** vào `customer_clean` mà ở lại `stg_customer` kèm cờ lỗi, và được liệt kê trong báo cáo hồ sơ lỗi (UC9).

### 2.5. Thực thể dữ liệu (chi tiết ở `data-requirement-spec.md`)
`load_batch`, `stg_customer`, `customer_clean`, `dup_candidate`, `dq_rule`, `dq_result`.

---

## 3. Yêu cầu chức năng

### 3.1. Danh sách User Story

Vai trò, mục tiêu, giá trị nối trực tiếp về vấn đề V1/V5 ở Bảng 2.2 của case study.

| ID | User Story | MoSCoW |
|---|---|---|
| US1 | Là **Nhân viên Marketing**, tôi muốn **nạp tệp khách hàng thô (CSV/Excel) vào hệ thống và nhận số điện thoại đã ở dạng chuẩn `0xxxxxxxxx`** để **không phải sửa tay từng dòng dữ liệu rải rác (V1)**. | **MUST** |
| US2 | Là **Nhân viên Marketing**, tôi muốn **họ tên khách được chuẩn hóa về một dạng thống nhất** để **so khớp mờ cho kết quả chính xác hơn và danh sách gọi tên khách đúng, đẹp**. | SHOULD |
| US3 | Là **Nhân viên Marketing**, tôi muốn **hệ thống tự phát hiện các cặp hồ sơ nghi trùng và cho điểm tương đồng** để **không phải dò thủ công 18–22% hồ sơ trùng (V1)**. | **MUST** |
| US4 | Là **Nhân viên Marketing**, tôi muốn **xem danh sách cặp nghi trùng sắp theo điểm tương đồng giảm dần** để **xử lý các cặp có khả năng trùng cao nhất trước**. | SHOULD |
| US5 | Là **Nhân viên Marketing**, tôi muốn **xác nhận từng cặp nghi trùng là gộp hoặc giữ riêng** để **hồ sơ khách hàng được hợp nhất đúng mà không bị gộp nhầm hai người khác nhau**. | **MUST** |
| US6 | Là **Quản lý cửa hàng**, tôi muốn **cấu hình trọng số và ngưỡng của các quy tắc chất lượng dữ liệu** để **tiêu chí đánh giá phù hợp từng giai đoạn mà không cần sửa mã**. | SHOULD |
| US7 | Là **Quản lý cửa hàng**, tôi muốn **chỉ số mức độ sạch được tính và lưu cho từng đợt nạp** để **biết một đợt dữ liệu đạt hay chưa đạt chất lượng**. | SHOULD |
| US8 | Là **Quản lý cửa hàng**, tôi muốn **xem báo cáo mức độ sạch dữ liệu theo thời gian** để **biết chất lượng dữ liệu đang cải thiện hay xấu đi, thay vì chờ tổng hợp thủ công (V4)**. | SHOULD |
| US9 | Là **Quản lý cửa hàng**, tôi muốn **xuất danh sách hồ sơ thiếu hoặc sai định dạng ra tệp CSV** để **cửa hàng sửa tại nguồn và lỗi không lặp lại ở đợt sau**. | COULD |
| US10 | Là **Nhân viên Marketing**, tôi muốn **hệ thống tự động gộp các cặp có điểm tương đồng rất cao mà không cần xác nhận** để **tiết kiệm thời gian xử lý**. | **WON'T** (loại khỏi phạm vi: gộp nhầm khách khó hoàn tác, xem QL7-01) |
| US11 | Là **Quản lý cửa hàng**, tôi muốn **số điện thoại khách bị che với nhân viên thông thường** để **giảm rủi ro lộ dữ liệu cá nhân (QT-15)**. | SHOULD |

### 3.1.1. Kiểm tra INVEST và lỗi thường gặp

| ID | I | N | V | E | S | T | Ghi chú xử lý |
|---|---|---|---|---|---|---|---|
| US1 | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | Chuẩn hóa số điện thoại là kết quả bắt buộc của bước nạp (QT-02) nên nằm trong cùng story; chuẩn hóa họ tên tách riêng ở US2. |
| US2 | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | Sửa lỗi "mơ hồ": quy tắc tên được định nghĩa cụ thể ở FR-03. |
| US3 | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | Công thức điểm và ngưỡng nêu ở FR-04, kiểm chứng được bằng tập cặp gán nhãn tay. |
| US4 | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | Phụ thuộc dữ liệu từ US3, kiểm thử được bằng dữ liệu cặp mẫu. |
| US5 | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | "Gộp hoặc giữ riêng" là hai nhánh của **một** quyết định, không phải hai yêu cầu. |
| US6 | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | – |
| US7 | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | – |
| US8 | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | Mức chi tiết: theo đợt nạp, ngày, quy tắc. |
| US9 | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | – |
| US10 | ✔ | – | ✔ | – | ✔ | ✔ | Nêu **giải pháp** thay vì vấn đề, và mâu thuẫn với phạm vi đã duyệt; giữ lại để ghi nhận quyết định WON'T. |
| US11 | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | – |

### 3.1.2. Tiêu chí chấp nhận (Given – When – Then) cho story MUST

**US1 – Nạp tệp khách hàng thô và chuẩn hóa số điện thoại**
- **AC1.1 (luồng chính)**: *Given* một tệp CSV hợp lệ có cột `full_name` và `phone`, 5.000 dòng; *When* nhân viên chọn tệp và bấm nạp; *Then* hệ thống tạo một đợt nạp mới và ghi đủ 5.000 dòng vào `stg_customer` kèm `batch_id`, báo số dòng đã nạp.
- **AC1.2 (chuẩn hóa)**: *Given* các số điện thoại dạng `+84901234567`, `84901234567`, `0901 234 567`, `0901.234.567`; *When* đợt nạp được xử lý; *Then* `phone_norm` của cả bốn dòng đều là `0901234567`; dòng không quy về được 10 chữ số bắt đầu bằng 0 được gắn cờ `INVALID_PHONE`.
- **AC1.3 (ngoại lệ)**: *Given* tệp thiếu cột bắt buộc `phone` hoặc sai định dạng tệp; *When* nhân viên bấm nạp; *Then* hệ thống từ chối cả đợt, thông báo rõ tên cột thiếu, và `stg_customer` không thay đổi.

**US3 – Phát hiện cặp nghi trùng**
- **AC3.1**: *Given* hai hồ sơ có `phone_norm` giống nhau; *When* nhân viên chạy phát hiện; *Then* hệ thống tạo một cặp nghi trùng loại `EXACT_PHONE` với trạng thái `PENDING` và điểm tương đồng.
- **AC3.2**: *Given* hai hồ sơ có số điện thoại khác nhau đúng một chữ số và họ tên giống nhau sau khi bỏ dấu; *When* chạy phát hiện; *Then* cặp được tạo với điểm tương đồng ≥ 85 (ngưỡng mặc định).
- **AC3.3 (ngoại lệ)**: *Given* một hồ sơ không có số điện thoại; *When* chạy phát hiện; *Then* hệ thống không đưa hồ sơ đó vào so khớp, gắn cờ `MISSING_PHONE`, và không tạo cặp chỉ dựa trên tên giống nhau.

**US5 – Xác nhận gộp hoặc giữ riêng**
- **AC5.1 (gộp)**: *Given* một cặp `PENDING`; *When* nhân viên chọn "Gộp", chọn hồ sơ gốc và xác nhận; *Then* `customer_clean` còn một hồ sơ hiệu lực (trường trống được bổ sung từ hồ sơ còn lại), hồ sơ còn lại `is_active = false`, cặp chuyển `MERGED`, lịch sử ghi người thực hiện và thời điểm.
- **AC5.2 (giữ riêng)**: *Given* một cặp `PENDING` có hai số điện thoại khác nhau; *When* nhân viên chọn "Giữ riêng"; *Then* cặp chuyển `KEEP_SEPARATE`, hai hồ sơ không đổi, cặp không được đề xuất lại ở các lần phát hiện sau.
- **AC5.3 (ngoại lệ)**: *Given* một cặp `PENDING` và nhân viên chưa quyết định; *When* đợt phát hiện kết thúc hoặc điểm tương đồng là 100; *Then* hệ thống **không** tự động gộp, cặp giữ nguyên `PENDING` (QL7-01).

### 3.2. Use Case Diagram
File gốc: **`docs/usecase_L7.drawio`** (mở bằng draw.io; xuất PNG để chèn báo cáo nếu cần).

Ranh giới hệ thống: *Hệ thống Smart CRM – Mekong Mobile · L7 Chất lượng dữ liệu khách hàng*. Actor đều nằm ngoài ranh giới.

| UC | Tên use case | Actor | Quan hệ |
|---|---|---|---|
| UC1 | Nạp dữ liệu khách hàng thô | Nhân viên Marketing; Tệp nguồn Excel/CSV | «include» UC2, «include» UC7 |
| UC2 | Chuẩn hóa số điện thoại và họ tên | (do UC1 gọi) | – |
| UC3 | Phát hiện hồ sơ nghi trùng | Nhân viên Marketing | – |
| UC4 | Xem danh sách cặp nghi trùng | Nhân viên Marketing | – |
| UC5 | Xác nhận gộp hoặc giữ riêng cặp nghi trùng | Nhân viên Marketing | «include» UC4 |
| UC6 | Cấu hình quy tắc chất lượng dữ liệu | Quản lý cửa hàng | – |
| UC7 | Tính chỉ số mức độ sạch dữ liệu | (do UC1 gọi) | – |
| UC8 | Xem báo cáo mức độ sạch theo thời gian | Nhân viên Marketing; Quản lý cửa hàng | – |
| UC9 | Xuất danh sách hồ sơ lỗi | Nhân viên Marketing; Quản lý cửa hàng | – |

### 3.3. Đặc tả chi tiết UC5 – Xác nhận gộp hoặc giữ riêng cặp nghi trùng
(Chọn UC5 vì đây là use case quan trọng nhất: bước duy nhất làm thay đổi hồ sơ đã làm sạch và là điểm kiểm soát "không tự động gộp".)

| Mục | Nội dung |
|---|---|
| Mã / Tên | UC5 – Xác nhận gộp hoặc giữ riêng cặp nghi trùng |
| Actor chính | Nhân viên Marketing |
| Mục tiêu | Quyết định một cặp nghi trùng là cùng một khách hàng (gộp) hoặc khác nhau (giữ riêng) và cập nhật hồ sơ đã làm sạch. |
| Điều kiện trước | Đã đăng nhập; UC3 đã chạy; có ít nhất một cặp ở trạng thái `PENDING`. |
| Điều kiện sau (thành công) | Cặp ở trạng thái `MERGED` hoặc `KEEP_SEPARATE`; `customer_clean` cập nhật đúng; lịch sử ghi người, thời điểm, điểm tương đồng. Không có hồ sơ nào bị xóa vật lý (QT-13). |
| Điều kiện sau (thất bại) | Dữ liệu không đổi, cặp vẫn `PENDING`. |
| Quy tắc liên quan | QT-01, QT-13, QT-15, QL7-01 |

**Luồng chính (gộp)**
1. Nhân viên mở danh sách cặp nghi trùng (UC4) và chọn một cặp.
2. Hệ thống hiển thị hai hồ sơ cạnh nhau (tên, số điện thoại che theo QT-15, email, địa chỉ), điểm tương đồng và các trường khác nhau.
3. Nhân viên chọn quyết định **Gộp**.
4. Hệ thống yêu cầu chọn hồ sơ gốc.
5. Nhân viên chọn hồ sơ gốc.
6. Hệ thống hiển thị bản xem trước hồ sơ sau gộp (giữ giá trị của hồ sơ gốc, bổ sung trường trống từ hồ sơ còn lại).
7. Nhân viên bấm **Xác nhận**.
8. Hệ thống kiểm tra số điện thoại sau gộp vẫn duy nhất trong `customer_clean` (QT-01).
9. Hệ thống cập nhật hồ sơ gốc, đặt hồ sơ còn lại `is_active = false` và `merged_into = <hồ sơ gốc>` (QT-13), chuyển cặp sang `MERGED`, ghi lịch sử, kích hoạt tính lại chỉ số mức độ sạch (UC7).
10. Hệ thống thông báo thành công và quay lại danh sách. *Kết thúc.*

**Luồng ngoại lệ (đánh số theo bước)**
- **3a. Nhân viên chọn "Giữ riêng"**: hệ thống chuyển cặp sang `KEEP_SEPARATE`, ghi lý do (nếu có) và loại cặp khỏi danh sách `PENDING`; use case kết thúc thành công. *Riêng cặp `EXACT_PHONE`*: hai hồ sơ cùng số điện thoại không thể cùng tồn tại trong `customer_clean` (QT-01), nên hệ thống yêu cầu nhập số điện thoại hợp lệ khác cho một hồ sơ; nếu không có, hệ thống từ chối "Giữ riêng" và quay lại bước 3.
- **3b. Nhân viên chọn "Để sau"**: cặp giữ `PENDING`, quay lại danh sách.
- **5a. Hai hồ sơ có số điện thoại khác nhau**: hệ thống yêu cầu chọn số điện thoại chính cho hồ sơ gộp; số còn lại được lưu vào lịch sử gộp (không mất thông tin). Tiếp tục bước 6.
- **6a. Cặp đã được xử lý bởi người khác trong lúc xem**: hệ thống báo "Cặp này đã được xử lý", làm mới danh sách, quay lại bước 1.
- **8a. Số điện thoại sau gộp trùng với một hồ sơ thứ ba trong `customer_clean`**: hệ thống từ chối gộp, hiển thị hồ sơ thứ ba và gợi ý xử lý cặp liên quan trước; quay lại bước 1.
- **9a. Lỗi ghi cơ sở dữ liệu**: hệ thống hoàn tác toàn bộ giao dịch (rollback), thông báo lỗi, cặp giữ `PENDING`, quay lại bước 1.

### 3.4. Danh sách yêu cầu chức năng (FR)

| Mã | Yêu cầu chức năng | Ưu tiên |
|---|---|---|
| FR-01 | Hệ thống cho phép nạp tệp CSV/Excel vào `stg_customer`, mỗi lần nạp tạo một `batch_id`; từ chối cả đợt nếu thiếu cột bắt buộc `full_name`, `phone`; cảnh báo khi nạp lại tệp đã nạp (cùng giá trị băm). | MUST |
| FR-02 | Hệ thống chuẩn hóa số điện thoại về 10 chữ số bắt đầu bằng 0 (QT-02): bỏ dấu cách, dấu chấm, dấu gạch; đổi `+84`/`84` đầu thành `0`; gắn cờ `INVALID_PHONE` / `MISSING_PHONE` khi không hợp lệ. | MUST |
| FR-03 | Hệ thống chuẩn hóa họ tên: bỏ khoảng trắng thừa, viết hoa chữ cái đầu mỗi từ; giữ nguyên dấu tiếng Việt; tạo thêm khóa tên không dấu dùng cho so khớp. | SHOULD |
| FR-04 | Hệ thống phát hiện cặp nghi trùng bằng so khớp mờ; điểm tương đồng = 0,6 × độ giống số điện thoại + 0,4 × độ giống họ tên (không dấu); cặp có điểm ≥ ngưỡng (mặc định 85) hoặc cùng số điện thoại chuẩn hóa được ghi vào `dup_candidate` trạng thái `PENDING`. | MUST |
| FR-05 | Hệ thống hiển thị danh sách cặp nghi trùng sắp theo điểm giảm dần, lọc theo trạng thái và khoảng điểm, xem hai hồ sơ cạnh nhau. | SHOULD |
| FR-06 | Hệ thống cho phép xác nhận **Gộp** hoặc **Giữ riêng** từng cặp; gộp cập nhật `customer_clean`, ngừng sử dụng hồ sơ còn lại, ghi lịch sử; **không bao giờ tự động gộp**. | MUST |
| FR-07 | Hệ thống cho phép thêm/sửa/tắt quy tắc `dq_rule` (loại kiểm tra, trọng số, ngưỡng đạt); tổng trọng số các quy tắc đang bật phải bằng 1,0. | SHOULD |
| FR-08 | Hệ thống tính tỉ lệ đạt từng quy tắc và chỉ số mức độ sạch của từng đợt nạp, lưu vào `dq_result`; tính lại khi có gộp. | SHOULD |
| FR-09 | Hệ thống hiển thị báo cáo mức độ sạch theo thời gian: xu hướng theo đợt nạp, theo ngày; phân rã theo quy tắc; số cặp nghi trùng theo trạng thái. | SHOULD |
| FR-10 | Hệ thống xuất danh sách hồ sơ có cờ lỗi (`MISSING_PHONE`, `INVALID_PHONE`, thiếu họ tên) của một đợt nạp ra tệp CSV. | COULD |
| FR-11 | Hệ thống che số điện thoại (ví dụ `090****567`) với mọi vai trò trừ Quản lý và Ban giám đốc (QT-15). | SHOULD |

---

## 4. Yêu cầu phi chức năng (có ngưỡng đo được)

| Mã | Nhóm | Yêu cầu | Cách kiểm chứng |
|---|---|---|---|
| NFR-01 | Hiệu năng | Nạp và chuẩn hóa tệp 8.000 dòng trong ≤ 60 giây; chạy phát hiện cặp nghi trùng trên 8.000 hồ sơ trong ≤ 120 giây (dùng kỹ thuật blocking, không so sánh toàn bộ cặp). | Đo thời gian chạy trên máy phát triển, 3 lần, lấy lớn nhất. |
| NFR-02 | Độ chính xác chuẩn hóa | Số điện thoại hợp lệ được chuẩn hóa đúng ≥ 99% trên tập kiểm thử 200 dòng gán nhãn tay, bao gồm cả bốn dạng lỗi ở Bảng 5.2. | Test tự động so với nhãn tay. |
| NFR-03 | Chất lượng phát hiện trùng | Trên tập 300 cặp gán nhãn tay: precision ≥ 85% và recall ≥ 80% ở ngưỡng điểm mặc định 85. | Tính precision/recall; chỉnh ngưỡng nếu chưa đạt. |
| NFR-04 | Toàn vẹn dữ liệu | 100% số điện thoại trong `customer_clean` đúng mẫu `^0\d{9}$` và duy nhất (QT-01, QT-02); 0 hồ sơ bị xóa vật lý (QT-13). | Truy vấn kiểm tra sau mỗi đợt nạp. |
| NFR-05 | Tái lập | Nạp lại cùng tệp và cùng cấu hình cho kết quả giống nhau 100% (số cặp nghi trùng, chỉ số mức độ sạch); mọi bước ngẫu nhiên đặt `seed` và ghi vào README. | So sánh kết quả hai lần chạy. |
| NFR-06 | Hiệu năng báo cáo | Dashboard tải xong ≤ 5 giây với ≤ 24 đợt nạp. | Đo thời gian tải. |
| NFR-07 | Bảo mật | 0 thông tin bí mật (mật khẩu, khóa) trong repo; thông tin kết nối chỉ trong `.env` (không commit); số điện thoại hiển thị dạng che với vai trò không phải Quản lý/Ban giám đốc (QT-15). | Quét repo; kiểm tra giao diện theo vai trò. |
| NFR-08 | Truy vết | 100% thao tác gộp/giữ riêng có bản ghi lịch sử gồm người thực hiện, thời điểm, điểm tương đồng. | Đối chiếu `dup_candidate` với thao tác. |

---

## 5. Quy tắc nghiệp vụ

**Quy tắc của case study áp dụng cho L7**

| Mã | Quy tắc | Áp dụng ở |
|---|---|---|
| QT-01 | Số điện thoại khách hàng duy nhất. Khi số đã tồn tại, hiển thị hồ sơ có sẵn thay vì tạo hồ sơ mới. | FR-02, FR-04, FR-06, UC5 (8a) |
| QT-02 | Số điện thoại chuẩn hóa về 10 chữ số bắt đầu bằng 0; các dạng `+84…`, `84…`, dấu cách, dấu chấm đều quy về dạng chuẩn. | FR-02, US1 |
| QT-13 | Không xóa vật lý hồ sơ khách hàng; chỉ ngừng sử dụng, giữ lịch sử. | FR-06, NFR-04 |
| QT-14 | Nhân viên chỉ xem dữ liệu đơn vị mình; quản lý xem toàn bộ đơn vị phụ trách. | Phạm vi xem báo cáo (FR-09) |
| QT-15 | Số điện thoại hiển thị dạng che với mọi vai trò trừ Quản lý và Ban giám đốc. | FR-11, UC5 |

**Quy tắc riêng của luồng L7**

| Mã | Quy tắc |
|---|---|
| QL7-01 | Không tự động gộp hồ sơ trong bất kỳ trường hợp nào (kể cả điểm = 100); mọi lần gộp cần xác nhận của nhân viên. |
| QL7-02 | Cặp nghi trùng khi điểm tương đồng ≥ ngưỡng (mặc định 85, cấu hình được) hoặc cùng số điện thoại chuẩn hóa. |
| QL7-03 | Điểm tương đồng = 0,6 × độ giống số điện thoại + 0,4 × độ giống họ tên. Độ giống số điện thoại = 100 nếu bằng nhau, ngược lại là tỉ lệ giống chuỗi 10 chữ số. Độ giống họ tên tính trên tên không dấu, không phân biệt hoa/thường, không phụ thuộc thứ tự từ. |
| QL7-04 | Hồ sơ không có số điện thoại hợp lệ chỉ ở lại `stg_customer` kèm cờ lỗi, không vào `customer_clean`, không tham gia so khớp. |
| QL7-05 | Cặp đã `KEEP_SEPARATE` không được đề xuất lại. |
| QL7-06 | Chỉ số mức độ sạch của đợt = Σ (trọng số quy tắc × tỉ lệ bản ghi đạt quy tắc) × 100; tổng trọng số các quy tắc đang bật bằng 1,0. Phân loại: ≥ 90 Tốt; 75 đến dưới 90 Chấp nhận được; dưới 75 Kém. |
| QL7-07 | Khi gộp hai hồ sơ có số điện thoại khác nhau, bắt buộc chọn số chính; số còn lại lưu vào lịch sử gộp. |

---

## 6. Bảng truy vết

| FR | User Story | Use Case | MoSCoW |
|---|---|---|---|
| FR-01 | US1 | UC1 | MUST |
| FR-02 | US1 | UC1, UC2 | MUST |
| FR-03 | US2 | UC2 | SHOULD |
| FR-04 | US3 | UC3 | MUST |
| FR-05 | US4 | UC4 | SHOULD |
| FR-06 | US5 | UC5 | MUST |
| FR-06 (ràng buộc, nêu ở QL7-01) | US10 | UC5 | WON'T |
| FR-07 | US6 | UC6 | SHOULD |
| FR-08 | US7 | UC7 | SHOULD |
| FR-09 | US8 | UC8 | SHOULD |
| FR-10 | US9 | UC9 | COULD |
| FR-11 | US11 | UC4, UC5 | SHOULD |

Kiểm tra truy vết ngược: UC1–UC9 đều xuất hiện ít nhất một lần ở cột Use Case; US1–US11 đều xuất hiện ở cột User Story.
