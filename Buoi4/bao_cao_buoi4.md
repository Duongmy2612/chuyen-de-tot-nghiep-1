BÁO CÁO BUỔI 4 – Dương Diễm My – 2374802010318 – Track DA – Luồng L7
(Các chỗ có [ ] là số/link em phải tự điền sau khi push và sau khi làm thật)

1. Link commit cuối buổi (docs/srs.md + file .drawio): [https://github.com/Duongmy2612/chuyen-de-tot-nghiep-1/tree/main/Buoi4]
2. User Story: 11 story | 3 MUST | 9 tiêu chí GWT (3 ngoại lệ)
3. Use Case: 3 actor | 9 use case | UC đặc tả chi tiết: UC5 – Xác nhận gộp hoặc giữ riêng cặp nghi trùng (6 luồng ngoại lệ: 3a, 3b, 5a, 6a, 8a, 9a)
4. SRS: 6/6 mục | 11 FR có mã | 8 NFR có ngưỡng số | truy vết còn 0 ô trống
5. Mẫu track (Data requirement spec): Xong – còn thiếu: tỉ lệ thiếu từng cột (chờ chạy script profiling trên customers_raw.csv thật)
6. Giờ thực tế xong: User Story [23:19] | Use Case [23:19] | SRS + track [23:19]
7. Checklist đạt [10]/10 – các mục chưa đạt (ghi số mục): [0]
8. Peer review với: [2374802010489 - Nguyễn Hoàng Thuận] (track [SE]) – tóm tắt phạm vi: [Đúng] – góp ý sẽ sửa: [hong có]
9. Chỗ chưa xong/còn phân vân – đã thử gì, kết quả ra sao: Phân vân chọn 3 MUST. Em xếp US1 (nạp + chuẩn hóa số điện thoại), US3 (phát hiện nghi trùng), US5 (xác nhận gộp) là MUST; chuẩn hóa họ tên (US2) để SHOULD vì chỉ hỗ trợ độ chính xác so khớp. Đã thử đưa chuẩn hóa số điện thoại thành story MUST riêng nhưng thành 4 MUST, vượt khuyến nghị 2–3.
10. Một quyết định em đưa ra hôm nay – dựa trên tiêu chí nào đã học: Chọn UC5 làm use case đặc tả chi tiết và đặt US10 (tự động gộp) ở WON'T, vì UC5 là bước duy nhất làm thay đổi hồ sơ đã làm sạch và phạm vi đã duyệt ở Buổi 2 ghi rõ "không tự động gộp khi chưa xác nhận" (tiêu chí: xác định giới hạn phạm vi và use case mang lại giá trị hoàn chỉnh cho actor).
11. Việc còn dở – hạn em tự đặt (trước buổi 5): Chạy profiling điền tỉ lệ thiếu; gán nhãn tay 200 dòng điện thoại và 300 cặp trùng để kiểm NFR-02, NFR-03; vẽ nháp ERD 6 bảng.
12. Tự đánh giá: Đạt hết mục tiêu
