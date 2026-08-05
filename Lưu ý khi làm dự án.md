# Ghi chú dùng chung Host và Raspberry Pi cho nhiều dự án Qt + ESP32 + MQTT

## 1. Phân chia vai trò

```text
HOST Ubuntu x86_64
├── Viết code Qt
├── Viết code ESP32
├── Chạy thử giao diện Host
├── Cross-compile ARM64
├── Đóng gói
└── Deploy qua SSH/SCP

RASPBERRY PI ARM64
├── Chạy ứng dụng Qt
├── Chạy Mosquitto MQTT
├── Lưu SQLite
└── Hiển thị giao diện

ESP32
├── Đọc cảm biến
├── Điều khiển relay
├── Gửi dữ liệu MQTT
└── Nhận lệnh MQTT
```

Host chỉ phát triển và build. Không nên build Qt trực tiếp trên Pi nếu đã có bộ cross-compile.

---

## 2. Mỗi dự án phải có thư mục riêng

### Trên Host

```text
/home/pi/Du/ten-du-an-host/
├── qt/
├── build-host/
├── build-arm64/
├── package/
└── scripts/
```

### Trên Raspberry Pi

```text
/home/pi/Du/ten-du-an/
├── bin/
├── config/
├── data/
├── logs/
└── scripts/
```

Không chép file của dự án mới đè lên dự án cũ.

Ví dụ:

```text
/home/pi/Du/tram-thoi-tiet/
/home/pi/Du/rfid/
/home/pi/Du/du-an-moi/
```

---

## 3. Không ghi cứng đường dẫn Host theo tên người dùng

Nên dùng:

```bash
BASE="$HOME/Du/ten-du-an-host"
```

Không nên dùng:

```bash
BASE="/home/pi/Du/ten-du-an-host"
```

Vì khi chuyển sang máy Host khác có username `nampe`, `$HOME` vẫn tự đổi đúng.

---

## 4. Bộ Qt và cross-compiler hiện tại

```text
Qt Host x86_64: $HOME/Qt6Cross/qt6/host
Qt ARM64:       $HOME/Qt6Cross/qt6/pi
Sysroot Pi:     $HOME/Qt6Cross/rpi-sysroot
Toolchain:      $HOME/Qt6Cross/qt6/pi-build/toolchain.cmake
Cross GCC:      /opt/cross-pi-gcc/bin/aarch64-linux-gnu-gcc
Cross G++:      /opt/cross-pi-gcc/bin/aarch64-linux-gnu-g++
```

Phiên bản đang dùng:

```text
GCC/G++ 12.2.0
Binutils 2.40
Qt 6.5.1
```

---

## 5. Phải tách build Host và ARM64

### Build Host

```bash
"$HOME/Du/ten-du-an-host/scripts/build_host.sh"
```

Kết quả phải là `ELF x86-64`.

### Build Raspberry Pi

```bash
"$HOME/Du/ten-du-an-host/scripts/build_arm64.sh"
```

Kết quả phải là `ELF ARM aarch64`.

Kiểm tra:

```bash
file "$HOME/Du/ten-du-an-host/build-arm64/ten_chuong_trinh"
```

Không được chép nhầm binary x86-64 sang Raspberry Pi.

---

## 6. Tránh trộn hai phiên bản Qt ARM64

Đã từng gặp lỗi do trộn Qt 6.5.1 tự build với Qt 6.4.2 trong sysroot, gây lỗi linker liên quan `libQt6DBus.so.6`.

Cách xử lý:

```cmake
find_package(Qt6 REQUIRED COMPONENTS
    Widgets
    Sql
    Network
    DBus
)

target_link_libraries(ten_app PRIVATE
    Qt6::Widgets
    Qt6::Sql
    Qt6::Network
    Qt6::DBus
)
```

Các thư viện Qt phải ưu tiên lấy từ:

```text
$HOME/Qt6Cross/qt6/pi/lib
```

Không để linker tự lấy Qt cũ trong sysroot.

---

## 7. Mỗi dự án MQTT phải dùng bộ topic riêng

Ví dụ:

```text
RFID:            access/...
Trạm môi trường: tramthoitiet/...
Dự án nhà kính:  nhakinh/...
```

Không dùng topic quá chung như:

```text
data
status
control
```

Nên dùng:

```text
nhakinh/data
nhakinh/status
nhakinh/control
nhakinh/config
```

---

## 8. Mỗi ứng dụng nên có tài khoản MQTT riêng

Ví dụ:

