# Data Requirement Specification – L7: Chất lượng dữ liệu khách hàng

Sinh viên: Dương Diễm My – 2374802010318 · Track DA · Đính kèm BT1 (xem `docs/srs.md`).

> **Lưu ý đo trên dữ liệu thật:** các ô `__ %` ở mục 2 phải được điền sau khi chạy script profiling ở Phụ lục A trên mẫu `customers_raw.csv` thật. Danh sách cột nguồn là **giả định theo case study** (Mục 5.2, 8); đối chiếu với header thật của tệp rồi sửa nếu khác.

## 1. Bảng nguồn dữ liệu

| Hệ thống nguồn | Tệp / định dạng | Tần suất cập nhật | Khối lượng ước tính | Ghi chú |
|---|---|---|---|---|
| Hồ sơ khách hàng thô hợp nhất từ Excel từng cửa hàng, Zalo, sổ tay bảo hành (V1) | `customers_raw.csv` (UTF-8) | Nạp theo đợt, theo yêu cầu (đề xuất hằng tháng) | ~65.000 dòng; mẫu con 5.000–8.000 dòng để xử lý | Dữ liệu mô phỏng, cố ý có lỗi; không commit tệp lớn (`data/raw/` nằm trong `.gitignore`), chỉ commit mẫu vài trăm dòng ở `data/sample/` |
| Tệp Excel khách hàng của cửa hàng (nếu bổ sung) | `.xlsx` | Theo đợt | Vài trăm đến vài nghìn dòng/tệp | Cùng cấu trúc cột sau khi ánh xạ |

## 2. Từ điển dữ liệu nguồn (`customers_raw.csv`, giả định)

| Cột | Kiểu | Ý nghĩa | Giá trị hợp lệ | Tỉ lệ thiếu (đo trên mẫu) |
|---|---|---|---|---|
| `full_name` | Văn bản | Họ tên khách | 2–120 ký tự, không chứa chữ số | `__ %` |
| `phone` | Văn bản | Số điện thoại, nhiều dạng: `0901234567`, `84901234567`, `0901 234 567`, `+84…` | Sau chuẩn hóa: `^0\d{9}$` | `__ %` |
| `email` | Văn bản | Thư điện tử | Đúng mẫu email; có thể trống | `__ %` |
| `address` | Văn bản | Địa chỉ liên hệ | ≤ 255 ký tự; có thể trống | `__ %` |
| `source` (nếu có) | Văn bản | Nguồn ghi nhận (cửa hàng / Zalo / sổ bảo hành) | Một trong các nguồn đã biết | `__ %` |
| `created_at` (nếu có) | Văn bản | Thời điểm ghi nhận | Ngày hợp lệ; có thể lẫn 3 định dạng | `__ %` |

Lỗi quan sát được cần phát hiện (theo Bảng 5.2): số điện thoại ba dạng và ô trống; tên hoa/thường lẫn lộn, có dấu và không dấu, khoảng trắng thừa; hồ sơ trùng (ước tính 18–22%).

## 3. Mô hình dữ liệu của luồng (6 bảng)

Bốn bảng đã đăng ký ở Buổi 2 (`stg_customer`, `customer_clean`, `dq_rule`, `dq_result`) cộng hai bảng bổ sung cần cho chức năng xác nhận gộp và theo dõi đợt nạp (`load_batch`, `dup_candidate`). *Mô hình chi tiết (ERD) hoàn thiện ở Buổi 5.*

| Bảng | Cột chính | Ghi chú |
|---|---|---|
| `load_batch` | `batch_id` PK, `file_name`, `file_hash`, `row_count`, `loaded_at`, `loaded_by` | Mỗi lần nạp một dòng; `file_hash` để cảnh báo nạp lại. |
| `stg_customer` | `stg_id` PK, `batch_id` FK, `row_no`, `full_name_raw`, `phone_raw`, `email_raw`, `address_raw`, `phone_norm`, `name_norm`, `name_key`, `flags`, `status` | Giữ nguyên giá trị thô để đối chiếu; `flags` chứa `INVALID_PHONE`, `MISSING_PHONE`, `MISSING_NAME`… |
| `customer_clean` | `customer_id` PK, `full_name`, `phone` UNIQUE NOT NULL, `email`, `address`, `is_active`, `merged_into`, `created_at`, `updated_at` | Bám theo bảng `customer` tham chiếu (Mục 8); thêm `is_active`, `merged_into` cho xóa mềm (QT-13). |
| `dup_candidate` | `pair_id` PK, `record_a`, `record_b`, `match_type` (`EXACT_PHONE`/`FUZZY`), `phone_sim`, `name_sim`, `score`, `status` (`PENDING`/`MERGED`/`KEEP_SEPARATE`), `survivor_id`, `decided_by`, `decided_at`, `note`, `batch_id` | Lưu lịch sử quyết định (NFR-08). |
| `dq_rule` | `rule_id` PK, `rule_code`, `rule_name`, `rule_type`, `field_name`, `weight`, `pass_threshold`, `is_active` | Cấu hình được (FR-07). |
| `dq_result` | `result_id` PK, `batch_id` FK, `rule_id` FK (NULL = dòng tổng), `total_records`, `pass_count`, `pass_rate`, `weighted_score`, `measured_at` | Chỉ số mức độ sạch của đợt = dòng tổng. |

## 4. Quy tắc chất lượng dữ liệu phải đạt (có ngưỡng)

| Mã quy tắc | Loại | Mô tả | Trọng số | Ngưỡng đạt |
|---|---|---|---|---|
| DQ-01 | Completeness | `phone` không trống | 0,25 | ≥ 98% |
| DQ-02 | Validity | `phone_norm` khớp `^0\d{9}$` | 0,25 | ≥ 95% |
| DQ-03 | Completeness | `full_name` không trống | 0,15 | ≥ 99% |
| DQ-04 | Validity | `full_name` 2–120 ký tự, không chứa chữ số | 0,10 | ≥ 97% |
| DQ-05 | Uniqueness | Không trùng theo khóa nghiệp vụ `phone_norm` | 0,20 | ≥ 98% (tức tỉ lệ trùng ≤ 2%) |
| DQ-06 | Validity | `email` đúng mẫu khi có giá trị | 0,05 | ≥ 95% |
| | | **Tổng trọng số** | **1,00** | |

**Công thức:** chỉ số mức độ sạch của một đợt = Σ (trọng số × tỉ lệ đạt quy tắc) × 100.
**Mục tiêu chung:** completeness tổng thể ≥ 95%; không trùng theo khóa nghiệp vụ; chỉ số mức độ sạch sau làm sạch ≥ 90 (loại *Tốt*) và cao hơn chỉ số trước làm sạch ít nhất 10 điểm.

**Ví dụ tính (số liệu minh họa, không phải kết quả đo):** tỉ lệ đạt DQ-01..06 lần lượt 0,92 · 0,85 · 0,98 · 0,95 · 0,80 · 0,90 → 100 × (0,25×0,92 + 0,25×0,85 + 0,15×0,98 + 0,10×0,95 + 0,20×0,80 + 0,05×0,90) = 100 × 0,8195 ≈ **81,95** → loại *Chấp nhận được*.

## 5. Thuật toán so khớp mờ (đặc tả)

1. **Chuẩn bị:** `phone_norm` (QT-02); `name_key` = họ tên bỏ dấu, chữ thường, bỏ khoảng trắng thừa.
2. **Blocking** (tránh so sánh mọi cặp, NFR-01): chỉ so sánh hai hồ sơ nếu cùng 5 số đầu, hoặc cùng 5 số cuối của số điện thoại, hoặc cùng từ cuối của `name_key` (tên gọi).
3. **Độ giống số điện thoại:** 100 nếu bằng nhau; ngược lại `rapidfuzz.fuzz.ratio` trên hai chuỗi 10 chữ số.
4. **Độ giống họ tên:** `rapidfuzz.fuzz.token_sort_ratio` trên `name_key`.
5. **Điểm tương đồng** = 0,6 × độ giống số điện thoại + 0,4 × độ giống họ tên.
6. **Quyết định:** cùng `phone_norm` → cặp `EXACT_PHONE`; ngược lại điểm ≥ 85 → cặp `FUZZY`. Ví dụ: hai số khác đúng 1 chữ số (độ giống 90) và tên giống hệt → 0,6×90 + 0,4×100 = 94 → được đề xuất.
7. Ngưỡng 85 là giá trị khởi đầu, chỉnh theo kết quả precision/recall (NFR-03).

## 6. Câu hỏi phân tích và mức chi tiết báo cáo

| # | Câu hỏi phân tích | Truy vết User Story | Mức chi tiết |
|---|---|---|---|
| Q1 | Mỗi đợt nạp có bao nhiêu % hồ sơ trùng, thiếu, sai định dạng? | US7, US8 | Theo đợt nạp |
| Q2 | Chỉ số mức độ sạch thay đổi thế nào theo thời gian? | US8 | Theo đợt nạp, theo ngày, theo tháng |
| Q3 | Quy tắc nào kéo chỉ số xuống nhiều nhất? | US7, US8 | Theo quy tắc (DQ-01..06) |
| Q4 | Có bao nhiêu cặp nghi trùng đã gộp, giữ riêng, còn chờ xử lý? | US5, US8 | Theo đợt nạp, theo trạng thái |
| Q5 | Phân bố điểm tương đồng và tỉ lệ gộp theo khoảng điểm là bao nhiêu (để chỉnh ngưỡng)? | US3, US5 | Khoảng điểm 85–89, 90–94, 95–100 |
| Q6 | Dạng lỗi số điện thoại nào phổ biến nhất (84…, dấu cách, thiếu số)? | US1, US9 | Theo loại lỗi |

## Phụ lục A – Script profiling tỉ lệ thiếu (chạy để điền mục 2)

```python
import pandas as pd

df = pd.read_csv("data/raw/customers_raw.csv", dtype=str).sample(n=6000, random_state=42)
missing = (df.isna() | (df.apply(lambda s: s.str.strip() == ""))).mean().mul(100).round(2)
print(missing)          # điền vào cột "Tỉ lệ thiếu"
print(df.columns.tolist())  # đối chiếu với từ điển dữ liệu ở mục 2
```
Ghi `random_state=42` (seed) vào README để tái lập.
