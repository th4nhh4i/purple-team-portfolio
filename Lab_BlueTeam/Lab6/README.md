
> 🎬 **Video hướng dẫn / diễn tập (Lab6):** [https://drive.google.com/file/d/1B2TeohTb3k2rBjpq3Su1b11NbI6hA0q-/view?usp=sharing](https://drive.google.com/file/d/1B2TeohTb3k2rBjpq3Su1b11NbI6hA0q-/view?usp=sharing)

Kịch bản 6: NETWORK SCANNING & RECONNAISSANCE (DÒ QUÉT MẠNG & THU THẬP THÔNG TIN)

1. Mô tả tình huống

Kẻ tấn công thực hiện gửi hàng loạt các gói tin thăm dò hệ thống (như ICMP Echo Request, TCP SYN) tới một dải địa chỉ IP hoặc một máy chủ mục tiêu. Mục đích nhằm xác định các máy chủ đang hoạt động (Host Discovery), các cổng dịch vụ đang mở (Port Scanning) và nhận diện phiên bản hệ điều hành đang sử dụng. Đây là bước chuẩn bị (Reconnaissance) cực kỳ quan trọng giúp hacker tìm ra các dịch vụ lỗi thời, chưa vá lỗi nhằm xác định "mắt xích yếu nhất" để bắt đầu chiến dịch tấn công chính thức.  

2. Mô hình triển khai lab

2. Chuẩn bị môi trường 

Máy tấn công (Attacker) 

Kali Linux    

Công cụ Nmap hoặc tcpdump  

Bật Security Audit cho Windows Filtering Platform

auditpol /set /subcategory:"Filtering Platform Packet Drop" /failure:enable

auditpol /set /subcategory:"Filtering Platform Connection" `

    /success:enable /failure:enable

Lệnh kiểm tra

auditpol /get /subcategory:"Filtering Platform Packet Drop"

auditpol /get /subcategory:"Filtering Platform Connection"

Kết quả:

Filtering Platform Packet Drop    Failure

Filtering Platform Connection     Success and Failure

Vào đường dẫn trên windows server 2019 C:\Program Files (x86)\ossec-agent\ossec.conf

# thêm 2 thẻ này vào

  <localfile>

    <location>Security</location>

    <log_format>eventchannel</log_format>

  </localfile>

  <localfile>

    <location>C:\Windows\System32\LogFiles\Firewall\pfirewall.log</location>

    <log_format>syslog</log_format>

  </localfile>

Khởi động lại agent: Restart-Service WazuhSvc

Kiểm tra log xem các cấu hình đã hoạt động chưa

Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 40

Chúng ta cần thấy dòng:

Connected to the server

Analyzing file: 'C:\Windows\System32\LogFiles\Firewall\pfirewall.log'

Máy nạn nhân (Victim) 

Windows Server 

Kích hoạt và mở đồng thời nhiều dịch vụ mạng cơ bản để làm mục tiêu: SSH (Port 22), Web Server (Port 80/443), SMB (Port 445)    

Cấu hình tường lửa nội bộ (Windows Firewall) ở trạng thái mặc định (mở phản hồi) để hacker thu thập được banner dịch vụ    

Cấu hình đẩy nhật ký hệ thống và log mạng về SIEM  

3. Các bước thực hiện tấn công

Mở Terminal trên Kali Linux, chạy lệnh quét nhanh các cổng phổ biến trong dải mạng:  

192.168.98.193: chính là máy Windows Server: Máy này đang mở hàng loạt cổng đặc trưng của Windows như 80/tcp open http (Dịch vụ IIS mà bạn vừa bật thành công) , 445/tcp open microsoft-ds (Dịch vụ SMB), cùng các cổng quản lý hệ thống khác như 135, 139, 389 (LDAP).   

192.168.98.1: Thường là IP của máy Host hoặc Gateway ảo của VMware/VirtualBox (đang mở cổng 445 và 139). 

192.168.98.131: là máy Linux khác trong Lab đang mở cổng 22/tcp open ssh và 443/tcp open https. 

Chạy tiếp lệnh quét chuyên sâu để nhận diện dịch vụ và hệ điều hành của Windows Server: 

Hệ điều hành chính xác: Windows Server 2016 Standard Evaluation 14393.   

Vai trò của Server: Máy này không chỉ chạy Web mà còn là Domain Controller (DC) quản lý toàn bộ hạ tầng mạng, vì nó đang mở các cổng Active Directory như 88 (Kerberos), 389/3268 (LDAP) và tên Host là DC01. 

Thông tin định danh nội bộ (Information Leakage): 

Domain Name: bt.lab    

Forest Name: bt.labFQDN: 

DC01.bt.lab 

Các dịch vụ làm "bia ngắm": 

Port 80: Chạy Microsoft IIS httpd 10.0.   

Port 445: Chạy microsoft-ds (SMB). 

Sử dụng các script tự động để kiểm tra lỗ hổng bảo mật (CVE): 

nmap --script vuln 192.168.98.193 

Lỗ hổng bảo mật nghiêm trọng (Vulnerability Assessment) 

Lỗ hổng phát hiện: Hệ thống dính lỗi thực thi mã từ xa cực kỳ nguy hiểm MS17-010 (EternalBlue / CVE-2017-0143). 

Nguyên nhân: Do máy mục tiêu vẫn đang kích hoạt giao thức mạng lỗi thời SMBv1 chưa được vá lỗi. Hacker có thể lợi dụng lỗ hổng này để chiếm quyền kiểm soát toàn bộ máy chủ 

Phản ứng an ninh của hệ thống (Defense Detection) 

Cơ chế bảo vệ mạng: Quá trình quét ghi nhận thông báo lỗi đọc dữ liệu Connection reset by peer. Điều này cho thấy hệ thống đích đã kích hoạt tính năng tự vệ (như SynAttackProtect) để chủ động ngắt kết nối khi phát hiện lưu lượng quét dồn dập.   

Trạng thái Tường lửa: Xuất hiện nhiều cổng ở trạng thái Filtered, chứng tỏ có sự can thiệp chặn gói tin từ Windows Firewall. 

4. Kiểm tra trên SIEM 

Mở

Discover

Lọc các Filter quan sát 

Mở

6. Hướng dẫn viết rule

<group name="local,custom_network_scan,">

<rule id="100700" level="15">

<match>ALLOW TCP</match>

<description>

[CUSTOM] Network Scan Detected

</description>

<group>

network_scan,

reconnaissance,

custom,

</group>

<mitre>

<id>T1046</id>

</mitre>

</rule>

</group> 

7. Tấn công sau khi xây dựng Rule 

Chạy lại

nmap -F 192.168.98.193

hoặc

nmap -sV -sC 192.168.98.193

8. Xử lý sự cố

Tự động (Automated): 

Cấu hình tính năng SynAttackProtect và MaxConnectBacklog trực tiếp trong Registry của Windows Server nhằm kích hoạt cơ chế tự động phòng vệ khi phát hiện số lượng kết nối nửa mở (Half-open connections) vượt ngưỡng an toàn (mô phỏng phản ứng tự động của hệ thống). 

Nhấn Windows + R, gõ regedit và nhấn Enter để mở Registry Editor. 

Tìm kiếm theo đường dẫn chính xác sau: 

HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters 

Tạo mới và thiết lập thông số cho 2 giá trị sau: 

Giá trị 1: Click chuột phải vào vùng trống chọn New -> DWORD (32-bit) Value. Đặt tên là SynAttackProtect. Click đúp vào nó và sửa giá trị (Value data) thành 2 (Hệ số cơ số Hex hoặc Decimal đều được). 

Giải thích cơ chế: 

0: Tắt bảo vệ. 

1: Bật bảo vệ nâng cao khi hệ thống nhận diện có dấu hiệu bị quét/tấn công SYN. 

2: Mức bảo vệ tối đa. Khi số kết nối nửa mở vượt ngưỡng, Windows sẽ giảm thời gian chờ phản hồi (Timeout), hủy bớt các kết nối cũ chưa hoàn tất và trì hoãn việc cấp phát tài nguyên cho đến khi bắt tay 3 bước (Three-way handshake) hoàn thành hoàn toàn. 

Giá trị 2: Tiếp tục tạo một DWORD (32-bit) Value mới. Đặt tên là MaxConnectBacklog. Click đúp và sửa giá trị thành 1024 (Chọn cơ số Decimal - Thập phân). 

Hành động cường hóa ngăn xếp TCP/IP (TCP/IP Stack Hardening) bằng cách thiết lập cấu hình SynAttackProtect = 2 và MaxConnectBacklog = 1024 mang ý nghĩa quyết định trong việc thiết lập hàng rào tự vệ chủ động cho máy chủ Domain Controller. Giải pháp này giải quyết triệt để bài toán: Vừa bảo vệ tài nguyên hệ thống không bị cạn kiệt trước các đợt quét tốc độ cao , vừa làm phá sản chiến thuật thu thập thông tin tình báo (Banner Grabbing) của hacker , đồng thời duy trì tính sẵn sàng và vận hành liên tục cho các dịch vụ mạng nội bộ. 

Thủ công (Manual): 

Thực hành lệnh chặn đứng IP của máy Kali (Attacker) bằng Firewall có sẵn trên OS của Victim:  

Trên Windows: New-NetFirewallRule -DisplayName "Block Attacker" -Direction Inbound -Action Block -RemoteAddress 192.168.98.131 

Sau khi xác định chính xác địa chỉ IP nguồn phát động cuộc quét thông qua hệ thống giám sát tập trung, biện pháp ngăn chặn khẩn cấp đã được thực thi bằng cách sử dụng công cụ Windows PowerShell quyền quản trị cao nhất.  

Lệnh triển khai: New-NetFirewallRule -DisplayName "Block Attacker" -Direction Inbound -Action Block -RemoteAddress 192.168.98.131.  

Kết quả vận hành: Hệ thống đã thiết lập một luật chặn đứng (Drop/Block) có hiệu lực ngay lập tức đối với mọi gói tin đi vào từ IP nguồn độc hại. Rule tường lửa chuyển trạng thái Enabled: True, đảm bảo cắt đuôi hoàn toàn chiến dịch rà quét dịch vụ AD (389), SMB (445) và chặn đứng nguy cơ hệ thống bị khai thác lỗ hổng nghiêm trọng ở các bước tiếp theo. 

Phòng ngừa (Proactive):  

Thực hành "Tàng hình hóa" dịch vụ mạng bằng cách cấu hình lại Firewall mặc định ở chế độ DROP thay vì REJECT khi có gói tin lạ đến, khiến công cụ quét tự động của hacker bị mất thời gian chờ (Timeout). 

Dùng lệnh PowerShell  

Disable-NetFirewallRule -Name "ADDS-ICMP4-In"  

Get-NetFirewallRule | Where-Object {$_.DisplayName -like "Echo Request"} | Select-Object Name, DisplayName, Enabled 

Bản chất vận hành: Chuyển đổi trạng thái xử lý gói tin lạ của Windows Firewall sang cơ chế DROP. Hệ thống sẽ giữ sự im lặng tuyệt đối, lặng lẽ hủy bỏ các gói tin thăm dò mạng mà không phản hồi bất kỳ gói tin ICMP Echo Reply nào ra bên ngoài.  

Hiệu quả phòng thủ thực tế: Đảm bảo "tàng hình hóa" hoàn toàn thiết bị Domain Controller cốt lõi đối với các công cụ quét tự động của hacker. Mọi nỗ lực gửi gói tin kiểm tra trạng thái từ mạng bên ngoài sẽ rơi vào trạng thái mất thời gian chờ (Timeout), làm mất thời gian và làm sai lệch sơ đồ trinh sát của kẻ tấn công. 

Kết quả Scan từ Attacker 