# BÁO CÁO CHẨN ĐOÁN MẠNG VÀ XỬ LÝ TRÙNG CỔNG
**Môn học:** Quản trị Hệ thống & Mạng / DevOps  
**Bài tập:** Session 01 - Exercise 05  
**Học viên thực hiện:** sonledev  

---

## 1. Tình huống 1: Xử lý trùng cổng 8080 (Address already in use)

### 1.1. Bối cảnh
Khi khởi chạy dịch vụ `user-service`, hệ thống thông báo lỗi `java.net.BindException: Address already in use: bind` do cổng **8080** đang bị một tiến trình khác trên hệ thống chiếm dụng.

### 1.2. Các bước thực hiện và câu lệnh

#### Bước 1: Tìm Process ID (PID) đang chiếm dụng cổng 8080
```bash
sudo ss -tulpn | grep :8080
```
* **Giải thích các tham số:**
  * `ss` (Socket Statistics): Công cụ truy vấn thông tin chi tiết về socket mạng nhanh và tối ưu hơn lệnh `netstat` cũ.
  * `-t` (TCP): Lọc các socket thuộc giao thức TCP.
  * `-u` (UDP): Lọc các socket thuộc giao thức UDP.
  * `-l` (listening): Chỉ hiển thị các socket đang mở ở trạng thái chờ kết nối (LISTEN).
  * `-p` (processes): Hiển thị tên tiến trình và PID (Process ID) sở hữu socket (cần quyền `sudo` để xem tiến trình hệ thống).
  * `-n` (numeric): Hiển thị địa chỉ IP và số hiệu cổng dưới dạng số trực tiếp, không qua bước phân giải DNS giúp lệnh phản hồi nhanh.
  * `| grep :8080`: Lọc riêng dòng có chứa cổng `:8080`.

* **Kết quả thực thi:**
```text
tcp   LISTEN 0      128          0.0.0.0:8080       0.0.0.0:*    users:(("user-service",pid=2845,fd=7))
```
-> Xác định được PID của tiến trình đang chiếm cổng là: **2845**.

#### Bước 2: Dừng tiến trình chiếm cổng để giải phóng
```bash
sudo kill -9 2845
```
* **Giải thích:**
  * `kill`: Lệnh gửi tín hiệu (signal) can thiệp đến tiến trình qua PID.
  * `-9` (tương đương tín hiệu `SIGKILL`): Buộc tiến trình dừng hoạt động ngay lập tức, kernel giải phóng toàn bộ tài nguyên và cổng mạng mà tiến trình đó đang giữ. Tiến trình không thể bỏ qua (catch/ignore) tín hiệu này.
  * `2845`: PID xác định được từ Bước 1.

#### Bước 3: Kiểm tra lại trạng thái cổng 8080
```bash
sudo ss -tulpn | grep :8080
```
* **Kết quả:** Lệnh không trả về bất kỳ dòng nào (output rỗng), chứng tỏ cổng 8080 đã được giải phóng thành công và dịch vụ `user-service` có thể khởi động lại bình thường.

---

## 2. Tình huống 2: Chẩn đoán kết nối tới Database PostgreSQL (10.0.1.20:5432)

### 2.1. Bối cảnh
Ứng dụng báo lỗi không thể kết nối tới cơ sở dữ liệu PostgreSQL tại địa chỉ IP `10.0.1.20` trên cổng mặc định `5432`.

### 2.2. Các bước thực hiện và câu lệnh

#### Bước 1: Kiểm tra đường truyền vật lý / kết nối mạng tầng IP (Layer 3)
```bash
ping -c 4 10.0.1.20
```
* **Giải thích:**
  * `ping`: Sử dụng giao thức ICMP (Echo Request / Echo Reply) để kiểm tra khả năng liên lạc mạng hai chiều giữa máy chủ ứng dụng và máy chủ database.
  * `-c 4` (count): Chỉ gửi đúng 4 gói tin ICMP rồi tự động kết thúc (tránh gửi liên tục vô hạn).
* **Đánh giá kết quả:**
  * Nếu nhận được `4 packets transmitted, 4 received, 0% packet loss`: Đường truyền vật lý và routing mạng hoạt động tốt.
  * Nếu nhận được `Destination Host Unreachable` hoặc `100% packet loss`: Sự cố nằm ở đường cáp/switch, cấu hình sai dải IP hoặc tường lửa/ACL chặn hoàn toàn giao thức ICMP.

#### Bước 2: Thăm dò trạng thái mở của cổng 5432 trên Database (Layer 4)
Có thể sử dụng một trong hai phương án sau:

##### Cách 1: Sử dụng Netcat (`nc`) - Khuyến nghị
```bash
nc -zv 10.0.1.20 5432
```
* **Giải thích:**
  * `nc` (Netcat): Công cụ kiểm tra và thao tác kết nối mạng TCP/UDP.
  * `-z` (Zero-I/O mode): Chỉ thực hiện bắt tay TCP 3 bước (3-way handshake) để xác định cổng mở hay đóng mà không gửi dữ liệu ứng dụng.
  * `-v` (Verbose): In thông tin chi tiết quá trình kết nối ra màn hình.
  * `5432`: Cổng mặc định của hệ quản trị cơ sở dữ liệu PostgreSQL.
* **Đánh giá kết quả:**
  * `Connection to 10.0.1.20 5432 port [tcp/postgresql] succeeded!`: Cổng mở, dịch vụ PostgreSQL đang chạy và cho phép kết nối.
  * `Connection refused`: Máy chủ đích đang bật, nhưng dịch vụ PostgreSQL bị tắt hoặc cấu hình `listen_addresses` chưa cho phép IP bên ngoài truy cập.
  * `Connection timed out`: Tường lửa (Firewall / Security Group / iptables) đang chặn cổng 5432.

##### Cách 2: Sử dụng `curl -v`
```bash
curl -v telnet://10.0.1.20:5432
```
* **Giải thích:**
  * `curl -v`: Hiển thị chi tiết quá trình kết nối socket TCP (Verbose).
  * Giao thức `telnet://`: Thử nghiệm bắt tay kết nối TCP trực tiếp vào cổng đích 5432 để kiểm tra phản hồi.