```text
esp32_nhakinh
qt_nhakinh
```

ESP32 thường cần:

```text
write dữ liệu và trạng thái
read lệnh điều khiển và cấu hình
```

Qt thường cần:

```text
read dữ liệu và trạng thái
write lệnh điều khiển và cấu hình
```

Ví dụ ACL:

```text
user esp32_nhakinh
topic write nhakinh/data
topic write nhakinh/status
topic read nhakinh/control
topic read nhakinh/config

user qt_nhakinh
topic read nhakinh/data
topic read nhakinh/status
topic write nhakinh/control
topic write nhakinh/config
```

Không nên để Qt đăng nhập bằng tài khoản ESP32.

Sau khi sửa ACL:

```bash
sudo systemctl restart mosquitto
systemctl is-active mosquitto
```

---

## 9. Cấu hình MQTT phải tách khỏi source code

Nên lưu tại:

```text
/home/pi/Du/ten-du-an/config/cau_hinh.ini
```

Ví dụ:

```ini
[mqtt]
host=127.0.0.1
port=1883
username=qt_ten_du_an
password=mat_khau

[database]
path=data/ten_du_an.sqlite
```

ESP32 kết nối tới IP thật của Pi:

```cpp
#define MQTT_BROKER "192.168.137.227"
```

Qt chạy ngay trên Pi có thể dùng `127.0.0.1`.

---

## 10. Đường dẫn gốc của ứng dụng phải đúng

Binary nằm trong:

```text
/home/pi/Du/ten-du-an/bin/
```

Nếu dùng thẳng `QCoreApplication::applicationDirPath()` thì có thể tạo nhầm `bin/config`, `bin/data`, `bin/logs`.

Cần lùi lên một cấp khi thư mục hiện tại là `bin`:

```cpp
QDir thuMucBin(QCoreApplication::applicationDirPath());
QString thuMucGoc = thuMucBin.absolutePath();

if (thuMucBin.dirName() == "bin")
{
    thuMucBin.cdUp();
    thuMucGoc = thuMucBin.absolutePath();
}
```

Kết quả đúng:

```text
/home/pi/Du/ten-du-an/config
/home/pi/Du/ten-du-an/data
/home/pi/Du/ten-du-an/logs
```

---

## 11. MQTT callback và SQLite phải chạy đúng luồng Qt

`libmosquitto` chạy callback ở luồng riêng. SQLite của Qt thường chỉ dùng trong luồng đã tạo kết nối database.

Cần chuyển callback về luồng Qt chính:

```cpp
QMetaObject::invokeMethod(
    dichVu,
    [dichVu, topic, payload]()
    {
        dichVu->xuLyBanTin(topic, payload);
    },
    Qt::QueuedConnection
);
```

Nếu không làm, Qt có thể hiển thị dữ liệu nhưng SQLite không lưu bản ghi.

---

## 12. JSON ESP32 và Qt phải thống nhất tên trường

Ví dụ:

```json
{
  "nhiet_do": 29.5,
  "ap_suat": 997.4,
  "anh_sang": 376,
  "quat": 0,
  "den": 1,
  "che_do": "TU_DONG"
}
```

Không dùng lẫn `temperature` với `nhiet_do`, `pressure` với `ap_suat`, hoặc `fan` với `quat` trong cùng dự án.

---

## 13. Chuẩn hóa trạng thái và lệnh

Nên thống nhất:

```text
ONLINE / OFFLINE
ON / OFF
TU_DONG / THU_CONG
```

Relay hiện tại dùng active HIGH:

```cpp
#define RELAY_BAT HIGH
#define RELAY_TAT LOW
```

Phải kiểm tra loại relay trước khi dùng cho dự án khác vì có module active LOW.

---

## 14. Trạng thái MQTT nên dùng retained message

```text
topic: ten-du-an/status
ONLINE: retained
OFFLINE: Last Will retained
```

Ví dụ:

```cpp
mqttClient.connect(
    CLIENT_ID,
    MQTT_USERNAME,
    MQTT_PASSWORD,
    TOPIC_STATUS,
    1,
    true,
    "OFFLINE"
);

mqttClient.publish(TOPIC_STATUS, "ONLINE", true);
```

Dữ liệu cảm biến thời gian thực thường không cần retain.

---

## 15. Tránh lỗi gõ sai topic

Đã từng nhầm `tranthoitiet` với `tramthoitiet`.

Nên khai báo biến khi kiểm tra:

