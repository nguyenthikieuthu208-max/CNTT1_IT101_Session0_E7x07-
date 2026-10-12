***Bài 7 : Debug Shell Script xử lý log CRITICAL*** 

**Phần 1: Phân tích lỗi (Trace code)**

Dựa vào cấu trúc trích đoạn của file server.log và logic vận hành của deploy\_checker.ps1, nguyên nhân khiến file báo cáo daily\_critical.txt bị trống phần nội dung lỗi là do sự bất đồng bộ chữ hoa/thường kết hợp với cờ nghiêm ngặt \-CaseSensitive:

1. Dữ liệu log thực tế: Hệ thống ghi nhận chuỗi sự kiện lỗi ở định dạng viết hoa toàn bộ: CRITICAL.  
2. Cú pháp lọc của script: Câu lệnh đang cấu hình mẫu tìm kiếm là chữ thường "critical" đồng thời ép buộc hệ thống bật chế độ kiểm tra chính xác từng ký tự hoa/thường bằng cờ \-CaseSensitive.  
3. Hậu quả: Lệnh Select-String tiến hành so khớp và nhận thấy từ khóa "critical" (chữ thường) không trùng khớp với nhãn CRITICAL (chữ hoa) trong log. Do đó, luồng xuất chuẩn trả về rỗng, khiến file daily\_critical.txt hoàn toàn trống nội dung lỗi ở giữa, chỉ ghi nhận 2 dòng tiêu đề và dòng cuối đếm tổng số dòng hiện có của file báo cáo (là 4 dòng văn bản tiêu đề).

**Phần 2: Sửa lỗi và kiểm thử**

1\. Đoạn script deploy\_checker.ps1 đã sửa đúng

Cách sửa tối ưu nhất là loại bỏ hoàn toàn cờ \-CaseSensitive để đưa Select-String về trạng thái mặc định (không phân biệt chữ hoa hay chữ thường), giúp script chạy linh hoạt và an toàn hơn:

powershell  
\# Script: deploy\_checker.ps1  
\# Mục đích: Trích xuất lỗi CRITICAL từ server.log vào daily\_critical.txt

"=== Bat dau quet log: \$(Get-Date) \===" | Out-File daily\_critical.txt  
Select-String \-Path server.log \-Pattern "CRITICAL" | Out-File \-Append daily\_critical.txt  
"=== Hoan tat. So dong loi: \===" | Out-File \-Append daily\_critical.txt  
((Get-Content daily\_critical.txt) | Select-String \-Pattern "CRITICAL").Count | Out-File \-Append daily\_critical.txt

2\. Kết quả mong đợi của file daily\_critical.txt sau khi chạy đúng

*(Giả sử thời gian chạy script là lúc 05:00:00 sáng ngày 18/09/2026)*

text  
\=== Bat dau quet log: 09/18/2026 05:00:00 \===  
server.log:2:2026-09-18 02:30:11 CRITICAL Database connection pool exhausted  
server.log:3:2026-09-18 02:30:45 CRITICAL Retry limit exceeded for payment-service  
\=== Hoan tat. So dong loi: \===

3\. Hiện tượng xảy ra khi file server.log không tồn tại

Nếu tệp tin server.log bị thiếu hoặc sai đường dẫn, script sẽ không im lặng mà sẽ lập tức bắn ra các thông báo lỗi màu đỏ (Error/Exception) hiển thị trực tiếp trên Terminal.

Lệnh Select-String sẽ báo lỗi đỏ với nội dung dạng: *Cannot find path... because it does not exist*.

Tuy nhiên, do script chạy tuần tự không có bẫy lỗi try-catch, hệ thống vẫn sẽ thực thi các dòng lệnh Out-File còn lại. Hậu quả là file daily\_critical.txt vẫn được tạo ra nhưng nội dung sẽ bị lỗi logic (chỉ chứa dòng tiêu đề bắt đầu, dòng tiêu đề hoàn tất, và số dòng đếm bằng 0 hoặc báo lỗi).

