
> 🎬 **Video hướng dẫn / diễn tập (Lab1):** [https://drive.google.com/drive/folders/1ZP1N_SjpqM1MotUzIZ-xUL3k2El-2Nd9](https://drive.google.com/drive/folders/1ZP1N_SjpqM1MotUzIZ-xUL3k2El-2Nd9)

DIỄN TẬP LAB
Kịch bản 1: PRIVILEGE ESCALATION TRÊN LINUX

1. Mô tả tình huống

Kẻ tấn công sau khi có được quyền truy cập ban đầu vào hệ thống Linux dưới tài khoản người dùng thông thường (user thường) sẽ tìm cách khai thác các cấu hình sai lệch liên quan đến sudo hoặc các binary có gán quyền SUID nhằm leo thang đặc quyền lên quyền root. Sau khi chiếm được quyền root, attacker có thể thực hiện toàn quyền trên hệ thống như đọc dữ liệu nhạy cảm, tạo tài khoản backdoor, cài persistence hoặc xóa log phục vụ che giấu dấu vết.

Đây là một trong những kỹ thuật phổ biến trong giai đoạn hậu khai thác (Post-Exploitation) và thường được sử dụng sau khi attacker đã thực hiện thành công Initial Access hoặc Remote Code Execution.

2. Chuẩn bị môi trường Lab

Máy tấn công (Attacker)

Kali Linux

CPU:2 core

RAM: 2GB

Disk: 50GB

Máy nạn nhân (Victim)

Ubuntu Server 24.04

CPU:2 core

RAM: 2GB

Disk: 50GB

Hệ thống giám sát

Wazuh

CPU: 4 core

RAM: 6GB

Disk: 100GB

3. Mô hình triển khai Lab

4. Cấu hình môi trường ban đầu

Bước 1: Tạo user thường trên Ubuntu

Bước 2: Kích hoạt SSH Service

Bước 3: Cài đặt Wazuh Agent

Bước 4.  Thiết lập lỗ hổng Privilege Escalation

Cấu hình sudo sai cho tài khoản labuser. Tài khoản này được phép chạy /usr/bin/find dưới quyền root mà không cần nhập mật khẩu:

Mở file:

Thêm dòng sau:

Đây là cấu hình sai phổ biến thường bị attacker khai thác trong thực tế.

5. Hướng dẫn tấn công (Attacker Command)

Bước 1: SSH vào máy nạn nhân

Sau khi đăng nhập, kiểm tra tài khoản hiện tại: whoami

Bước 2: Kiểm tra quyền sudo

Kết quả trả về cho thấy user được phép chạy binary find dưới quyền root mà không cần mật khẩu

Bước 3: Khai thác privilege escalation

Bước 4: Kiểm tra quyền root

Kết quả mong muốn: root

Bước 5: Sinh log phục vụ điều tra

Sau khi leo quyền thành công, attacker thực hiện một số command nhằm sinh log:

6. Kiểm tra trên SIEM

Kiểm tra trên Wazuh để xác định hệ thống đã ghi nhận được những hoạt động nào xảy ra trên máy Ubuntu Web Server. 

Wazuh ghi nhận hoạt động thực thi lệnh sudo thành công với quyền root thông qua Rule 5402 

Các sự kiện mở và đóng phiên đặc quyền được phát hiện bởi Rule 5501 và Rule 5502 

Sau khi giành được quyền root, tài khoản thực hiện kiểm thử tiếp tục tạo tài khoản người dùng và nhóm mới trên hệ thống. Các hành vi này lần lượt được Wazuh ghi nhận thông qua Rule 5902 và Rule 5901 

Tài khoản labuser lợi dụng quyền sudo đối với binary find để thực thi /bin/bash dưới quyền root. Sau khi lệnh được thực hiện, hệ thống ghi nhận phiên PAM của tài khoản root được mở, chứng minh quá trình leo thang đặc quyền thành công.

7. Xây dựng Rule trên SIEM

Đăng nhập Wazuh Dashboard.

Đi theo đường dẫn:

<group name="local,linux,privilege_escalation,sudo_abuse,">

  <rule id="110400" level="12">

    <if_sid>5402, 5403</if_sid>

    <field name="command" type="pcre2">^/usr/bin/find\s+\.\s+-exec\s+/bin/bash\s*;\s*-quit$</field>

    <description>Detect sudo find spawning a root shell.</description>

    <mitre>

      <id>T1548.003</id>

    </mitre>

    <group>privilege_escalation,sudo_abuse,suspicious_shell,</group>

  </rule>

</group>

IOC phổ biến

8. Kiểm tra trên SIEM 

 Sau khi triển khai custom Rule 110400, Wazuh phát hiện hành vi sử dụng binary find để mở shell dưới quyền root và sinh cảnh báo ở mức 12. 

Ngay sau cảnh báo này, hệ thống ghi nhận phiên PAM đặc quyền được mở thông qua Rule 5501. 

Các hành vi tạo tài khoản và nhóm mới được phát hiện bởi Rule 5902 và Rule 5901, cho thấy người thực hiện kiểm thử đã tiến hành các hoạt động hậu khai thác sau khi leo thang đặc quyền thành công. 

Trường full_log cho thấy tài khoản labuser thực thi lệnh /usr/bin/find . -exec /bin/bash ; -quit với tài khoản đích là root. Custom Rule 110400 đã đối chiếu nội dung câu lệnh và phân loại sự kiện thành hành vi lợi dụng sudo find để mở shell quyền root.

Cảnh báo được gán mức độ 12 và các nhóm privilege_escalation, sudo_abuse và suspicious_shell. Kết quả này chứng minh custom rule đã bổ sung khả năng nhận diện chuyên biệt, giúp phân biệt hành vi khai thác với các hoạt động sử dụng sudo thông thường.

9. Hướng dẫn điều tra (Investigation)

Trên SIEM

Thực hiện truy vấn:

event.action: sudo

Hoặc:

process.name: sudo AND user.name: labuser

Phân tích:

User thực hiện sudo

Command được chạy

Timeline hoạt động

Session root được mở khi nào

Trên Endpoint Linux

Kiểm tra auth.log:

grep sudo /var/log/auth.log

Log ghi nhận hành vi lợi dụng find để mở shell quyền root

Kiểm tra session root:

grep "session opened" /var/log/auth.log

Thấy tài khoản labuser đã đăng nhập vào máy Ubuntu qua SSH và đã mở được phiên làm việc với root

Kiểm tra command history:

history

Kiểm tra mở rộng

Kiểm tra có tài khoản backdoor mới hay không.

Kiểm tra cronjob bất thường.

Kiểm tra persistence.

Kiểm tra service lạ.

Kiểm tra thay đổi file hệ thống.

10. Hướng dẫn xử lý sự cố (Response & Mitigation)

Sau khi xác nhận tài khoản labuser đã lợi dụng quyền chạy chương trình find để mở shell có quyền root, đơn vị xử lý sự cố cần ưu tiên bảo toàn bằng chứng, ngăn chặn truy cập tiếp diễn, loại bỏ nguyên nhân và khôi phục hệ thống về trạng thái an toàn. Các lệnh kỹ thuật dưới đây được thực hiện trên máy Ubuntu Web Server, trừ trường hợp có ghi chú khác.

Bước 1. Tiếp nhận và xác nhận sự cố

Nhân viên SOC mở chi tiết cảnh báo Rule 110400 trên Wazuh và xác minh các thông tin chính và đối chiếu cảnh báo với log trên máy chủ.:

Máy chủ bị ảnh hưởng: ubuntu-web-server.

Tài khoản thực hiện: labuser.

Tài khoản đích: root.

Câu lệnh: /usr/bin/find . -exec /bin/bash ; -quit.

Thời gian xảy ra sự kiện và địa chỉ IP đăng nhập SSH.

Các cảnh báo liên quan như đăng nhập SSH, mở phiên root, tạo tài khoản và nhóm mới.

Do tài khoản đã mở được shell root, sự cố cần được ưu tiên xử lý ở mức cao vì người thực hiện có khả năng đọc, sửa hoặc xóa toàn bộ dữ liệu và cấu hình trên máy chủ.

Bước 2: Bảo toàn bằng chứng

Trước khi khóa tài khoản, xóa tệp hoặc khởi động lại máy chủ, cần sao lưu log và các cấu hình liên quan để phục vụ điều tra. Không nên tắt hoặc khởi động lại máy chủ khi chưa thu thập đủ bằng chứng, trừ trường hợp cần ngăn chặn thiệt hại đang tiếp diễn.

Tạo thư mục lưu bằng chứng:

Sao lưu log xác thực và cấu hình sudo:

Thu thập thông tin về phiên đăng nhập, tiến trình và kết nối mạng:

Tạo mã kiểm tra tính toàn vẹn cho các tệp bằng chứng:

Trên Wazuh Dashboard, xuất cảnh báo Rule 110400 và các sự kiện liên quan trong cùng khoảng thời gian. Hồ sơ bằng chứng cần ghi rõ người thu thập, thời gian thu thập, nguồn dữ liệu và mã băm của tệp.

Bước 3: Ngăn chặn truy cập tiếp diễn

Sau khi đã lưu bằng chứng, khóa tài khoản labuser:

Kết thúc các phiên đang hoạt động của tài khoản:

Kiểm tra các phiên đăng nhập còn tồn tại:

Nếu xác định được địa chỉ IP nguồn và việc chặn không ảnh hưởng đến hoạt động quản trị hợp lệ, có thể chặn tạm thời trên tường lửa:

Đối với hệ thống thực tế, cần cô lập máy chủ khỏi các vùng mạng không cần thiết nhưng vẫn duy trì kênh quản trị phục vụ điều tra. Việc cô lập phải được phối hợp với quản trị hệ thống, mạng và chủ quản dịch vụ để tránh gây gián đoạn ngoài kế hoạch.

Bước 4: Loại bỏ nguyên nhân leo thang đặc quyền

Nguyên nhân trực tiếp của sự cố là cấu hình cho phép labuser chạy /usr/bin/find dưới quyền root mà không cần nhập mật khẩu. Xóa cấu hình không an toàn:

Kiểm tra lại cú pháp toàn bộ cấu hình sudo:

Kiểm tra quyền của tài khoản labuser:

Kết quả sau xử lý không được còn quyền: (root) NOPASSWD: /usr/bin/find

Tiếp tục rà soát toàn bộ cấu hình sudo để phát hiện các tài khoản khác có quyền tương tự:

Không chỉ sửa riêng tài khoản labuser; cần xác định liệu cùng cấu hình sai có tồn tại đối với tài khoản hoặc nhóm khác hay không.

Bước 5: Xóa các thay đổi trái phép

Nếu tài khoản attacker1 được tạo trong quá trình hậu khai thác, kiểm tra trước khi xóa:

Sau khi đã lưu bằng chứng, xóa tài khoản và thư mục cá nhân:

Nếu nhóm attacker1 vẫn tồn tại và không phục vụ mục đích hợp lệ, thực hiện:

Kiểm tra các tài khoản khác có UID bằng 0:

Ngoài tài khoản root, nếu xuất hiện tài khoản UID 0 không được phê duyệt thì phải thu thập bằng chứng và loại bỏ theo quy trình quản lý tài khoản của đơn vị.

Bước 6: Kiểm tra cơ chế duy trì truy cập

Sau khi có quyền root, người tấn công có thể tạo cronjob, SSH key, service hoặc tệp khởi động để tiếp tục truy cập. Cần kiểm tra các vị trí sau:

Kiểm tra các tệp được thay đổi trong khoảng thời gian xảy ra sự cố:

Ví dụ, có thể sử dụng khoảng thời gian từ lúc tài khoản đăng nhập SSH đến khi phiên root kết thúc. Không xóa ngay tệp nghi vấn; cần sao lưu, tạo mã băm và xác minh với bộ phận quản trị trước khi loại bỏ.

Bước 7: Thay đổi thông tin xác thực

Đặt lại mật khẩu của labuser và các tài khoản có khả năng bị lộ. Nếu tài khoản không còn nhu cầu sử dụng, tiếp tục duy trì trạng thái khóa hoặc thực hiện xóa theo quy trình quản lý tài khoản.

Kiểm tra và thay mới:

Mật khẩu của tài khoản bị ảnh hưởng.

SSH key đã sử dụng để đăng nhập.

Mật khẩu hoặc khóa của tài khoản quản trị.

Thông tin xác thực được lưu trên máy chủ.

Khóa API, token hoặc mật khẩu dịch vụ có khả năng bị đọc sau khi quyền root bị chiếm.

Việc đổi mật khẩu chỉ được thực hiện sau khi đã loại bỏ cơ chế duy trì truy cập; nếu thực hiện quá sớm, thông tin xác thực mới vẫn có thể tiếp tục bị thu thập.

Bước 8: Khôi phục hệ thống

Do người thực hiện đã có quyền root, không thể chỉ dựa vào việc xóa tài khoản và file sudoers để kết luận hệ thống hoàn toàn an toàn. Nếu không xác định được đầy đủ các thay đổi, ưu tiên khôi phục máy chủ từ snapshot hoặc bản sao lưu tin cậy được tạo trước thời điểm xảy ra sự cố.

Sau khi khôi phục, thực hiện:

Kiểm tra lại tài khoản, nhóm, quyền sudo, cronjob, SSH key, dịch vụ và các cổng mạng đang mở. Chỉ đưa máy chủ trở lại hoạt động khi bộ phận quản trị và an toàn thông tin xác nhận các cấu hình quan trọng đã được kiểm tra.

Bước 9: Giám sát sau khôi phục

Sau khi kết nối lại máy chủ vào hệ thống, tiếp tục giám sát tăng cường trên Wazuh. Theo dõi các cảnh báo liên quan đến:

Đăng nhập SSH của labuser hoặc các tài khoản bất thường.

Hoạt động sudo lên quyền root.

Rule 110400.

Tạo mới hoặc thay đổi tài khoản, nhóm.

Thay đổi file /etc/sudoers và /etc/sudoers.d.

Tạo cronjob, service và SSH key mới.

Thực hiện lại hai trường hợp kiểm thử:

Lệnh thông thường không được kích hoạt Rule 110400.

Trong môi trường kiểm thử được kiểm soát, lệnh trên phải kích hoạt Rule 110400 với mức cảnh báo đã cấu hình.

Bước 10: Báo cáo và rút kinh nghiệm

Sau khi hoàn thành xử lý, lập báo cáo sự cố gồm:

Thời gian phát hiện, xác nhận, ngăn chặn và khôi phục.

Máy chủ, tài khoản và dữ liệu bị ảnh hưởng.

Địa chỉ IP nguồn và phương thức đăng nhập.

Nguyên nhân gốc là cấu hình sudo không an toàn.

Câu lệnh đã được sử dụng để mở shell root.

Các tài khoản, nhóm, tệp hoặc cấu hình đã bị thay đổi.

Biện pháp xử lý và kết quả kiểm tra sau khôi phục.

Bằng chứng đã thu thập và mã băm tương ứng.

Kiến nghị phòng ngừa và người chịu trách nhiệm thực hiện.

Nếu sự cố thuộc phạm vi phải báo cáo hoặc điều phối theo quy định pháp luật, cấp độ hệ thống hoặc phương án ứng cứu đã được phê duyệt, đơn vị phải thông báo cho đầu mối ứng cứu và cơ quan có thẩm quyền theo đúng thời hạn, biểu mẫu và kênh liên lạc quy định.