```bash
TOPIC="tramthoitiet/#"

mosquitto_sub \
-h 127.0.0.1 \
-u tram_qt \
-P 123 \
-t "$TOPIC" \
-v
```

---

## 16. Kiểm tra MQTT theo từng tầng

### Kiểm tra Mosquitto

```bash
systemctl is-active mosquitto
```

### Kiểm tra kết nối ESP32 tới Pi

```bash
sudo ss -tnp | grep ':1883'
```

### Theo dõi toàn bộ topic dự án

```bash
mosquitto_sub \
-h 127.0.0.1 \
-p 1883 \
-u ten_user \
-P mat_khau \
-t "ten_du_an/#" \
-v
```

### Kiểm tra ACL và lỗi xác thực

```bash
sudo journalctl -u mosquitto \
--since "5 minutes ago" \
--no-pager
```

Nếu ESP32 báo MQTT kết nối thành công nhưng Pi không thấy dữ liệu, kiểm tra:

```text
IP broker
topic
ACL
username/password
quyền publish
```

---

## 17. SQLite phải đặt riêng cho từng dự án

Ví dụ:

```text
/home/pi/Du/nhakinh/data/nhakinh.sqlite
```

Không dùng chung một database cho nhiều dự án nếu cấu trúc bảng khác nhau.

Nên có các bảng:

```text
du_lieu_cam_bien
lich_su_canh_bao
lich_su_dieu_khien
cau_hinh
```

Kiểm tra:

```bash
sqlite3 duong_dan_database ".tables"
sqlite3 duong_dan_database ".schema"
```

---

## 18. Deploy phải giữ lại config và database

Khi cập nhật phiên bản mới, chỉ nên thay `bin/` và `scripts/`.

Phải giữ lại:

```text
config/cau_hinh.ini
data/*.sqlite
```

Script deploy nên:

1. Sao lưu bản cũ.
2. Chép gói mới.
3. Giữ cấu hình cũ.
4. Giữ database cũ.
5. Cấp quyền chạy.
6. Kiểm tra kiến trúc ARM64.

---

## 19. Package có thể chứa binary cũ

Đã từng xảy ra:

```text
build-arm64 = giao diện mới
package = giao diện cũ
Pi = giao diện cũ
```

Sau mỗi lần build phải chép lại:

```bash
cp -f \
"$BASE/build-arm64/ten_app" \
"$BASE/package/ten-du-an/bin/ten_app"
```

Tạo lại file nén:

```bash
rm -f "$BASE/package/ten-du-an-arm64.tar.gz"

tar -C "$BASE/package" \
-czf "$BASE/package/ten-du-an-arm64.tar.gz" \
"ten-du-an"
```

Kiểm tra SHA256:

```bash
sha256sum \
"$BASE/build-arm64/ten_app" \
"$BASE/package/ten-du-an/bin/ten_app"
```

Hai mã phải giống hệt nhau.

---

## 20. Kiểm tra nội dung binary trước khi deploy

```bash
strings "$BASE/build-arm64/ten_app" \
| grep "CHU_MOI_TREN_GIAO_DIEN"
```

Nếu Host build có chữ mới nhưng package không có thì chưa copy binary mới vào package.

---

## 21. Chạy ứng dụng Qt trên màn hình Raspberry Pi

Pi đang dùng X11:

```text
DISPLAY=:0
QT_QPA_PLATFORM=xcb
```

Script chạy:

```bash
export QT_QPA_PLATFORM=xcb
export LD_LIBRARY_PATH="/usr/local/qt6/lib:${LD_LIBRARY_PATH:-}"
export QT_PLUGIN_PATH="/usr/local/qt6/plugins"
export QT_QPA_PLATFORM_PLUGIN_PATH="/usr/local/qt6/plugins/platforms"

DISPLAY=:0 /duong/dan/den/app
```

Nếu báo không tìm thấy plugin `wayland`, ép dùng `xcb`.

---

## 22. Không dùng `pkill -f` trong cùng lệnh SSH

Lệnh này có thể giết luôn phiên SSH:

```bash
pkill -f "/home/pi/Du/ten-du-an/bin/ten_app"
```

Nên dùng:

```bash
pkill -x ten_app
```

Lưu ý tên tiến trình Linux chỉ so khớp tối đa khoảng 15 ký tự. Với tên dài, dùng:

```bash
ps -ef | grep "[t]en_app"
```

hoặc lấy PID rồi kill riêng.

---

## 23. Khóa chống mở hai ứng dụng

Dùng `QLockFile`:

