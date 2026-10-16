# LAB 6: TỔNG HỢP LỖI THƯỜNG GẶP VÀ CÁCH XỬ LÝ TRÊN pfSense & ACTIVE DIRECTORY

Tài liệu này tổng hợp các lỗi kỹ thuật phổ biến thường gặp trong quá trình thực hiện **Lab 6 (Xác thực Active Directory và Chính sách truy cập trên pfSense)** kèm theo nguyên nhân và các bước khắc phục chi tiết.

---

## 1. Lỗi xác thực Active Directory qua LDAP (`Authentication Failed`)

### Biểu hiện
* Khi kiểm tra tại `Diagnostics > Authentication` hoặc khi đăng nhập Captive Portal, hệ thống báo lỗi xác thực dù tài khoản và mật khẩu trên `DC01` hoàn toàn chính xác.
* Trạng thái trả về từ pfSense báo lỗi kết nối hoặc không tìm thấy thông tin người dùng (`User not found`).

### Nguyên nhân phổ biến
1. **Sai cấu hình Base DN hoặc Bind Credentials:** Tài khoản dùng để Bind LDAP (`svc_pfsense_ldap`) không đủ quyền hoặc định dạng Base DN (`DC=lab6,DC=test`) bị nhập sai chính tả.
2. **Cổng kết nối LDAP không phù hợp:** Sử dụng cổng LDAPS (636) nhưng chưa cấu hình chứng chỉ hoặc CA, trong khi chưa bật hoặc cấu hình đúng cổng LDAP thường (389).
3. **Đồng bộ thời gian (Clock Skew):** Thời gian giữa pfSense và máy chủ Domain Controller (`DC01`) lệch nhau quá 5 phút, khiến gói tin xác thực bị từ chối.

### Hướng dẫn khắc phục
* **Bước 1:** Kiểm tra lại kết nối mạng từ pfSense đến `DC01` thông qua công cụ `Diagnostics > Ping` hoặc `Diagnostics > Test Port` (kiểm tra port 389).
* **Bước 2:** Xác thực lại cấu hình Authentication Server trên pfSense:
  * **Server Address:** IP của `DC01` (`10.6.0.10`).
  * **Port:** `389` (nếu chưa cấu hình SSL/TLS).
  * **Bind credentials:** Nhập chính xác tên tài khoản dạng UPN (`svc_pfsense_ldap@lab6.test`) hoặc DN đầy đủ.
  * **Base DN:** `DC=lab6,DC=test`.
* **Bước 3:** Kiểm tra đồng bộ thời gian trên cả pfSense và `DC01`, đảm bảo cấu hình chung một máy chủ NTP hoặc khớp giờ thực tế.

---

## 2. Lỗi Captive Portal (Không hiện trang đăng nhập hoặc vòng lặp chuyển hướng)

### Biểu hiện
* Máy Client (Client-A, Client-B) khi kết nối mạng LAN và mở trình duyệt web không tự động bật trang đăng nhập Captive Portal.
* Hoặc bị kẹt ở trạng thái chuyển hướng liên tục (`Redirect loop`) khi bấm đăng nhập.

### Nguyên nhân phổ biến
1. **Sai Interface chọn cho Captive Portal Zone:** Zone được gán nhầm vào interface WAN hoặc MGMT thay vì giao diện LAN (`lab6-lan`).
2. **Vấn đề phân giải DNS:** Trình duyệt không thể phân giải tên miền trang đăng nhập do máy Client không trỏ DNS về `10.6.0.10` (`DC01`) hoặc pfSense.
3. **Bypassed MAC / IP chưa chuẩn:** Client nằm trong danh sách được bỏ qua Captive Portal (`Allowed IP/MAC`) hoặc bị chặn bởi cấu hình Firewall trước khi chạm tới Captive Portal.

### Hướng dẫn khắc phục
* **Bước 1:** Kiểm tra cấu hình Captive Portal tại `Services > Captive Portal`, đảm bảo chọn đúng Zone `LAB6_USERS` và giao diện `LAN`.
* **Bước 2:** Kiểm tra cài đặt IP/DNS trên máy Client: Lệnh `ipconfig /all` phải đảm bảo DNS Server trỏ đúng về `10.6.0.10`.
* **Bước 3:** Thử truy cập trực tiếp bằng địa chỉ IP của cổng LAN pfSense kèm cổng Captive Portal (ví dụ: `http://10.6.0.1:8000`) để ép trình duyệt gọi trang đăng nhập.

