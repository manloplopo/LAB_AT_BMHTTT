# BÁO CÁO THỰC HÀNH AN TOÀN HỆ THỐNG THÔNG TIN: THIẾT LẬP MÔ HÌNH TƯỜNG LỬA PFSENSE

- **Sinh viên:** Nguyễn Quang Mẫn[cite: 5, 6]
- **Lớp:** 11_THMT[cite: 5, 6]
- **MSSV:** 1150080065[cite: 5, 6]
- **Video minh chứng:** https://youtu.be/8f58ax-TRYc[cite: 5, 6]

---

## 1. MÔ HÌNH HỆ THỐNG & ĐỊNH TUYẾN MẠNG

Hệ thống được ảo hóa trên Oracle VirtualBox với pfSense CE 2.7.2 làm Firewall trung tâm kết nối 3 phân vùng mạng[cite: 4, 5, 6]:
- **WAN Interface:** Card 1 (Bridged Adapter / DHCP), IP: `192.168.1.26/24` kết nối mạng ngoài[cite: 4, 5, 6].
- **LAN Interface:** Card 2 (Host-only Adapter), Subnet: `10.0.0.0/8`, Gateway Firewall: `10.0.0.1`[cite: 4, 5, 6].
- **DMZ Interface:** Card 3 (Internal Network `dmz-net`), Subnet: `172.16.0.0/16`, Gateway Firewall: `172.16.0.1`[cite: 4, 5, 6].

### Thiết bị tham gia thử nghiệm
1. **pfSense Firewall:** Cổng LAN `10.0.0.1/8`, Cổng DMZ `172.16.0.1/16`, Cổng WAN `192.168.1.26`[cite: 4, 5, 6].
2. **Domain Controller (DC):** Windows Server 2022, IP `10.0.0.2/8`, Gateway `10.0.0.1`, DNS `10.0.0.2`[cite: 4, 5, 6].
3. **DMZ-Web:** Windows Server 2022 (IIS Web Server), IP `172.16.0.2/16`, Gateway `172.16.0.1`[cite: 4, 5, 6].
4. **Máy trạm quản trị (Host vật trị):** IP VirtualBox Host-Only `10.0.0.100/8` (không gateway)[cite: 4, 6].

---

## 2. NỘI DUNG CÁC BƯỚC ĐÃ THỰC HIỆN

### Bước 1: Khởi tạo nền tảng (Base Setup)
- Tạo và cài đặt pfSense 2.7.2 trên VirtualBox, gán đúng 3 card mạng tương ứng[cite: 4, 6].
- Đặt IP tĩnh cho cổng LAN `10.0.0.1/8` qua menu console pfSense (tắt DHCP server)[cite: 4, 5, 6].
- Gán card mạng thứ 3 làm `DMZ` (OPT1) với IP `172.16.0.1/16`[cite: 4, 5, 6].
- Kiểm tra chế độ Outbound NAT tự động/Hybrid cho cả LAN và DMZ[cite: 4, 6].
- Dựng máy ảo Windows Server DC (`10.0.0.2`), kích hoạt Active Directory Domain Services và DNS Server[cite: 4, 5, 6].
- Chuẩn hóa ruleset LAN: Tắt 2 rule mặc định `Default allow LAN to any`, tạo Rule nền tảng (`LAN net -> Any`) và thử nghiệm Reset States[cite: 4].

### Bước 2: Tình huống 1 — Chặn ICMP nhưng cho phép Web/DNS
- Vô hiệu hóa rule nền tảng `LAN net -> Any`[cite: 4].
- Cấu hình cụm 3 rule trên tab LAN[cite: 4]:
  1. `Block ICMP` (chặn Ping ra ngoài)[cite: 4].
  2. `Pass TCP/UDP 53` (cho phép DNS ra ngoài)[cite: 4].
  3. `Pass TCP 80, 443` (cho phép duyệt Web HTTP/HTTPS)[cite: 4].
- **Kết quả:** Lệnh `ping 8.8.8.8` bị chặn hoàn toàn (Request timed out), nhưng phân giải `nslookup google.com` và duyệt web thành công[cite: 4, 5, 6].

### Bước 3: Tình huống 2 — Kiểm soát truy cập máy trạm LAN
- Thiết lập chính sách kiểm soát địa chỉ IP nguồn: Chỉ cho phép máy Domain Controller (`10.0.0.2`) được phép ra mạng, chặn toàn bộ các máy trạm còn lại (`LAN net`)[cite: 4, 6].

