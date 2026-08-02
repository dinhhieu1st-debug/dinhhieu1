# KIẾN TRÚC VÀ LUỒNG HOẠT ĐỘNG HỆ THỐNG KIỂM SOÁT RA VÀO

## 1. Tổng quan hệ thống

Hệ thống kiểm soát ra vào gồm 3 khối chính:

```text
Máy Host Ubuntu
        │
        │ Biên dịch chéo + triển khai chương trình
        ▼
Raspberry Pi
        │
        │ MQTT qua mạng Wi-Fi/LAN
        ▼
ESP32 + RFID + cảm biến + servo + còi + OLED
```

Kiến trúc đang sử dụng:

```text
Host Ubuntu
- Viết và sửa mã nguồn Qt
- Thiết kế giao diện
- Biên dịch chéo chương trình ARM64
- Gửi chương trình sang Raspberry Pi
- Không trực tiếp vận hành cửa

Raspberry Pi
- Chạy giao diện Qt
- Chạy Mosquitto MQTT Broker
- Xác thực thẻ RFID
- Lưu dữ liệu SQLite
- Gửi lệnh điều khiển cho ESP32

ESP32
- Đọc thiết bị phần cứng
- Gửi UID và trạng thái cảm biến
- Nhận lệnh mở/đóng cửa
- Điều khiển servo, còi và OLED
```

---

## 2. Các chức năng của hệ thống

### 2.1. Chức năng người dùng

- Đăng nhập hệ thống.
- Phân quyền quản trị viên và nhân viên.
- Quản lý tài khoản.
- Thêm, sửa, xóa và tìm kiếm thẻ RFID.
- Xem danh sách thẻ.
- Xem lịch sử ra vào.
- Tìm kiếm và lọc lịch sử.
- Xuất lịch sử ra file CSV.
- Mở cửa thủ công.
- Đóng cửa thủ công.
- Cài đặt địa chỉ MQTT.
- Cài đặt cổng MQTT.
- Cài đặt thời gian tự đóng cửa.

### 2.2. Chức năng kiểm soát cửa

- Phát hiện người đứng trước cửa.
- Bật đầu đọc RFID khi có người.
- Đọc UID thẻ.
- Kiểm tra thẻ hợp lệ hoặc không hợp lệ.
- Với thẻ hợp lệ:
  - Còi kêu 1 tiếng.
  - Servo mở cửa.
  - Lưu lịch sử cho phép.
- Với thẻ không hợp lệ:
  - Còi kêu 3 tiếng.
  - Cửa không mở.
  - Lưu lịch sử từ chối.
- Tự động đóng cửa khi người đã đi qua.
- Tự động đóng cửa khi hết thời gian chờ.
- Cho phép quét lại ngay sau khi quét thẻ sai.

### 2.3. Chức năng hệ thống

- MQTT có tài khoản và mật khẩu.
- MQTT có ACL phân quyền topic.
- Qt tự kết nối MQTT.
- Đồng bộ thời gian đóng cửa từ Qt sang ESP32.
- SQLite lưu dữ liệu trực tiếp trên Raspberry Pi.
- Chặn mở nhiều chương trình Qt cùng lúc.
- Host có script tự build và triển khai sang Pi.

---

## 3. Vai trò của máy Host Ubuntu

Máy Host là **máy phát triển phần mềm**, không phải máy vận hành cửa.

### Host thực hiện

- Viết code Qt/C++.
- Thiết kế giao diện Qt.
- Sửa lỗi chương trình.
- Cấu hình CMake.
- Biên dịch chéo từ kiến trúc `x86_64` sang `ARM aarch64`.
- Tạo file chạy dành cho Raspberry Pi.
- Chép file sang Raspberry Pi bằng SCP.
- Dừng chương trình cũ trên Pi.
- Cập nhật chương trình mới.
- Khởi chạy lại chương trình trên Pi.

### Quy trình triển khai từ Host

```text
Sửa code
   ↓
CMake cấu hình project
   ↓
Cross-compile bằng GCC ARM64
   ↓
Tạo file Hethongkiemsoatravao ARM aarch64
   ↓
SCP sang Raspberry Pi
   ↓
Chạy chương trình trên Pi
```

Host không trực tiếp:

- Đọc RFID.
- Điều khiển servo.
- Lưu dữ liệu vận hành chính.
- Chạy giao diện cuối cùng của hệ thống.

---

## 4. Vai trò của Raspberry Pi

Raspberry Pi là **bộ xử lý trung tâm và máy vận hành hệ thống**.

### Raspberry Pi chạy giao diện Qt