---

## 3. Lỗi tường lửa: Quy tắc (Firewall Rules) không có hiệu lực

### Biểu hiện
* Đã cấu hình rule chặn `CLIENT_A` truy cập `JOB_SITE` hoặc chặn `RESTRICTED_SITE`, nhưng máy client vẫn truy cập bình thường.

### Nguyên nhân phổ biến
1. **Thứ tự ưu tiên của Rule (Rule Evaluation Order):** pfSense đọc các quy tắc tường lửa **từ trên xuống dưới**. Nếu phía trên có một rule dạng `Pass` (cho phép tất cả) nằm trước rule `Block`, gói tin sẽ được cho phép trước khi kịp đối chiếu với rule chặn.
2. **Sai thông tin trong Aliases:** Địa chỉ IP khai báo trong Aliases không khớp với IP thực tế của máy Client hoặc cụm Web server ở DMZ.
3. **State Table caching:** Kết nối mạng cũ đã được lưu trong bảng trạng thái (`State Table`) của tường lửa, khiến pfSense vẫn cho phép luồng dữ liệu cũ tiếp tục chạy dù rule mới đã được áp dụng.

### Hướng dẫn khắc phục
* **Bước 1:** Di chuyển các quy tắc chặn (`Block` / `Reject`) lên **vị trí phía trên cùng** của bảng rule trên giao diện LAN.
* **Bước 2:** Xóa sạch các state cũ liên quan đến kết nối đang kiểm tra bằng cách vào `Diagnostics > States > Reset States` hoặc lọc theo IP và chọn Kill state.
* **Bước 3:** Kiểm tra lại log tường lửa tại `Status > System Logs > Firewall` để xem gói tin đang khớp với rule nào (`Matched rule`).

---

## 4. Lỗi tính năng Schedules (Quy tắc theo thời gian không chạy đúng giờ)

### Biểu hiện
* Cấu hình chặn quyền truy cập `SOCIAL_SITE` trong giờ làm việc hoặc chỉ cho phép giờ nghỉ trưa, nhưng thời gian chặn/mở không đúng với thực tế cài đặt.

### Nguyên nhân phổ biến
1. **Múi giờ (Timezone) trên pfSense chưa chuẩn:** pfSense chưa cấu hình đúng múi giờ địa phương (ví dụ: `Asia/Ho_Chi_Minh`), dẫn đến lệch giờ so với Schedules được định nghĩa.
2. **Sai cú pháp khoảng thời gian hoặc ngày trong tuần** khi thiết lập Schedule.

### Hướng dẫn khắc phục
* **Bước 1:** Kiểm tra và cấu hình lại múi giờ tại `System > General Setup > Timezone` (chọn `Asia/Ho_Chi_Minh` hoặc múi giờ `GMT+7`).
* **Bước 2:** Đảm bảo pfSense đã kết nối Internet để đồng bộ thời gian chuẩn qua NTP Server.
* **Bước 3:** Kiểm tra lại dải giờ cấu hình trong `Firewall > Schedules`, lưu ý cấu hình đúng các ngày trong tuần (`Days of the week`) và khung giờ (`Start/End time`).

---

## 5. Lỗi kết nối giữa mạng LAN và vùng DMZ (`DMZ-WEB`)

### Biểu hiện
* Không thể ping hoặc không truy cập được các dịch vụ web ảo hóa trên `DMZ-WEB` từ các máy Client trong mạng LAN.

### Nguyên nhân phổ biến
1. Thiếu quy tắc Firewall cho phép lưu lượng từ giao diện LAN sang giao diện DMZ.
2. Máy chủ `DMZ-WEB` chưa cấu hình đúng các IP phụ hoặc dịch vụ Nginx chưa lắng nghe trên các địa chỉ IP tương ứng (`172.16.6.10`, `.20`, `.30`, `.40`).

### Hướng dẫn khắc phục
* **Bước 1:** Kiểm tra rule trên tab `LAN` và tab `DMZ` trong `Firewall > Rules`, đảm bảo có rule cho phép LAN đi sang DMZ theo các cổng dịch vụ yêu cầu (`WEB_PORTS`).
* **Bước 2:** Trên `DMZ-WEB`, kiểm tra lệnh `ip a` để xác nhận các IP phụ đã được gán đầy đủ và kiểm tra dịch vụ Nginx đang chạy (`sudo systemctl status nginx`).
