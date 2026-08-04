# CẤU HÌNH BIÊN DỊCH CHÉO QT CHO RASPBERRY PI

## 1. Phiên bản môi trường

### Máy Host

```text
Hệ điều hành: Ubuntu 22.04
Kiến trúc: x86_64
CMake: 3.22.1
Build system: Ninja
```

### Raspberry Pi

```text
Thiết bị: Raspberry Pi 4
Hệ điều hành: Debian / Raspberry Pi OS 12 Bookworm 64-bit
Kiến trúc: ARM64 aarch64
glibc: 2.36
IP Raspberry Pi: 192.168.137.49
User: pi
```

### Bộ biên dịch chéo

```text
Target: aarch64-linux-gnu
Binutils: 2.40
GCC: 12.2.0
G++: 12.2.0
```

### Qt

```text
Qt: 6.5.1
Qt Host: x86_64
Qt Target: ARM64 aarch64
```

### Thư viện C++ trên Raspberry Pi

```text
libstdc++ từ GCC 12.2.0
Dùng để đáp ứng các phiên bản:
GLIBCXX_3.4.29
GLIBCXX_3.4.30
CXXABI_1.3.13
```

---

## 2. Luồng xử lý biên dịch chéo

```text
Máy Host Ubuntu x86_64
        ↓
Viết và sửa mã nguồn Qt/C++
        ↓
CMake cấu hình dự án
        ↓
Ninja thực hiện build
        ↓
GCC/G++ 12.2.0 biên dịch chéo
        ↓
Sử dụng sysroot lấy từ Raspberry Pi Bookworm
        ↓
Liên kết với Qt 6.5.1 ARM64
        ↓
Tạo chương trình ELF ARM aarch64
        ↓
Host dừng chương trình cũ trên Raspberry Pi
        ↓
Host chép chương trình mới sang Raspberry Pi bằng SCP
        ↓
Host gửi lệnh chạy sang Raspberry Pi bằng SSH
        ↓
Raspberry Pi nạp Qt 6.5.1 và thư viện GCC 12
        ↓
Chương trình chạy trên Raspberry Pi
        ↓
Giao diện hiển thị trên màn hình Raspberry Pi
```

---

## 3. Vai trò của từng thiết bị

### Host Ubuntu

```text
- Viết mã nguồn.
- Thiết kế giao diện Qt.
- Cấu hình CMake.
- Biên dịch chéo từ x86_64 sang ARM64.
- Tạo chương trình dành cho Raspberry Pi.
- Chép chương trình sang Raspberry Pi.
- Gửi lệnh chạy chương trình.
```

### Raspberry Pi

```text
- Nhận chương trình ARM64 từ Host.
- Chạy Qt 6.5.1.
- Hiển thị giao diện.
- Chạy logic trung tâm của dự án.
- Có thể chạy Mosquitto MQTT và SQLite.
- Giao tiếp với ESP32 và các thiết bị IoT.
```

### ESP32 trong dự án IoT

```text
- Đọc cảm biến và RFID.
- Gửi dữ liệu lên Raspberry Pi qua MQTT.
- Nhận lệnh điều khiển từ Raspberry Pi.
- Điều khiển servo, còi, OLED và thiết bị chấp hành.
```

---

## 4. Luồng hệ thống IoT hoàn chỉnh

```text
Host Ubuntu
- Viết code
- Biên dịch chéo
- Deploy chương trình
        ↓
Raspberry Pi
- Chạy giao diện Qt
- Xử lý logic
- Chạy MQTT
- Lưu SQLite
        ↓
ESP32
- Đọc cảm biến
- Đọc RFID
- Điều khiển phần cứng
```

Mô tả ngắn:

```text
Host = nơi phát triển và biên dịch
Raspberry Pi = bộ xử lý trung tâm
ESP32 = bộ điều khiển phần cứng
MQTT = kênh truyền dữ liệu
Qt = giao diện và logic quản lý
SQLite = nơi lưu dữ liệu
```

---

## 5. Lệnh test biên dịch chéo chạy trên Host

Chạy trên máy Host Ubuntu:

```bash
cd ~/Qt6Cross_Bookworm_651/test_qt_arm64 && ./build_and_run_pi.sh
```

Lệnh này sẽ tự động:

```text
Build chương trình Qt trên Host
→ tạo chương trình ARM64
→ dừng chương trình cũ trên Raspberry Pi
→ chép chương trình mới sang Raspberry Pi
→ gửi lệnh chạy qua SSH
→ Raspberry Pi chạy chương trình
→ giao diện Qt hiển thị trên màn hình Raspberry Pi
```

Kết quả cuối cùng cần đạt:

```text
Qt 6.5.1 cross-compile ARM64 OK
```