```cpp
QLockFile khoa(
    QDir::temp().filePath("ten_du_an.lock")
);
```

Khi ứng dụng bị dừng bất thường, có thể cần xóa khóa:

```bash
rm -f /tmp/ten_du_an.lock
```

Mỗi dự án phải có file lock riêng.

---

## 24. Ghi log khi chạy nền

```bash
nohup env \
DISPLAY=:0 \
QT_QPA_PLATFORM=xcb \
/home/pi/Du/ten-du-an/scripts/run_pi.sh \
> /home/pi/Du/ten-du-an/logs/app.log 2>&1 &
```

Kiểm tra:

```bash
cat /home/pi/Du/ten-du-an/logs/app.log
```

Mỗi dự án phải có file log riêng.

---

## 25. Không dùng chung tên tiến trình giữa các dự án

Nên đặt tên executable riêng:

```text
tram_moi_truong_qt
rfid_access_qt
nha_kinh_qt
```

Không đặt tất cả thành `app`, `main`, hoặc `qt_app`.

---

## 26. File `.ui` và source phải được đưa vào CMake

```cmake
qt_add_executable(ten_app
    src/main.cpp
    src/cua_so_chinh.cpp
    include/cua_so_chinh.h
    ui/cua_so_chinh.ui
    resources/resources.qrc
)
```

File `resources.qrc` không được để rỗng hoàn toàn. Tối thiểu:

```xml
<RCC>
    <qresource prefix="/">
    </qresource>
</RCC>
```

Nếu rỗng sẽ báo `Premature end of document`.

---

## 27. Giao diện chưa có ESP32 phải hiện trạng thái rõ ràng

Không nên tạo dữ liệu giả trong bản nộp chính thức.

```text
Nhiệt độ: --
Áp suất: --
Ánh sáng: --
ESP32: OFFLINE
```

Chỉ cập nhật biểu đồ khi nhận dữ liệu hợp lệ thật.

---

## 28. Biểu đồ không nhất thiết phải dùng Qt Charts

Có thể tự vẽ bằng:

```text
QPainter
QPainterPath
QLinearGradient
```

Ưu điểm:

```text
Không cần cài thêm Qt Charts
Build Host và ARM64 đơn giản
Ít phụ thuộc thư viện
Dễ tùy chỉnh giao diện
```

Nên giới hạn 36–100 điểm gần nhất, không giữ vô hạn trong RAM.

---

## 29. Quy trình chuẩn khi sửa code

```text
1. Sửa source trên Host
2. Build Host
3. Chạy thử Host
4. Build ARM64
5. Kiểm tra file ARM64
6. Copy vào package
7. Tạo lại tar.gz
8. Deploy sang Pi
9. Dừng bản cũ
10. Chạy bản mới
11. Kiểm tra log
12. Kiểm tra MQTT và SQLite
```

Không sửa source trực tiếp trên Pi vì sẽ làm Host và Pi lệch phiên bản.

---

## 30. Bộ lệnh mẫu cho dự án mới

### Mở Qt Creator

```bash
cd "$HOME/Du/ten-du-an-host/qt" && \
qtcreator CMakeLists.txt
```

### Build Host

```bash
"$HOME/Du/ten-du-an-host/scripts/build_host.sh"
```

### Chạy Host

```bash
"$HOME/Du/ten-du-an-host/scripts/run_host.sh"
```

### Build ARM64

```bash
"$HOME/Du/ten-du-an-host/scripts/build_arm64.sh"
```

### Kiểm tra binary

```bash
file "$HOME/Du/ten-du-an-host/build-arm64/ten_app"
```

### Deploy

```bash
"$HOME/Du/ten-du-an-host/scripts/deploy_pi.sh" \
192.168.137.227
```

### Chạy trên Pi

```bash
ssh pi@192.168.137.227 '
rm -f /tmp/ten_du_an.lock

nohup env \
DISPLAY=:0 \
QT_QPA_PLATFORM=xcb \
/home/pi/Du/ten-du-an/scripts/run_pi.sh \
> /home/pi/Du/ten-du-an/logs/app.log 2>&1 &
'
```

---

## Nguyên tắc quan trọng nhất

```text
Mỗi dự án:
- thư mục riêng
- executable riêng
- file lock riêng
- database riêng
- config riêng
- log riêng
- MQTT topic riêng
- MQTT user riêng
- ACL riêng
- script build/deploy riêng
```

Làm đúng các điểm này thì nhiều dự án có thể dùng chung một Host, một Raspberry Pi và một Mosquitto mà không xung đột.
