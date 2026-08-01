
> 🎬 **Video hướng dẫn / diễn tập (Lab4):** [https://drive.google.com/file/d/1gkWVBIK7Ll_VuUY4e7tdkfzmq87pIQrQ/view?usp=sharing](https://drive.google.com/file/d/1gkWVBIK7Ll_VuUY4e7tdkfzmq87pIQrQ/view?usp=sharing)

Kịch bản 4:  BRUTE-FORCE SSH-RDP

1. Mô tả tình huống

Kẻ tấn công sử dụng công cụ Hydra thực hiện tấn công Brute Force nhằm dò quét tài khoản và mật khẩu của dịch vụ SSH trên máy Ubuntu và dịch vụ Remote Desktop (RDP) trên máy Windows Server.

Trong quá trình tấn công, Wazuh Agent sẽ thu thập log xác thực từ các máy chủ và gửi về Wazuh Server để phân tích, phát hiện và sinh cảnh báo.

2. MÔ HÌNH TRIỂN KHAI LAB

Luồng dữ liệu

Kali Linux

Ubuntu SSH Server / Windows RDP Server

Wazuh Agent

Wazuh Manager

Wazuh Dashboard

3. CHUẨN BỊ MÔI TRƯỜNG LAB

3.1 Trên Wazuh Server

Kiểm tra trạng thái Manager

    	systemctl status wazuh-manager

Kiểm tra Dashboard

    	systemctl status wazuh-dashboard

Kiểm tra Indexer

    	systemctl status wazuh-indexer

Kiểm tra Agent

    	/var/ossec/bin/agent_control -l

Đảm bảo Ubuntu Agent và Windows Agent ở trạng thái Active.

3.2 Trên Ubuntu Server

Kiểm tra SSH

    	sudo systemctl status ssh

Nếu SSH chưa hoạt động

sudo systemctl enable ssh

sudo systemctl start ssh

Kiểm tra cổng SSH

    	ss -tunlp | grep 22

Kiểm tra file log

    	tail -f /var/log/auth.log

3.3 Trên Windows Server

Bật Remote Desktop

System Properties

Remote

Enable Remote Desktop

Kiểm tra Firewall

    	Get-NetFirewallRule -DisplayGroup "Remote Desktop"

3.4 Trên Kali Linux

Kiểm tra Hydra

    	hydra -h

Chuẩn bị wordlist

    	ls

Ví dụ

10_million_password_list_top_1000000.txt

Nếu chưa có thì down: “ git clone https://github.com/kkrypt0nn/wordlists.git ”

cd ~/wordlists/wordlists/usernames

ls /usr/share/wordlists

sudo gzip -d /usr/share/wordlists/rockyou.txt.gz

4. CẤU HÌNH MÔI TRƯỜNG BAN ĐẦU

Ubuntu

Đăng nhập thử SSH

    	ssh root@192.168.174.31

Thử kết nối RDP

nmap -p3389 192.168.174.163

Đảm bảo dịch vụ hoạt động bình thường.

5. HƯỚNG DẪN TẤN CÔNG (ATTACKER COMMAND)

5.1 Brute Force RDP

Trên Kali Linux

hydra -t 1 -W 3 \-l Administrator \-P /usr/share/wordlists/rockyou.txt \rdp://192.168.174.129

Đợi Hydra thực hiện nhiều lần đăng nhập thất bại.

5.2 Kiểm tra Event Windows

Mở

Event Viewer

Windows Logs

Security

Lọc Event

4625

5.3 Brute Force SSH

hydra \

-l ubuntu \

-P /usr/share/wordlists/rockyou.txt \

ssh://192.168.174.131

Đợi Hydra hoàn thành.

5.4 Kiểm tra log Ubuntu

    	tail -f /var/log/auth.log

Kiểm tra số lượng đăng nhập thất bại

    	grep "Failed password" /var/log/auth.log

6. KIỂM TRA TRÊN SIEM

Đăng nhập Dashboard

https://192.168.174.25

Mở

Security Events

Lọc Event Windows

data.win.system.eventID:4625

Lọc địa chỉ IP nguồn

data.srcip:192.168.174.131

Kiểm tra số lượng Event phát sinh.

7. XÂY DỰNG RULE TRÊN SIEM

Mở file

nano /var/ossec/etc/rules/local_rules.xml

Thêm Rule phát hiện SSH Failed

<group name="local,custom_bruteforce,">

     <!-- SSH Brute Force - High Severity                	-->

    <!-- Trigger khi Wazuh đã phát hiện SSH Brute Force 	-->

    <rule id="100600" level="15">

        <if_sid>5763</if_sid>

        <description>

        	[CUSTOM] Critical SSH Brute Force Attack Detected

        </description>

        <group>

            ssh,bruteforce,attack,custom,

        </group>

        <mitre>

        	<id>T1110</id>

        </mitre>

    </rule>

    <!-- SSH Root Brute Force                          	-->

    <rule id="100601" level="16">

        <if_sid>5760</if_sid>

        <match>root</match>

        <description>

        	[CUSTOM] SSH Root Account Brute Force

        </description>

        <group>

        	ssh,root,attack,custom,

        </group>

    </rule>

    <!-- Windows RDP Brute Force                           -->

    <rule id="100602" level="15">

        <if_sid>60204</if_sid>

        <description>

        	[CUSTOM] Critical Windows RDP Brute Force

        </description>

        <group>

        	windows,rdp,bruteforce,custom,

        </group>

        <mitre>

        	<id>T1110</id>

        </mitre>

    </rule>

    <!-- Administrator Failed Login                    	-->

    <rule id="100603" level="13">

        <if_sid>60122</if_sid>

        <field name="win.eventdata.targetUserName">

        	Administrator

        </field>

        <description>

        	[CUSTOM] Administrator Login Failure

        </description>

        <group>

        	windows,administrator,custom,

        </group>

    </rule>

</group>

Lưu file.

Khởi động lại Wazuh

systemctl restart wazuh-manager

Kiểm tra cấu hình

/var/ossec/bin/wazuh-analysisd -t

8. TẤN CÔNG SAU KHI XÂY DỰNG RULE

Thực hiện lại Brute Force SSH

hydra \

-l ubuntu \

-P /usr/share/wordlists/rockyou.txt \

ssh://192.168.174.131

Thực hiện lại Brute Force RDP

hydra -t 1 -W 3 \-l Administrator \-P /usr/share/wordlists/rockyou.txt \rdp://192.168.174.129

Theo dõi Alert

tail -f /var/ossec/logs/alerts/alerts.json

Kiểm tra Dashboard.

9. HƯỚNG DẪN XỬ LÝ

Trên Ubuntu

Chặn IP tấn công

sudo iptables -A INPUT -s 192.168.174.132 -j DROP

Hoặc

sudo ufw deny from 192.168.174.162

Khóa tài khoản

sudo passwd -l root

Trên Windows

Chặn IP

New-NetFirewallRule `

-DisplayName "Block Attacker" `

-Direction Inbound `

-RemoteAddress 192.168.174.132 `

-Action Block

Khóa tài khoản

Computer Management

Local Users and Groups

Users

Administrator

 Disable Account

Kiểm tra kết quả

Thực hiện lại Brute Force từ Kali Linux.

Đảm bảo Hydra không thể kết nối đến dịch vụ SSH hoặc RDP.

Kiểm tra Dashboard xác nhận không còn phát sinh phiên đăng nhập thành công và các kết nối từ địa chỉ IP tấn công đã bị chặn.