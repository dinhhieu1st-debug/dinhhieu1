# CẤU HÌNH MÔI TRƯỜNG BIÊN DỊCH CHÉO QT CHO RASPBERRY PI

## 1. Máy Host

```text
Hệ điều hành: Ubuntu 22.04 x86_64
Kiến trúc Host: x86_64
CMake: 3.22.1
Build system: Ninja
```

## 2. Raspberry Pi

```text
Thiết bị: Raspberry Pi 4
Hệ điều hành: Debian / Raspberry Pi OS 12 Bookworm 64-bit
Kiến trúc: aarch64
glibc: 2.36
User: pi
IP hiện tại: 192.168.137.49
```

## 3. Bộ biên dịch chéo

```text
Target: aarch64-linux-gnu
Binutils: 2.40
GCC: 12.2.0
G++: 12.2.0
```

Thư mục cài đặt:

```text
/opt/cross-pi-gcc-bookworm
```

Compiler C:

```text
/opt/cross-pi-gcc-bookworm/bin/aarch64-linux-gnu-gcc
```

Compiler C++:

```text
/opt/cross-pi-gcc-bookworm/bin/aarch64-linux-gnu-g++
```

## 4. Sysroot Raspberry Pi

Sysroot được đồng bộ trực tiếp từ Raspberry Pi Bookworm:

```text
~/Qt6Cross_Bookworm_651/rpi-sysroot
```

Các thư mục chính:

```text
~/Qt6Cross_Bookworm_651/rpi-sysroot/usr/include
~/Qt6Cross_Bookworm_651/rpi-sysroot/usr/lib
~/Qt6Cross_Bookworm_651/rpi-sysroot/lib
~/Qt6Cross_Bookworm_651/rpi-sysroot/usr/share/pkgconfig
```

## 5. Qt

```text
Phiên bản Qt: 6.5.1
Qt Host: x86_64
Qt Target: ARM64 aarch64
```

Qt Host dùng để chạy công cụ build:

```text
~/Qt6Cross_Bookworm_651/qt6/host
```

Qt ARM64 staging dùng để liên kết chương trình:

```text
~/Qt6Cross_Bookworm_651/qt6/pi
```

Qt runtime trên Raspberry Pi:

```text
/usr/local/qt6
```

## 6. Toolchain CMake

File toolchain:

```text
~/Qt6Cross_Bookworm_651/qt6/pi-build/toolchain.cmake
```

Khi tích hợp vào dự án mới:

```bash
-DCMAKE_TOOLCHAIN_FILE=$HOME/Qt6Cross_Bookworm_651/qt6/pi-build/toolchain.cmake
```

Đường dẫn Qt Host:

```bash
-DQT_HOST_PATH=$HOME/Qt6Cross_Bookworm_651/qt6/host
```

Đường dẫn Qt ARM64:

```bash
-DQt6_DIR=$HOME/Qt6Cross_Bookworm_651/qt6/pi/lib/cmake/Qt6
```

Ví dụ các module Qt:

```bash
-DQt6Core_DIR=$HOME/Qt6Cross_Bookworm_651/qt6/pi/lib/cmake/Qt6Core
-DQt6Gui_DIR=$HOME/Qt6Cross_Bookworm_651/qt6/pi/lib/cmake/Qt6Gui
-DQt6Widgets_DIR=$HOME/Qt6Cross_Bookworm_651/qt6/pi/lib/cmake/Qt6Widgets
-DQt6DBus_DIR=$HOME/Qt6Cross_Bookworm_651/qt6/pi/lib/cmake/Qt6DBus
-DQt6Sql_DIR=$HOME/Qt6Cross_Bookworm_651/qt6/pi/lib/cmake/Qt6Sql
-DQt6Network_DIR=$HOME/Qt6Cross_Bookworm_651/qt6/pi/lib/cmake/Qt6Network
```

## 7. Thư viện C++ bổ sung trên Raspberry Pi

Do Qt 6.5.1 được build bằng GCC 12.2.0, Raspberry Pi cần thư viện `libstdc++.so.6` tương ứng.

Thư mục:

```text
/home/pi/gcc12-libs
```

Khi chạy chương trình:

```bash
export LD_LIBRARY_PATH=/home/pi/gcc12-libs:/usr/local/qt6/lib:$LD_LIBRARY_PATH
```

Không ghi đè thư viện `libstdc++.so.6` của hệ thống Raspberry Pi.

## 8. Biến môi trường chạy Qt trên Raspberry Pi

```bash
export DISPLAY=:0
export XAUTHORITY=/home/pi/.Xauthority
export XDG_RUNTIME_DIR=/run/user/$(id -u)
export LD_LIBRARY_PATH=/home/pi/gcc12-libs:/usr/local/qt6/lib:$LD_LIBRARY_PATH
export QT_PLUGIN_PATH=/usr/local/qt6/plugins
export QT_QPA_PLATFORM=xcb
```

## 9. Kết quả đầu ra yêu cầu

Chương trình được build trên Host phải có kiến trúc:

```text
ELF 64-bit LSB executable, ARM aarch64
```

Kiểm tra bằng:

```bash
file ten_chuong_trinh
```

## 10. Luồng biên dịch chéo

```text
Ubuntu 22.04 x86_64
    ↓
Binutils 2.40
    ↓
GCC/G++ 12.2.0 aarch64-linux-gnu
    ↓
Sysroot Raspberry Pi Bookworm glibc 2.36
    ↓
Qt 6.5.1 ARM64
    ↓
Tạo chương trình ELF ARM aarch64
    ↓
SCP sang Raspberry Pi
    ↓
Raspberry Pi chạy bằng Qt tại /usr/local/qt6
```
Lệnh test chạy trên HOST
```text
cd ~/Qt6Cross_Bookworm_651/test_qt_arm64 && \
./build_and_run_pi.sh