Giao diện Qt dùng để:

- Đăng nhập.
- Quản lý người dùng.
- Quản lý thẻ RFID.
- Hiển thị UID.
- Xác thực thẻ.
- Điều khiển cửa.
- Xem lịch sử.
- Cài đặt hệ thống.

### Raspberry Pi chạy MQTT Broker

Mosquitto trên Pi làm trung gian liên lạc giữa Qt và ESP32:

```text
ESP32 → Mosquitto → Qt
Qt → Mosquitto → ESP32
```

### Raspberry Pi lưu cơ sở dữ liệu

File dữ liệu chính:

```text
/home/pi/he-thong-kiem-soat-ra-vao/data/hethongkiemsoat.sqlite
```

Các bảng chính:

```text
tai_khoan
the_rfid
lich_su_ra_vao
```

Pi lưu:

- Tài khoản đăng nhập.
- Quyền người dùng.
- UID thẻ RFID.
- Họ tên chủ thẻ.
- Mã số.
- Thời gian quét.
- Kết quả cho phép hoặc từ chối.
- Trạng thái cửa.

### Raspberry Pi quyết định thẻ hợp lệ

ESP32 chỉ đọc UID. Raspberry Pi mới là thiết bị kiểm tra UID đó có trong cơ sở dữ liệu hay không.

```text
ESP32 đọc UID
        ↓
Pi tìm UID trong SQLite
        ↓
Có thẻ → GRANTED
Không có → DENIED
```

---

## 5. Vai trò của ESP32

ESP32 là **bộ điều khiển phần cứng tại cửa**.

### ESP32 quản lý

- RFID RC522.
- Cảm biến vật cản hoặc cảm biến người.
- Servo cửa.
- Buzzer.
- OLED.
- Wi-Fi.
- MQTT Client.

### ESP32 thực hiện

- Kết nối Wi-Fi.
- Kết nối Mosquitto trên Raspberry Pi.
- Đọc cảm biến.
- Đánh thức RFID và OLED khi có người.
- Đọc UID thẻ.
- Gửi UID lên MQTT.
- Nhận kết quả `GRANTED` hoặc `DENIED`.
- Mở hoặc đóng servo.
- Điều khiển số lần còi kêu.
- Hiển thị trạng thái trên OLED.
- Gửi trạng thái cửa về Pi.
- Nhận thời gian tự đóng cửa từ Qt.

ESP32 không lưu danh sách thẻ hợp lệ. Vì vậy thay đổi danh sách thẻ chỉ cần sửa trong Qt/SQLite, không cần nạp lại firmware ESP32.

---

## 6. Các thiết bị kết nối với nhau như thế nào

### 6.1. Host Ubuntu kết nối với Raspberry Pi

Dùng SSH và SCP qua mạng:

```text
Host Ubuntu
   │
   ├── SSH: chạy lệnh trên Pi
   │
   └── SCP: chép chương trình sang Pi
```

Host và Pi phải cùng mạng và Pi phải bật SSH.

### 6.2. Raspberry Pi kết nối với ESP32

Dùng MQTT qua Wi-Fi.

ESP32 kết nối tới:

```text
IP Raspberry Pi: 192.168.137.227
Port MQTT: 1883
```

Qt chạy ngay trên Pi nên kết nối Broker bằng:

```text
127.0.0.1:1883
```

Luồng mạng:

```text
ESP32 ──Wi-Fi──> Mosquitto trên Pi
Qt trên Pi ────> Mosquitto trên Pi
```

---

## 7. Các topic MQTT đang sử dụng

### 7.1. ESP32 gửi lên Pi

#### `access/rfid`

Gửi UID thẻ:

```text
D72D6303
```

#### `access/sensor`

Gửi trạng thái cảm biến:

```text
OBSTACLE
CLEAR
```

#### `access/status`

Gửi trạng thái ESP32:

```text
ONLINE
OFFLINE
```

#### `access/door/status`

Gửi trạng thái cửa:

```text
OPENED
CLOSED
```

### 7.2. Qt gửi xuống ESP32

#### `access/door/command`

Lệnh cửa:

```text
OPEN
CLOSE
```

#### `access/result`

Kết quả kiểm tra thẻ:

```text
GRANTED
DENIED
```

#### `access/config/door_timeout`

Thời gian tự đóng cửa:

```text
15
```

---

## 8. Luồng xử lý khi quét thẻ hợp lệ

