# BÁO CÁO THỰC HÀNH LAB 1: BẮT VÀ PHÂN TÍCH GÓI TIN TELNET - SSH (WIRESHARK)

## 1. THÔNG TIN SINH VIÊN
* **Họ và tên:** Nguyễn Quang Mẫn
* **Mã số sinh viên:** 1150080065
* **Lớp:** 11_THMT
* **Môn học:** An toàn Hệ thống Thông tin
* **Tên bài Lab:** Lab 1 - Bắt và phân tích gói tin Telnet & SSH trên Wireshark (Examining SSH & Telnet in Wireshark)
* **Link Video Demo (YouTube):** `https://youtu.be/SqcLj2O8-30`
## 2. TỔNG QUAN & MỤC TIÊU BÀI LAB
* Thiết lập môi trường kết nối giữa Client (Windows Host) và Server (Kali Linux trên VirtualBox).
* Khởi tạo dịch vụ truyền thông điều khiển từ xa không mã hóa (Telnet - Port 23) và có mã hóa (SSH - Port 22).
* Sử dụng công cụ Wireshark (Adapter for loopback traffic capture / Virtual interface) để bắt, bóc tách và phân tích các luồng gói tin TCP Stream.
* Đánh giá trực quan sự khác biệt cốt lõi về mặt an toàn thông tin (Confidentiality, Integrity, Authentication) giữa Telnet và SSH.
## 3. MÔI TRƯỜNG & THÔNG SỐ KỸ THUẬT THỰC HÀNH
* **Máy trạm (Client / Attacker):** 
  * OS: Windows 11 (Host OS)
  * Phần mềm sử dụng: Command Prompt, Wireshark (v4.x), PuTTY / SSH Client
  * Địa chỉ IP Host / Loopback: `192.168.11.190` / Virtual Host-Only / Loopback
* **Máy chủ (Server):**
  * OS: Kali Linux 6.19.14 (VirtualBox Guest)
  * Dịch vụ kích hoạt: Telnet Server (Port 23), OpenSSH Server (Port 22)
  * Tài khoản kiểm thử: 
    * User 1: `kali`
    * User 2 (Tạo mới theo bài Lab): `uitlab` (Full name: `nguyenquangman`)
## 4. NỘI DUNG ĐÃ THỰC HIỆN
1. **Khởi tạo và kiểm tra kết nối mạng:**
   * Dùng lệnh `ip a` trên Kali Linux kiểm tra giao diện mạng `eth0` (`192.168.11.190/24`).
   * Thực hiện `ping 192.168.11.190` từ Windows Command Prompt -> Kết quả phản hồi TTL=64, mất gói 0% (RTT < 1ms).
2. **Cấu hình tài khoản và dịch vụ:**
   * Tạo tài khoản `uitlab` bằng lệnh `sudo adduser uitlab` (Full Name: `nguyenquangman`).
   * Kiểm tra port dịch vụ Telnet đang lắng nghe bằng lệnh: `ss -ltn | grep -E ':23'`.
3. **Bắt và phân tích gói tin Telnet:**
   * Kích hoạt Wireshark bắt luồng mạng với filter `tcp.port == 23`.
   * Thực hiện kết nối từ Client bằng lệnh `telnet` và nhập thông tin tài khoản đăng nhập.
   * Sử dụng tính năng **Follow TCP Stream** trên Wireshark để khôi phục dữ liệu truyền tải.
4. **Bắt và phân tích gói tin SSH:**
   * Kích hoạt Wireshark với filter `tcp.port == 22`.
   * Thực hiện kết nối SSH từ máy trạm vào server và thao tác các lệnh (`uname -a`, `id`, `ls`).
   * Khảo sát payload của gói tin SSH v2 (Encrypted packet payload).
## 5. KẾT QUẢ THỰC HIỆN & SO SÁNH
* **Kết quả Telnet (Port 23):**
  * Toàn bộ quá trình handshake, chuỗi banner chào mừng (`Welcome to Linux...`), quá trình đăng nhập (gồm cả tài khoản gõ sai `kali login: nguyenquangman`, các lần nhập password) và nội dung lệnh thực thi đều hiển thị hoàn toàn dưới dạng văn bản rõ (**Plaintext / Cleartext**).
  * Kẻ tấn công trên đường truyền có thể đọc được 100% dữ liệu nhạy cảm mà không cần khóa giải mã.
* **Kết quả SSH (Port 22):**
  * Quá trình thương lượng thuật toán (Key Exchange, Diffie-Hellman) diễn ra công khai ở bước đầu, nhưng ngay sau khi phiên khóa mã hóa đối xứng được thiết lập, toàn bộ dữ liệu (Payload, User, Password, Shell Session) đều chuyển thành chuỗi nhị phân mã hóa vô nghĩa (**Encrypted Packet**).
  * Wireshark chỉ quan sát được metadata tầng mạng và tầng giao vận (Source IP, Destination IP, Port 22, kích thước packet, timestamp), hoàn toàn **không thể giải mã hay đọc trộm được nội dung**.
## 6. LƯU Ý KHI KIỂM TRA & CHẠY LẠI BÀI LAB
* Đảm bảo cấu hình Card mạng máy ảo (Host-Only Adapter hoặc Bridged Adapter) để máy Host Windows và Kali Linux có thể ping thấy nhau.
* Khi dùng Wireshark trên Windows để bắt kết nối loopback/máy ảo cục bộ, chọn đúng Adapter card mạng tương ứng (`Npcap Loopback Adapter` hoặc card ảo VirtualBox Host-Only Ethernet Adapter).
* Kiểm tra tường lửa (`ufw` hoặc `iptables` trên Linux) không chặn cổng TCP 22 và TCP 23 trong suốt quá trình đo kiểm.