### Bước 4: Tình huống 3 — Triển khai Web Server DMZ & NAT Port Forwarding
- Nhân bản (Clone) máy ảo thành `DMZ-Web`, chuyển card mạng sang `Internal Network` (`dmz-net`) và đặt IP tĩnh `172.16.0.2/16`[cite: 1, 4, 6].
- Cài đặt dịch vụ IIS (Internet Information Services) lắng nghe cổng `80` (`http://localhost`)[cite: 1, 4, 5, 6].
- Tạo rule **NAT Port Forwarding** trên pfSense: Ánh xạ cổng HTTP từ `WAN address:80` (hoặc 8080) trỏ về `172.16.0.2:80`[cite: 4, 6].
- Bỏ tích chọn `Block private networks` trên giao diện WAN của pfSense[cite: 4, 5, 6].
- **Kết quả:** Trình duyệt trên máy vật lý bên ngoài truy cập `http://192.168.1.26` hiển thị thành công trang chủ IIS mặc định[cite: 4, 5, 6].

### Bước 5: Tình huống 4 — Phân lập và bảo vệ an ninh vùng DMZ
- Cấu hình rule trên tab DMZ: Tạo rule `Block Any` từ nguồn `DMZ net` đi tới đích `LAN net`[cite: 4, 5, 6].
- Cấu hình rule trên tab LAN: Cho phép người dùng LAN kết nối cổng `80` sang máy chủ DMZ[cite: 4, 6].
- **Kết quả:** Đứng từ máy `DMZ-Web` (`172.16.0.2`) ping sang Domain Controller (`10.0.0.2`) bị chặn hoàn toàn; ngược lại từ máy DC truy cập Web DMZ (`http://172.16.0.2`) thành công[cite: 4, 5, 6].

### Bước 6: Tình huống 5 — Bật Logging và đọc Firewall Log
- Vào rule `Block DMZ to LAN`, tích chọn `Log packets that are handled by this rule` và bấm `Apply Changes`[cite: 4, 5, 6].
- Thực hiện `Reset States`[cite: 4].
- Từ máy DMZ thực hiện lệnh ping sang `10.0.0.2`[cite: 4, 5, 6].
- Vào **Status -> System Logs -> Firewall** để kiểm tra và xác nhận bản ghi dấu `X` đỏ ghi nhận lưu lượng ICMP bị chặn[cite: 4, 6].

---

## 3. CÁC KHÓ KHĂN KỸ THUẬT GẶP PHẢI & CÁCH GIẢI QUYẾT

| STT | Vấn đề / Triệu chứng lỗi | Nguyên nhân kỹ thuật | Cách xử lý đã thực hiện |
| :--- | :--- | :--- | :--- |
| **1** | Máy ảo DMZ ping gateway `172.16.0.1` bị báo `Destination host unreachable` / `Request timed out`. | Lệch tên switch ảo `Internal Network` trong VirtualBox hoặc sai Subnet Mask `/16` (`255.255.0.0`)[cite: 1, 4]. | Đồng bộ tên card mạng ở cả pfSense và DMZ-Web thành `dmz-net`[cite: 1, 4, 6]. Sửa Subnet Mask trên Windows Server thành `255.255.0.0`[cite: 1, 4]. |
| **2** | Không ping được cổng LAN `10.0.0.1` từ máy Domain Controller[cite: 1]. | Do đã tắt rule mặc định `Default allow LAN` và chỉ mở Web/DNS, giao thức ICMP ping vào pfSense bị chặn ngầm[cite: 1, 4]. | Bổ sung rule trên tab LAN: `Pass IPv4 ICMP` với `Destination: LAN address`[cite: 1]. |
| **3** | Cấu hình nhầm máy Domain Controller thành DMZ Web Server[cite: 1]. | Thay đổi trực tiếp card mạng và IP trên máy DC gốc làm hỏng hệ thống LAN[cite: 1]. | Đưa máy DC về card `Host-only`, gán lại IP `10.0.0.2/8`, sau đó sử dụng tính năng `Linked Clone` để tạo máy `DMZ-Web` độc lập[cite: 1, 4, 6]. |
| **4** | Laptop thật bị mất kết nối Internet khi đang làm lab. | Card `VirtualBox Host-Only Ethernet Adapter` trên máy thật bị điền Default Gateway `10.0.0.1`, làm sai lệch bảng định tuyến của hệ điều hành host[cite: 4]. | Vào `ncpa.cpl` trên máy thật, chỉnh card ảo Host-Only thành để trống hoàn toàn phần Default Gateway và DNS[cite: 4]. |
| **5** | Không thấy gói tin ICMP của DMZ trong bảng Firewall Log ở Tình huống 5[cite: 6]. | Sau khi sửa rule Block có bật log, chưa nhấn nút `Apply Changes` và chưa `Reset States` nên rule chưa nạp vào kernel[cite: 4, 6]. | Nhấn nút xanh `Apply Changes`, thực hiện xóa bảng trạng thái tại `Diagnostics -> States -> Reset States` rồi thử nghiệm lại lệnh ping[cite: 4]. |