```text
1. Cảm biến phát hiện có người
        ↓
2. ESP32 bật RFID và OLED
        ↓
3. Người dùng quét thẻ
        ↓
4. RC522 đọc UID
        ↓
5. ESP32 gửi UID lên access/rfid
        ↓
6. Mosquitto trên Pi nhận bản tin
        ↓
7. Qt nhận UID
        ↓
8. Qt tìm UID trong SQLite
        ↓
9. UID tồn tại
        ↓
10. Qt lưu lịch sử CHO PHEP
        ↓
11. Qt gửi GRANTED qua access/result
        ↓
12. Qt gửi OPEN qua access/door/command
        ↓
13. ESP32 nhận lệnh
        ↓
14. Còi kêu 1 tiếng
        ↓
15. Servo mở cửa
        ↓
16. ESP32 gửi OPENED về Pi
        ↓
17. Người đi qua cảm biến
        ↓
18. Khi người rời khỏi cửa, ESP32 đóng servo
        ↓
19. ESP32 gửi CLOSED về Pi
```

---

## 9. Luồng xử lý khi quét thẻ không hợp lệ

```text
1. ESP32 đọc UID
        ↓
2. Gửi UID lên Raspberry Pi
        ↓
3. Qt tìm trong SQLite
        ↓
4. Không tìm thấy UID
        ↓
5. Qt lưu lịch sử TU CHOI
        ↓
6. Qt gửi DENIED
        ↓
7. ESP32 kêu 3 tiếng
        ↓
8. Servo không mở
        ↓
9. ESP32 cho phép quét thẻ khác ngay
```

---

## 10. Luồng mở cửa thủ công

```text
Quản trị viên nhấn MO CUA trên Qt
        ↓
Qt gửi OPEN qua MQTT
        ↓
ESP32 nhận OPEN
        ↓
Servo mở cửa
        ↓
ESP32 gửi OPENED
        ↓
Qt cập nhật trạng thái cửa
```

Nếu không có người đi qua:

```text
Hết 15 giây
        ↓
ESP32 tự đóng cửa
```

Nếu đã có người đi qua:

```text
Cảm biến phát hiện người
        ↓
Người rời khỏi vùng cảm biến
        ↓
Chờ xác nhận khoảng 3 giây
        ↓
ESP32 đóng cửa
```

---

## 11. Quan hệ giữa các thiết bị

### Host Ubuntu là nơi phát triển

```text
Host không điều khiển cửa trực tiếp.
Host chỉ tạo và cập nhật phần mềm.
```

### Raspberry Pi là trung tâm quyết định

```text
Pi quản lý giao diện, dữ liệu và logic xác thực.
```

### ESP32 là thiết bị chấp hành

```text
ESP32 đọc phần cứng và thực hiện lệnh từ Pi.
```

Có thể mô tả ngắn gọn:

```text
Host = nơi phát triển
Pi = bộ não trung tâm
ESP32 = bộ điều khiển phần cứng
Mosquitto = cầu nối truyền thông
SQLite = nơi lưu dữ liệu
Qt = giao diện và logic quản lý
```

---

## 12. Kiến trúc hoàn chỉnh

```text
┌──────────────────────────────┐
│       HOST UBUNTU x86_64     │
│                              │
│  - VS Code / Qt Creator      │
│  - CMake                     │
│  - Cross compiler ARM64      │
│  - Build và deploy           │
└──────────────┬───────────────┘
               │ SSH / SCP
               ▼
┌──────────────────────────────┐
│       RASPBERRY PI 4         │
│                              │
│  - Chương trình Qt ARM64     │
│  - Mosquitto MQTT            │
│  - SQLite                    │
│  - Xác thực RFID             │
│  - Quản lý người dùng        │
│  - Quản lý lịch sử           │
└──────────────┬───────────────┘
               │ MQTT qua Wi-Fi
               ▼
┌──────────────────────────────┐
│            ESP32             │
│                              │
│  - RFID RC522                │
│  - Cảm biến vật cản          │
│  - Servo                     │
│  - Buzzer                    │
│  - OLED                      │
│  - Điều khiển cửa            │
└──────────────────────────────┘
```

---

## 13. Kết luận

Mô hình triển khai hiện tại đúng theo quy trình:

- **Host Ubuntu** dùng để phát triển, biên dịch chéo và triển khai.
- **Raspberry Pi** chạy ứng dụng Qt, Mosquitto và SQLite.
- **ESP32** đọc cảm biến và điều khiển phần cứng.
- **MQTT** là kênh truyền dữ liệu giữa Raspberry Pi và ESP32.
- **SSH/SCP** là kênh triển khai chương trình từ Host sang Raspberry Pi.
