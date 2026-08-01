
> 🎬 **Video hướng dẫn / diễn tập (Lab3):** [https://drive.google.com/file/d/1FNEb-hi_TQtOAH-lWFpZNp_oRG_dlpiA/view?usp=drive_link](https://drive.google.com/file/d/1FNEb-hi_TQtOAH-lWFpZNp_oRG_dlpiA/view?usp=drive_link)

Kịch bản 3:  INSIDER THREAT ĐÁNH CẮP DỮ LIỆU NỘI BỘ

1. Mô tả tình huống

Mục tiêu: Phát hiện hành vi người dùng nội bộ lạm dụng đặc quyền để thu thập dữ liệu nhạy cảm diện rộng, sử dụng công cụ hệ thống đóng gói và tải lên các dịch vụ lưu trữ đám mây cá nhân ngoài giờ làm việc

2. Chuẩn bị môi trường

3. Các bước thực hiện tấn công (Red Team)

4. Kiểm tra trên SIEM

Sau khi hoàn thành các bước tấn công, truy xuất log thông qua /var/ossec/logs/archives/archives.json và đồng thời truy cập Wazuh Dashboard, sau đó lựa chọn khoảng thời gian tương ứng. Trong Wazuh, tìm kiếm sự kiện của Window thông qua phần Threat Hunting:

Có thể thấy rằng trước thời điểm custom rule được triển khai (10:25), hệ thống không tạo ra bất kỳ cảnh báo nào liên quan đến hành vi exfiltration. Mặc dù Wazuh vẫn thu thập thành công các log từ Sysmon, bao gồm thông tin về tiến trình curl.exe, đường dẫn thực thi, tham số dòng lệnh và địa chỉ IP đích, nhưng các sự kiện này chỉ được lưu trữ dưới dạng log thông thường. Để nhận diện hành vi tải tệp ZIP lên máy chủ bên ngoài, nhà phân tích phải tự kiểm tra và phân tích nội dung của trường CommandLine. Điều này cho thấy hệ thống có khả năng quan sát và thu thập dữ liệu (Visibility) nhưng chưa có cơ chế phát hiện và cảnh báo tự động (Detection) đối với hành vi đánh cắp dữ liệu thông qua curl.exe. Do đó, nguy cơ bỏ sót các hoạt động exfiltration là tương đối cao nếu việc giám sát chỉ dựa vào quá trình phân tích thủ công.   

5. Xây dựng rule trên SIEM

6. Xây dựng rule trên Window Agent

 7. Kịch bản phát hiện và điều tra (Blue Team)

Suspicious Outbound Transfer: Ghi nhận tiến trình hệ thống curl.exe kết nối mạng ra địa chỉ IP Public lạ qua cổng 8000 ngoài giờ hành chính.

Kết quả cho thấy Wazuh ngay lập tức sinh ra một cảnh báo mới với mức độ nghiêm trọng cao. Alert hiển thị rõ ràng hệ thống đã phát hiện hành vi tải tệp bằng curl.exe, đồng thời gắn nhãn MITRE ATT&CK liên quan đến kỹ thuật Exfiltration. Nhà phân tích không còn phải tự đọc từng dòng CommandLine để xác định mối đe dọa như trước. 

Trước khi triển khai Custom Rule, Wazuh chỉ ghi nhận sự kiện Sysmon Process Create nhưng không hiểu rằng đây là hành vi exfiltration. Toàn bộ quá trình phân tích phụ thuộc vào khả năng quan sát và kinh nghiệm của SOC Analyst. Nếu số lượng log lớn, khả năng bỏ sót sự kiện là rất cao.

Sau khi triển khai Custom Rule, cùng một hành vi tấn công được tự động nhận diện và sinh cảnh báo ngay khi xảy ra. Hệ thống không chỉ thu thập dữ liệu mà còn có khả năng phát hiện chủ động. Điều này giúp giảm đáng kể thời gian điều tra, tăng khả năng phát hiện sớm và hỗ trợ SOC Analyst tập trung vào các sự kiện thực sự quan trọng thay vì phải rà soát thủ công hàng nghìn log mỗi ngày.

Như vậy, Custom Rule đã chuyển hệ thống từ trạng thái "có log nhưng không phát hiện" sang trạng thái "có log và tự động cảnh báo", qua đó nâng cao đáng kể hiệu quả giám sát và phát hiện các hành vi đánh cắp dữ liệu trong môi trường doanh nghiệp.

8. Quy trình xử lý sự cố (Incident Response Playbook)

Ngăn chặn: Thực hiện khóa (Disable) ngay lập tức tài khoản Active Directory của nhân viên kinh doanh trên Domain Controller. Sử dụng giải pháp EDR cách ly máy trạm khỏi mạng nội bộ.