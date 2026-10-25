# LAB 7 – Web Protection trên pfSense + Web Proxy

- **Sinh viên:** Nguyễn Quang Mẫn – MSSV 1150080065
- **Lớp:** 11_THMT
- **Hệ điều hành WEB-PROXY:** Ubuntu 26.04 (resolute), theo quy định của giảng viên
- **Trạng thái:** Chưa hoàn thành. Dừng ở **Phần C.1 – Chuẩn bị WEB-PROXY**, chưa thực hiện được kịch bản nào trong Phần F.

---

## 1. Tiến độ thực hiện

| Phần | Nội dung | Trạng thái |
|---|---|---|
| B | Kiểm tra môi trường (VM, IP, DNS) | Đã tạo VM WEB-PROXY, IP 10.6.0.30/24. **Chưa thông mạng với pfSense.** |
| C.1 | Cài Squid, E2Guardian, ClamAV | **Chưa cài được** (không có mạng, gõ sai tên gói) |
| C.2 | Luồng E2Guardian → Squid | Chưa thực hiện (Squid và E2Guardian chưa có) |
| C.3–C.5 | Website mô phỏng, DNS `lab7.test`, cấu hình proxy trình duyệt | Chưa thực hiện |
| D | Alias và rule pfSense ép lưu lượng web qua proxy | Chưa thực hiện |
| E | Chính sách Web (block, warning, exception, schedule, ClamAV, size) | Chưa thực hiện |
| F | 10 kịch bản kiểm thử | Chưa thực hiện kịch bản nào |
| G | HTTPS inspection (mở rộng) | Chưa thực hiện |

---

## 2. Các khó khăn gặp phải và lý do không thực hiện được

### Khó khăn 1: WEB-PROXY không ping được gateway pfSense (10.6.0.1)

- **Hiện tượng:** `ping -c 5 10.6.0.1` từ WEB-PROXY báo `Destination Host Unreachable`, mất 100% gói. DC01 cũng không ping được 10.6.0.1. Lệnh `ip neigh` trên WEB-PROXY cho thấy `10.6.0.1 dev enp0s3 FAILED`, trong khi `10.6.0.10` (DC01) có địa chỉ MAC và trạng thái `STALE`.
- **Kết quả kiểm tra thêm:**
  - Card `enp0s3` của WEB-PROXY ở trạng thái UP, IP `10.6.0.30/24` đúng.
  - Trên pfSense, `ifconfig em1` cho thấy `inet 10.6.0.1 netmask 255.255.255.0`, `status: active`.
  - Trên pfSense, `ping -c 3 10.6.0.10` và `ping -c 3 10.6.0.30` đều mất 100% gói, bảng `arp -an` của em1 chỉ có địa chỉ của chính pfSense.
- **Nguyên nhân xác định:** Lỗi ở lớp 2. Card LAN (em1) của pfSense không nằm cùng mạng ảo (Internal Network) với DC01 và WEB-PROXY, nên các máy không nhìn thấy nhau, dù IP đã đặt đúng.
- **Hậu quả:** WEB-PROXY không đi qua pfSense được, nên không ra Internet để tải gói cài đặt.

### Khó khăn 2: Không phân giải được tên miền, `apt update` thất bại

- **Hiện tượng:** `ping vn.archive.ubuntu.com` báo `Temporary failure in name resolution`. `sudo apt update` báo lỗi `Temporary failure resolving 'vn.archive.ubuntu.com'` và `'security.ubuntu.com'`.
- **Nguyên nhân:** Hệ quả trực tiếp của Khó khăn 1. WEB-PROXY không tới được gateway pfSense và DNS server (DC01) theo đường mạng đúng, nên không resolve được tên miền và không tải được danh sách gói.
- **Hậu quả:** Không cài được bất kỳ gói nào.

### Khó khăn 3: Cài đặt gói thất bại

- **Hiện tượng:** Lệnh `sudo apt install -y squid e2guardianc clamav clamav-daemon curl openssl` báo `Error: Unable to locate package e2guardianc`.
- **Nguyên nhân:**
  - Gõ sai tên gói (`e2guardianc`, tên đúng là `e2guardian`).
  - Danh sách gói chưa được cập nhật vì `apt update` thất bại (Khó khăn 2).
- **Hậu quả:** Squid, E2Guardian và ClamAV chưa được cài.

### Khó khăn 4: Squid và E2Guardian chưa có nên không khởi động được dịch vụ

