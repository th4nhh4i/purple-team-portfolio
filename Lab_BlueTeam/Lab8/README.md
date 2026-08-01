
> 🎬 **Video hướng dẫn / diễn tập (Lab8):** [https://drive.google.com/file/d/1dskTD_S04Fynf6X6T_Jhvp1ZvAIoX7eF/view?usp=sharing](https://drive.google.com/file/d/1dskTD_S04Fynf6X6T_Jhvp1ZvAIoX7eF/view?usp=sharing)

TẤN CÔNG PHÒNG THỦ

KỊCH BẢN DEMO: RANSOMWARE QUA PHISHING EMAIL

Hình 1. Tổng quan ransomware attack lifecycle: từ phishing đến disaster recovery.

KỊCH BẢN DEMO: RANSOMWARE QUA PHISHING EMAIL

Macro → PowerShell C2 → Credential Dump → Lateral Movement → Encrypt → DR Drill

1. MÔ TẢ TÌNH HUỐNG

Kẻ tấn công sử dụng email phishing giả mạo nội dung trao đổi công việc hoặc chứng từ thanh toán nhằm lừa người dùng mở tài liệu độc hại. Sau khi người dùng tương tác với file, chuỗi thực thi bất thường được kích hoạt và tạo điều kiện cho attacker thực thi PowerShell trên máy nạn nhân, thiết lập kết nối Command and Control (C2) và duy trì khả năng điều khiển hệ thống từ xa.

Sau khi có được foothold ban đầu, attacker tiến hành thu thập thông tin hệ thống, tìm kiếm credential có giá trị và khai thác các phiên đăng nhập hoặc tài khoản có đặc quyền để mở rộng phạm vi kiểm soát. Các credential bị lộ có thể được sử dụng để thực hiện lateral movement sang file server, máy chủ quan trọng hoặc các tài nguyên khác trong mạng nội bộ.

Khi đã đạt được quyền truy cập đủ rộng và xác định các tài sản có giá trị, attacker chuyển sang giai đoạn Impact bằng cách thực hiện mã hóa hàng loạt dữ liệu, làm gián đoạn hoạt động của người dùng và các dịch vụ doanh nghiệp. Sau khi sự cố được phát hiện, đội Incident Response tiến hành cô lập hệ thống bị ảnh hưởng, xác định phạm vi xâm nhập, xử lý các tài khoản bị compromise và thực hiện Disaster Recovery Drill nhằm đánh giá khả năng khôi phục dữ liệu, dịch vụ và trạng thái vận hành an toàn của hệ thống.

Đây là một kịch bản điển hình của ransomware attack lifecycle, trong đó hành vi mã hóa dữ liệu chỉ là giai đoạn cuối của một chuỗi tấn công gồm Initial Access, Execution, Command and Control, Credential Access, Discovery, Lateral Movement và Impact. Kịch bản này thường được sử dụng trong các bài thực hành Purple Team nhằm đánh giá khả năng phát hiện, điều tra và ứng phó của Blue Team tại từng giai đoạn của cuộc tấn công.

Bối cảnh: Công ty ABC — doanh nghiệp vừa, ~200 nhân viên. Hạ tầng gồm 1 Domain Controller, 2 File Server, ~50 workstation Windows 10. Phòng Kế Toán có 5 nhân viên, thường xuyên xử lý chứng từ thanh toán qua email.

Kịch bản: Kẻ tấn công gửi email phishing giả mạo Phòng Kế Toán, yêu cầu nạn nhân tải về và ký duyệt một chứng từ thanh toán gấp. Từ một cú click, toàn bộ hệ thống bị xâm nhập, credentials bị đánh cắp, dữ liệu bị mã hóa.

2. SƠ ĐỒ MẠNG LAB

Hình 2 : Hacker muốn tấn công vào một máy trong mạng

3. QUY TRÌNH TẤN CÔNG

CHUẨN BỊ CỦA ATTACKER

Trước khi tấn công, attacker đã chuẩn bị sẵn:

Kỹ thuật né tránh được áp dụng: - File .bat đặt tên tai_lieu.docx.bat — Windows mặc định ẩn đuôi .bat, nạn nhân chỉ thấy tai_lieu.docx - File .docm được tải qua WebClient.DownloadFile (raw .NET socket) — không bị Windows gắn Mark-of-the-Web, do đó Word không chặn macro - Macro chạy PowerShell với cửa sổ ẩn (WindowStyle=Hidden), nạn nhân không thấy gì bất thường - Toàn bộ script chạy tự động 5 phases, không cần C2 dispatch task — giảm traffic đáng ngờ

Giờ chúng ta sẽ phải chuẩn bị 1 file docx chứa 1 file thực thi ẩn bên trong

Giờ chúng ta cần có 1 server c2 để file ẩn đó liên lạc chúng ta có thể sử dụng repo này đây là một c2 mẫu : https://github.com/th4nhh4i/ServerC-Hai

Sau đó tải file macro_vba.bas này xuống

Chuẩn bị 1 file word có nội dung đồng nhất với mail sau đó sử dụng phím tắt Alt + F11 để mở cửa sổ soạn thảo Visual Basic for Applications (VBA) 

Sau đó chọn import và chọn file macro_vba.bas , để đưa mã ẩn vào word

Sau khi import thì ta thấy nó sẽ có thêm phần AutoOpen

GIAI ĐOẠN 1: PHISHING & INITIAL ACCESS (XÂM NHẬP BAN ĐẦU) 

8:30 - Kẻ tấn công: Gửi một email phishing giả mạo từ hòm thư ketoan@congty-noiba.com.vn tới anh Nguyễn Văn An. Email có tiêu đề hối thúc: "[CAN GAP] Don yeu cau thanh toan - Ma so #2024-8872" với nội dung yêu cầu ký duyệt gấp chứng từ thanh toán trị giá 250.000.000 VNĐ trước 17:00 cùng ngày.

Kính gửi Anh/Chị,

Phòng Kế toán đã nhận được Đơn yêu cầu thanh toán từ bộ phận Mua hàng (Mã số: #2024-8872) với giá trị 250.000.000 VND.

Để đảm bảo tiến độ thanh toán đúng hạn, vui lòng truy cập liên kết dưới đây để tải xuống tài liệu:

[Tải xuống Đơn yêu cầu thanh toán]

Sau khi tải xuống, xin vui lòng kiểm tra lại toàn bộ thông tin và ký xác nhận.

  - Hạn cuối thực hiện: 17:00 hôm nay (30/07/2026)

Trân trọng,

Phòng Kế toán

Công ty ABC

Email: ketoan@congty-noiba.com.vn

Lưu ý: Email này được gửi từ hệ thống nội bộ. Nếu có thắc mắc, vui lòng liên hệ bộ phận IT để được hỗ trợ.

08:35 - Nạn nhân: Anh An mở email, thấy nội dung rất gấp và có giao diện chuyên nghiệp nên đã bấm vào liên kết "Tải xuống Đơn yêu cầu thanh toán". 

08:36 - Nạn nhân & Kẻ tấn công:

Anh An click tải tài liệu “Tải xuống Đơn yêu cầu thanh toán”, một file có tên tai_lieu.docx được tải về (thực chất Windows đã ẩn đuôi file thật là tai_lieu.docx.bat). 

Anh An double-click để mở file này.

File word được mở và file ẩn bên trong được thực thi

GIAI ĐOẠN 2: EXECUTION & COMMAND AND CONTROL (THỰC THI & THIẾT LẬP KẾT NỐI)

8:40 - Kẻ tấn công:

File .bat kích hoạt ngầm một lệnh PowerShell để tải file tài liệu Word chứa mã độc Macro Don_Yeu_Cau_Thanh_Toan.docm về máy nạn nhân. 

Do tải trực tiếp bằng .NET socket (WebClient.DownloadFile), file này né được cơ chế Mark-of-the-Web của Windows nên bộ Office không chặn cảnh báo bảo mật nguy hiểm. Và file mã độc được tải xuống là standalone_ransome.ps1

08:42 - Nạn nhân & Kẻ tấn công:

File Word mở ra một "Giấy đề nghị thanh toán" hợp lệ để đánh lạc hướng anh An. Tuy nhiên, tiến trình Macro chạy ngầm bên dưới đã âm thầm khởi chạy PowerShell ở chế độ ẩn (WindowStyle=Hidden). 

PowerShell ẩn này lập tức thực thi tải payload tấn công standalone_ransom.ps1 và kết nối ngược về máy chủ C2 tại IP 192.168.121.253:8443. 

08:45 - Kẻ tấn công: Trên Dashboard C2 hiển thị thiết bị DESKTOP-2Q24A1O (User123) đã "Check-in" thành công. Kẻ tấn công nhận được toàn bộ thông tin trinh sát hệ thống gửi về bao gồm: Tên máy, hệ điều hành (Windows 10 Home), dung lượng RAM (2GB), danh sách IP, danh sách tài khoản local (User123 đang active) và phần mềm diệt virus đang chạy (Windows Defender). 

Các thông tin mà kẻ tấn công thu thập được: - Hostname, username, domain - Phiên bản Windows, dung lượng RAM - Danh sách địa chỉ IP, subnet - Danh sách user local, ai thuộc nhóm Administrators - Phần mềm diệt virus đang chạy (nếu có)

Sau khi agent trên máy DESKTOP-2Q24A1O kết nối thành công về C2 Server, dashboard ghi nhận phiên hoạt động của tài khoản User123 tại địa chỉ IP 192.168.121.187. Tại thời điểm quan sát, agent đang ở phase encrypt và vẫn duy trì trạng thái beacon/check-in với C2.

Trong quá trình System Reconnaissance, agent đã thu thập các thông tin chính gồm:

Hệ điều hành: Microsoft Windows 10 Home 

Domain/Workgroup: WORKGROUP 

Bộ nhớ: 2 GB 

Địa chỉ IP: 192.168.121.187 

Phần mềm bảo mật: Windows Defender 

C2 đồng thời liệt kê các tài khoản local trên hệ thống gồm Administrator, DefaultAccount, Guest, User123 và WDAGUtilityAccount. Trong đó, User123 là tài khoản đang được kích hoạt.

GIAI ĐOẠN 3: LATERAL MOVEMENT & IMPACT (MÃ HÓA DỮ LIỆU)

08:50 - Kẻ tấn công: Từ foothold (quyền truy cập ban đầu) chiếm được trên máy anh An, hacker thực hiện trích xuất thông tin đăng nhập (Credential Access), rà quét mạng nội bộ (Discovery) và tìm cách di chuyển sang máy chủ chia sẻ dữ liệu FILE-SRV01 (Lateral Movement) nhưng không thành công. 

09:15 - Kẻ tấn công: Kích hoạt payload ransomware mã hóa hàng loạt trên máy anh An và các phân vùng mạng chia sẻ mà tài khoản này có quyền truy cập. 

09:17 - Nạn nhân:

Toàn bộ tài liệu công việc trên máy anh An bị đổi đuôi thành .ENCRYPTED và không thể mở được. 

Một file văn bản tống tiền tên !!!README_DECRYPT!!! tự động xuất hiện trên màn hình desktop, yêu cầu thanh toán khoản tiền chuộc 1.0 BTC tới địa chỉ ví được chỉ định trong vòng 72 giờ.

Anh An hoảng hốt nhận ra máy tính bị hack và gọi điện báo khẩn cấp cho phòng IT. 

4. ỨNG PHÓ VÀ KHÔI PHỤC SỰ CỐ

Sau khi ransomware gây ảnh hưởng đến hệ thống, Blue Team chuyển từ giai đoạn phát hiện sang Incident Response và Disaster Recovery. Mục tiêu là cô lập sự cố, xác định phạm vi xâm nhập, loại bỏ quyền kiểm soát của attacker và khôi phục hệ thống từ trạng thái an toàn.

4.1. Phát hiện và tuyên bố sự cố

Khi phát hiện dấu hiệu mã hóa hàng loạt file, ransom note hoặc kết nối bất thường tới C2, Blue Team cần xác nhận sự cố và kích hoạt quy trình Incident Response.

Các thông tin ban đầu cần xác định:

Máy bị ảnh hưởng đầu tiên. 

Tài khoản đang được sử dụng. 

Các server đã bị truy cập. 

Phạm vi dữ liệu bị mã hóa. 

Trạng thái của Domain Controller và Backup Server. 

4.2. Cô lập hệ thống bị ảnh hưởng

Các máy bị compromise cần được cô lập khỏi hệ thống mạng nhằm hạn chế ransomware tiếp tục lây lan.

Blue Team thực hiện:

Cô lập endpoint bị nhiễm. 

Chặn IP hoặc domain của C2. 

Vô hiệu hóa tài khoản nghi bị compromise. 

Hạn chế các kết nối SMB, WinRM hoặc WMI bất thường. 

Bảo quản log và bằng chứng phục vụ điều tra. 

4.3. Xác định phạm vi xâm nhập

Blue Team tiến hành dựng lại attack timeline để xác định blast radius của sự cố.

Mỗi hệ thống cần được phân loại:

Compromised

Encrypted

Accessed

Clean

Unknown

Mục tiêu là xác định hệ thống nào đã bị attacker kiểm soát và hệ thống nào có thể sử dụng cho quá trình recovery

4.4. Disaster Recovery Drill

Disaster Recovery Drill là bài diễn tập nhằm kiểm tra khả năng khôi phục hệ thống sau sự cố ransomware.

Ví dụ tình huống:

Domain Controller compromised

File Server encrypted

Virtual machines unavailable

4.5. Thứ tự ưu tiên khôi phục

Việc khôi phục cần được thực hiện theo mức độ quan trọng của hệ thống.

Identity Infrastructure: Khôi phục Domain Controller, Active Directory và các dịch vụ authentication.

Core Network Services: Khôi phục DNS, DHCP và các dịch vụ mạng cơ bản.

Critical Applications: Khôi phục các ứng dụng nghiệp vụ quan trọng.

Databases: Restore database và kiểm tra tính toàn vẹn dữ liệu.

File Services: Khôi phục File Server từ backup sạch.

User Endpoints: Rebuild workstation, cập nhật bản vá và triển khai lại EDR.

4.6. Kiểm tra trước khi đưa hệ thống trở lại hoạt động

Trước khi đưa hệ thống trở lại production, Blue Team cần xác nhận:

No active C2 connection

No suspicious persistence

Compromised accounts reset

EDR operational

Logging enabled

Backup verified

Critical services functional

Quy trình cuối cùng: