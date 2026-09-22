# BÁO CÁO BÀI TẬP THỰC HÀNH LAB 3
**Học phần:** An toàn và Bảo mật Hệ thống Thông tin  
**Giảng viên hướng dẫn:** ThS. Phạm Trọng Huynh  

---

## 1. Thông tin sinh viên
- **Họ và tên:** Nguyễn Quang Mẫn
- **Mã số sinh viên (MSSV):** 1150080065
- **Mã lớp:** 11_THMT
- **Tên bài lab:** Lab 3 - Nhận diện và ứng phó các mối đe dọa đến an toàn thông tin (Identifying and Responding to Information Security Threats)
- **Repository:** [manloplopo/LAB_AT_BMHTTT](https://github.com/manloplopo/LAB_AT_BMHTTT)

---

## 2. Phiên bản môi trường thực hành
- **Nền tảng ảo hóa:** VMware Workstation Pro
- **Hệ điều hành máy ảo (Guest OS):** Windows Server 2025 Datacenter Evaluation (64-bit), OS Build 26100.ge_release.240331-1435
- **Cấu hình mạng:** Host-only / Loopback adapter
- **Môi trường dòng lệnh:** Windows PowerShell 5.1 (Run as Administrator)
- **Phiên bản các phần mềm & công cụ giám sát:**
  - Python: `3.14.0`
  - Wireshark / TShark: `4.6.8` (tích hợp Npcap loopback capture)
  - Microsoft Sysinternals Sysmon: `15.22` (Schema 4.90)
  - Microsoft Sysinternals Autoruns: `14.3`
  - Microsoft Sysinternals Process Explorer: `17.14`
  - Microsoft Defender Antivirus: `RealTimeProtectionEnabled = True`, `IsTamperProtected = True`

---

## 3. Cách dựng môi trường
1. **Thiết lập VM & Mạng:** Tạo máy ảo Windows Server 2025 trên VMware Workstation, đặt chế độ card mạng ở trạng thái `Host-only` theo mặc định.
2. **Khởi tạo cấu trúc thư mục làm việc:** Tạo cấu trúc thư mục tiêu chuẩn tại `C:\LAB3` gồm các nhánh: `Assets`, `Downloads`, `Evidence`, `Tools` và xuất mốc thời gian bắt đầu vào `start_time.txt`.
3. **Tiếp nhận gói tài nguyên:** Đưa gói `LAB3_Threats_Assets.zip` vào `C:\LAB3\Downloads` và giải nén sang thư mục `C:\LAB3`.
4. **Cài đặt công cụ phân tích:** Cài đặt Python 3.14 và Wireshark 4.6.8 có thành phần Npcap loopback adapter.
5. **Cài đặt bộ công cụ Sysinternals:** Tải trực tiếp các tệp lưu trữ từ máy chủ Microsoft (`download.sysinternals.com`), giải nén vào `C:\LAB3\Tools` và kiểm tra xác thực phiên bản công cụ.

---

## 4. Các tình huống thực hiện và kết quả

| Tình huống | Nội dung thực hiện | Bằng chứng telemetry / Output chính | Kết quả |
| :--- | :--- | :--- | :---: |
| **TH1** | Thu thập trạng thái Baseline hệ thống & Lập Risk Register | Xuất các tệp `baseline_os.txt`, `baseline_defender.txt`, `baseline_firewall.txt`, `baseline_network.txt`, `baseline_processes.txt`. Phân loại 5 nguồn đe dọa. | **PASS** |
| **TH2** | Kiểm chứng cơ chế phát hiện mã độc bằng chuỗi EICAR | Tệp `eicar.com` được tạo ra; Defender kích hoạt cơ chế phòng vệ và chặn tệp ngay khi ghi xuống đĩa, ghi log phát hiện. | **PASS** |
| **TH3** | Tấn công mật khẩu, Audit Logon & Credential Rotation | Bật Audit Logon Policy; tạo tài khoản `lab3user`; sinh Event 4624/4648 (thành công) và Event 4625 (nhập sai mật khẩu); đổi mật khẩu mới làm mất hiệu lực mật khẩu cũ. Log: `auth_events_before_rotation.txt`. | **PASS** |
| **TH4** | Nhận diện Persistence (Backdoor) & Tiến trình mở cổng | Nạp cấu hình Sysmon; tạo khóa Run Registry `LAB3_Run_Demo` và Scheduled Task `LAB3_Persistence_Demo` (sinh tệp `task_ran.txt`); phát hiện qua Autoruns; chạy HTTP listener 8080 trên `127.0.0.1` và ánh xạ chính xác về PID của tiến trình `python.exe` bằng Process Explorer. | **PASS** |
| **TH5** | Sniffing mạng nội bộ & So sánh HTTP / HTTPS | Bắt gói tin trên loopback bằng Wireshark; phân tích luồng TCP Stream thấy rõ plaintext HTTP payload và chuỗi truy vấn; đối chiếu với gói tin TLSv1.3 đã mã hóa toàn bộ dữ liệu ứng dụng. File: `network_traffic_analysis.pcapng`. | **PASS** |
| **TH6** | Phân tích tấn công DoS, DDoS và Mail Bombing | Phân biệt tấn công DoS đơn nguồn với DDoS đa nguồn phân tán; phân tích thống kê tần suất người gửi và dung lượng đột biến qua tập log offline mẫu. | **PASS** |
| **TH7** | Phân loại Kỹ nghệ xã hội (Social Engineering) & Phishing | Nhận diện 5 chỉ dấu trong mẫu email giả mạo (`phishing_email.txt`); lập bảng phân loại 6 kịch bản lừa đảo; phân tích các cơ chế phòng chống. | **PASS** |
| **Cleanup** | Cô lập, dọn dẹp, kiểm tra phục hồi & Băm tính toàn vẹn | Gỡ bỏ khóa Run, hủy Scheduled Task, dừng tiến trình HTTP server, xóa tài khoản lab; so sánh chênh lệch cấu hình Autoruns (`autoruns_diff.txt`); băm mã SHA-256 toàn bộ thư mục `Evidence` vào `evidence_sha256.csv`. | **PASS** |

---

## 5. Lỗi kỹ thuật gặp phải và cách khắc phục
1. **Sự cố đứng máy (Treo hệ thống máy ảo):**
   - *Hiện tượng:* Trong quá trình chạy các tác vụ nền và ghi log, máy ảo bị treo cứng (freeze), không thể thao tác qua chuột và bàn phím.
   - *Cách khắc phục:* Tiến hành giữ nút nguồn trên thanh công cụ VMware để Hard Reset máy ảo; sau khi khởi động lại, tiếp tục các kịch bản tiếp theo và lưu log theo từng bước nhỏ để tránh mất dữ liệu.
2. **Lỗi gõ nhầm lệnh trong PowerShell (`Select-Oject`):**
   - *Hiện tượng:* Xuất hiện lỗi `CommandNotFoundException` khi kiểm tra phiên bản TShark do gõ thiếu chữ cái.
   - *Cách khắc phục:* Chỉnh sửa lại đúng cú pháp `Select-Object`.
3. **Lỗi tham số Scheduled Task (`New-ScheduledTaskTrigger -AtLogOn`):**
   - *Hiện tượng:* Gặp lỗi ép kiểu dữ liệu `ParameterBindingArgumentTransformationException` do truyền nhầm tham số `$env:USERNAME` vào sau `-AtLogOn`.
   - *Cách khắc phục:* Khai báo tham số `-AtLogOn` độc lập theo đúng tài liệu chuẩn của PowerShell.
4. **Xung đột tệp khi xuất mã băm SHA-256 (`evidence_sha256.csv`):**
   - *Hiện tượng:* Xuất hiện lỗi `The process cannot access the file because it is being used by another process` do `Get-ChildItem` quét trúng chính file `evidence_sha256.csv` mà lệnh `Export-Csv` đang mở để ghi.
   - *Cách khắc phục:* Bổ sung tham số loại trừ `-Exclude "evidence_sha256.csv"` vào lệnh `Get-ChildItem` để tránh xung đột quyền truy cập tệp.

---

## 6. Ghi chú bảo mật & Tuân thủ quy định bài nộp
- Toàn bộ các thử nghiệm chỉ giới hạn trên địa chỉ loopback cục bộ `127.0.0.1` của máy ảo lab.
- Không thực hiện tấn công ra mạng bên ngoài, không phá hoại hệ thống thật.
- Đã kiểm tra và loại bỏ hoàn toàn mật khẩu thực, thông tin định danh cá nhân nhạy cảm trong các tệp log và tệp báo cáo trước khi đưa lên repository.
- Repository không chứa các tệp cài đặt `.exe`, `.msi`, binary của Sysinternals/Wireshark hoặc file bị Defender cách ly theo đúng quy ước an toàn.