- **Hiện tượng:**
  - `sudo systemctl restart squid` báo `Unit squid.service not found`.
  - `sudo systemctl restart e2guardian` báo `Unit e2guardian.service not found`.
  - `ss -tlpn | grep -E '8080|3128'` không có kết quả nào.
  - `curl -x http://10.6.0.30:8080 http://example.com` (và facebook.com, eicar.org) báo `Failed to connect ... Could not connect to server`.
- **Nguyên nhân:** Do Khó khăn 3, hai dịch vụ chưa được cài. File `squid.conf` đã được soạn trước nhưng không có dịch vụ nào đọc nó.
- **Hậu quả:** Cổng 8080 và 3128 chưa mở, nên không có proxy để Client-A/B kết nối, không thể thực hiện các kịch bản kiểm thử.

### Khó khăn 5: Máy ảo chạy chậm, báo lỗi `soft lockup`

- **Hiện tượng:** Console của WEB-PROXY liên tục báo `watchdog: BUG: soft lockup - CPU#0 stuck for ...s! [swapper/0:0]`.
- **Nguyên nhân (nghi ngờ):** Máy thật chạy nhiều máy ảo cùng lúc (pfSense, DC01, WEB-PROXY, các client), thiếu tài nguyên CPU/RAM. Đây chỉ là nghi ngờ, chưa có xác nhận.
- **Hậu quả:** Máy ảo phản hồi chậm, các lệnh bị treo, mất thời gian thao tác và kiểm tra.

### Khó khăn 6: pfSense mất card mạng sau khi chỉnh cấu hình VirtualBox

- **Hiện tượng:** Sau khi chỉnh card mạng của VM pfSense để thử sửa Khó khăn 1, pfSense khởi động lại và báo `Configuration references interfaces that do not exist: em3`, `Network interface mismatch -- Running interface assignment option`. Chỉ còn 3 card (em0, em1, em2) thay vì 4 như ban đầu, trong đó card quản trị 192.168.56.254 không còn.
- **Nguyên nhân (nghi ngờ):** Một adapter của VM pfSense bị tắt hoặc bị đổi trong lúc chỉnh, làm thứ tự em0–em3 bị lệch so với cấu hình LAB 6 đã lưu.
- **Hậu quả:** Cấu hình giao diện pfSense bị lệch, cần khôi phục đủ 4 adapter trước khi tiếp tục các phần sau.

---

## 3. Tóm tắt nguyên nhân gốc

1. **Mạng ảo LAN của pfSense (em1) không cùng Internal Network với DC01 và WEB-PROXY**, nên WEB-PROXY không tới được gateway, không có DNS, không ra Internet.
2. **Không có mạng nên không cài được Squid, E2Guardian, ClamAV**. Kèm theo đó là một lỗi gõ sai tên gói.
3. **Chưa có proxy chạy** nên toàn bộ Phần D, E, F phụ thuộc vào proxy chưa thể thực hiện.

---

## 4. Hướng khắc phục dự kiến

1. Khôi phục đủ 4 adapter cho VM pfSense (em0 NAT, em1 LAN, em2 quản trị, em3 DMZ) theo đúng địa chỉ MAC ban đầu.
2. Đổi Adapter 2 (LAN) của pfSense sang đúng tên Internal Network của DC01 và WEB-PROXY, đặt Promiscuous Mode = Allow All, tick Cable Connected.
3. Kiểm tra ping giữa pfSense, DC01 và WEB-PROXY, sau đó ping `8.8.8.8` và tên miền từ WEB-PROXY.
4. Chạy `sudo apt update`, bật Universe, rồi cài `squid e2guardian clamav clamav-daemon curl openssl` (gõ đúng tên gói). Nếu gói `e2guardian` không có trong kho của Ubuntu 26.04 thì xin hướng dẫn của giảng viên.
5. Cấu hình Squid nghe `127.0.0.1:3128`, E2Guardian nghe `10.6.0.30:8080`, rồi tiếp tục Phần C.3 trở đi.

---

## 5. Minh chứng đính kèm trong báo cáo

- Ảnh 1: `ping` gateway thất bại, `apt update` lỗi DNS, `Unable to locate package e2guardianc`.
- Ảnh 2: nội dung `squid.conf` và lỗi `squid.service not found`.
- Ảnh 3: lỗi `e2guardian.service not found`, `ss -tlpn` không có cổng 8080/3128, `curl` qua proxy thất bại.
