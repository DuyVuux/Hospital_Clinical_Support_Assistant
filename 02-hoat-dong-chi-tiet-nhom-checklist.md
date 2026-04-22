# Checklist công việc nhóm - Ngày tổng kết Phase 1 (AICB-P1)

## 1) Mục tiêu cuối ngày
- [ ] Hoàn thành bộ Worksheet buổi sáng (Worksheet 0-5)
- [ ] Hoàn thành mini project proposal (5-7 slide hoặc 1 poster/1-page)
- [ ] Chốt kết luận rõ ràng về deployment, cost, reliability, và hướng Track Phase 2
- [ ] Hoàn thành phiếu chấm peer review cho các đội khác trong zone

## 2) Checklist chuẩn bị đầu ngày
- [ ] Đủ thành viên nhóm (4-6 người)
- [ ] Chọn xong chủ đề hoặc scenario card
- [ ] Cử vai trò: Product lead, Architect, Cost lead, Reliability lead, Presenter
- [ ] Có nơi ghi chép chung (giấy A3 hoặc file chung)

## 3) Buổi sáng - Hoạt động và đầu ra

### 08:00-08:15 | Khởi động và mini reflection
- [ ] Mỗi thành viên viết 3 ý: học được nhiều nhất, điểm yếu nhất, hướng muốn đi sâu
- [ ] Cử 1 người tổng hợp điểm mạnh/điểm yếu chung của nhóm
- [ ] Đầu ra: 3-5 ý tổng hợp hiện trạng năng lực nhóm

### 08:15-08:40 | Hoạt động 1 - Learning Timeline (Worksheet 0)
- [ ] Liệt kê 2-3 kỹ năng nhóm tự tin nhất
- [ ] Mô tả ngắn sản phẩm/agent đã làm hoặc scenario được chọn
- [ ] Chốt 1 chủ đề theo xuyên suốt cả ngày
- [ ] Trả lời: Sản phẩm giải quyết bài toán gì?
- [ ] Trả lời: Ai là người dùng chính?
- [ ] Trả lời: Vì sao chủ đề này phù hợp để phân tích deployment và cost?
- [ ] Đầu ra: Worksheet 0

### 08:40-09:25 | Hoạt động 2 - Enterprise Deployment Clinic (Worksheet 1)
- [ ] Xác định bối cảnh tổ chức/khách hàng sử dụng hệ thống
- [ ] Liệt kê dữ liệu hệ thống sẽ động đến
- [ ] Đánh giá mức độ nhạy cảm của dữ liệu
- [ ] Liệt kê 3 ràng buộc enterprise lớn nhất
- [ ] Chọn mô hình: Cloud / On-prem / Hybrid
- [ ] Viết 2 lý do cho mô hình đã chọn
- [ ] Đầu ra: Worksheet 1

### 09:35-10:15 | Hoạt động 3 - Cost Anatomy Lab (Worksheet 2)
- [ ] Ước lượng user/ngày, request/ngày, peak traffic
- [ ] Ước lượng input/output tokens (nếu dùng LLM API)
- [ ] Liệt kê các lớp cost: token, compute, storage, human review, logging, maintenance
- [ ] Tính sơ bộ cost mức MVP
- [ ] Đánh giá khi user tăng 5x/10x, phần nào tăng mạnh nhất
- [ ] Trả lời: Cost driver lớn nhất là gì?
- [ ] Trả lời: Hidden cost nào dễ bị quên nhất?
- [ ] Trả lời: Ước lượng nào đang quá lạc quan?
- [ ] Đầu ra: Worksheet 2

### 10:15-10:50 | Hoạt động 4 - Cost Optimization Debate (Worksheet 3)
- [ ] Chọn 3 chiến lược optimization phù hợp nhất
- [ ] Với mỗi chiến lược: ghi rõ phần tiết kiệm, lợi ích, trade-off, thời điểm áp dụng
- [ ] Chốt chiến lược làm ngay và chiến lược để sau
- [ ] Đầu ra: Worksheet 3

### 10:50-11:25 | Hoạt động 5 - Scaling & Reliability Tabletop (Worksheet 4)
- [ ] Phân tích 3 tình huống: traffic đột biến, provider timeout, response chậm
- [ ] Với mỗi tình huống: mô tả tác động tới user
- [ ] Ghi phản ứng ngắn hạn và giải pháp dài hạn
- [ ] Chọn metric cần monitor
- [ ] Viết fallback proposal
- [ ] Đầu ra: Worksheet 4

### 11:25-11:45 | Hoạt động 6 - Skills Map & Track Direction (Worksheet 5)
- [ ] Mỗi thành viên tự chấm 1-5 ở 3 mảng: Business/Product, Infra/Data/Ops, AI Engineering/Application
- [ ] Thảo luận điểm mạnh nhóm nằm ở đâu
- [ ] Chọn Track Phase 2 phù hợp nhất
- [ ] Ghi 2-3 kỹ năng cần bù để tiếp tục dự án
- [ ] Đầu ra: Worksheet 5

## 4) Buổi chiều - Mini project và thuyết trình theo zone

### 13:30-13:45 | Chia zone và chốt vai trò
- [ ] Biết zone của nhóm
- [ ] Có presenter và người trả lời Q&A
- [ ] Thống nhất format nộp: slide hoặc poster

### 14:00-15:10 | Sprint 1 - Xây nội dung proposal
- [ ] Hoàn thành Project overview
- [ ] Hoàn thành Enterprise context
- [ ] Hoàn thành Deployment choice
- [ ] Hoàn thành Cost analysis (MVP + Growth)
- [ ] Hoàn thành Optimization plan
- [ ] Hoàn thành Reliability plan
- [ ] Hoàn thành Track recommendation + next step

### 15:20-16:00 | Sprint 2 - Đóng gói bản trình bày
- [ ] Slide 1: Tên dự án, user, bài toán
- [ ] Slide 2: Bối cảnh enterprise và ràng buộc
- [ ] Slide 3: Kiến trúc triển khai đề xuất
- [ ] Slide 4: Cost anatomy
- [ ] Slide 5: Cost optimization
- [ ] Slide 6: Reliability & scaling
- [ ] Slide 7: Track đề xuất và bước đi tiếp theo

### 16:10-17:20 | Zone presentation + peer review
- [ ] Trình bày 5 phút
- [ ] Q&A/phản biện 3 phút
- [ ] Chấm phiếu 2 phút cho các đội khác
- [ ] Không tự chấm đội mình
- [ ] Đặt ít nhất 1 câu hỏi thật cho ít nhất 2 đội khác
- [ ] Mỗi phản biện có đủ: 1 điểm mạnh, 1 điểm cần cải thiện, 1 đề xuất thay thế

## 5) Checklist nộp cuối ngày
- [ ] Worksheet 0-5 của nhóm
- [ ] Slide hoặc poster mini project
- [ ] Phiếu chấm cho các đội khác trong zone
- [ ] Kết luận Track Phase 2 của nhóm

## 6) Team note sheet (điền nhanh)
- [ ] Tên nhóm:
- [ ] Chủ đề:
- [ ] 3 ràng buộc enterprise lớn nhất:
- [ ] Cost driver lớn nhất:
- [ ] 3 chiến lược optimize được chọn:
- [ ] Fallback / reliability plan ngắn gọn:
- [ ] Track Phase 2 đề xuất:
