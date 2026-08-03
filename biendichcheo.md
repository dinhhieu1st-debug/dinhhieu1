# Qt 6.5.1 Cross-Compile cho Raspberry Pi 4 Bookworm

Tài liệu này ghi lại **quy trình cuối cùng đã chạy thành công** để:

- Build bộ cross-compiler GCC/G++ 12.2.0 trên Ubuntu x86_64.
- Lấy sysroot trực tiếp từ Raspberry Pi 4 chạy Raspberry Pi OS Bookworm 64-bit.
- Build Qt 6.5.1 cho máy Host.
- Cross-compile Qt 6.5.1 ARM64 cho Raspberry Pi.
- Build ứng dụng Qt trên Host, chép sang Pi và chạy giao diện trực tiếp trên màn hình Pi.

> Cấu hình đã kiểm thử:
>
> - Host: Ubuntu 22.04 x86_64
> - Target: Raspberry Pi 4, Debian/Raspberry Pi OS 12 Bookworm 64-bit
> - Target architecture: `aarch64`
> - glibc target: `2.36`
> - Binutils: `2.40`
> - GCC/G++: `12.2.0`
> - Qt: `6.5.1`
> - IP Raspberry Pi trong quá trình cài: `192.168.137.227`
> - User Raspberry Pi: `pi`

---

## 1. Kiến trúc hệ thống

```text
HOST Ubuntu x86_64
├── Viết mã nguồn Qt/C++
├── Build Qt Host
├── Cross-compile Qt ARM64
├── Build chương trình ARM64
└── SCP/SSH sang Raspberry Pi
              │
              ▼
Raspberry Pi 4 Bookworm aarch64
├── Chạy ứng dụng Qt ARM64
├── Chạy thư viện Qt tại /usr/local/qt6
├── Hiển thị giao diện trực tiếp trên màn hình Pi
└── Có thể chạy MQTT, SQLite, GPIO và các phần cứng khác
```

---

# PHẦN A — CHUẨN BỊ HOST UBUNTU

## 2. Cập nhật Host

Chạy trên **HOST Ubuntu**:

```bash
sudo apt update
sudo apt upgrade -y
```

Tắt tự động sleep:

```bash
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
gsettings set org.gnome.desktop.session idle-delay 0
```

Kiểm tra:

```bash
gsettings get org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type
gsettings get org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type
gsettings get org.gnome.desktop.session idle-delay
```

Kết quả mong đợi:

```text
'nothing'
'nothing'
uint32 0
```

## 3. Cài các gói cần thiết trên Host

```bash
cd ~ && \
sudo apt-get install -y \
make \
build-essential \
libclang-dev \
ninja-build \
gcc \
git \
bison \
python3 \
gperf \
pkg-config \
libfontconfig1-dev \
libfreetype6-dev \
libx11-dev \
libx11-xcb-dev \
libxext-dev \
libxfixes-dev \
libxi-dev \
libxrender-dev \
libxcb1-dev \
libxcb-glx0-dev \
libxcb-keysyms1-dev \
libxcb-image0-dev \
libxcb-shm0-dev \
libxcb-icccm4-dev \
libxcb-sync-dev \
libxcb-xfixes0-dev \
libxcb-shape0-dev \
libxcb-randr0-dev \
libxcb-render-util0-dev \
libxcb-util-dev \
libxcb-xinerama0-dev \
libxcb-xkb-dev \
libxkbcommon-dev \
libxkbcommon-x11-dev \
libatspi2.0-dev \
libgl1-mesa-dev \
libglu1-mesa-dev \
freeglut3-dev \
gawk \
texinfo \
file \
wget \
rsync \
flex \
libssl-dev \
gdbserver \
gdb-multiarch \
libxcb-cursor-dev
```

Nếu apt lỗi cache hoặc file tải dở:

```bash
cd ~ && \
sudo apt clean && \
sudo rm -rf /var/lib/apt/lists/* && \
sudo apt update
```

## 4. Cài CMake

```bash
cd ~ && \
sudo apt install -y cmake
```

Kiểm tra:

```bash
cmake --version
```

Phiên bản đã dùng thành công:

```text
cmake version 3.22.1
```

## 5. Tạo thư mục riêng cho bộ Bookworm

```bash
cd ~ && \
mkdir -p ~/Qt6Cross_Bookworm_651 && \
cd ~/Qt6Cross_Bookworm_651
```

Tạo cấu trúc thư mục:

```bash
cd ~/Qt6Cross_Bookworm_651 && \
mkdir -p \
gcc_all \
rpi-sysroot/usr/include \
qt6/host \
qt6/pi \
qt6/host-build \
qt6/pi-build \
qt6/src
```

---

# PHẦN B — BUILD CROSS-COMPILER GCC 12.2.0

## 6. Tải mã nguồn Binutils, GCC và glibc

Chạy trên **HOST Ubuntu**:

```bash
cd ~/Qt6Cross_Bookworm_651/gcc_all && \
wget https://ftp.gnu.org/gnu/binutils/binutils-2.40.tar.xz && \
wget https://ftp.gnu.org/gnu/gcc/gcc-12.2.0/gcc-12.2.0.tar.xz && \
wget https://ftp.gnu.org/gnu/glibc/glibc-2.36.tar.xz
```

Nếu một file tải lỗi, xóa đúng file đó rồi tải lại. Ví dụ:

```bash
cd ~/Qt6Cross_Bookworm_651/gcc_all && \
rm -f gcc-12.2.0.tar.xz && \
wget https://ftp.gnu.org/gnu/gcc/gcc-12.2.0/gcc-12.2.0.tar.xz
```

Giải nén:

```bash
cd ~/Qt6Cross_Bookworm_651/gcc_all && \
tar xf binutils-2.40.tar.xz && \
tar xf gcc-12.2.0.tar.xz && \
tar xf glibc-2.36.tar.xz
```

Tải phụ thuộc của GCC:

```bash
cd ~/Qt6Cross_Bookworm_651/gcc_all/gcc-12.2.0 && \
./contrib/download_prerequisites
```

Kiểm tra:

```bash
cd ~/Qt6Cross_Bookworm_651/gcc_all/gcc-12.2.0 && \
ls -ld gmp mpfr mpc isl
```

## 7. Tạo thư mục cài toolchain riêng

```bash
cd ~/Qt6Cross_Bookworm_651 && \
sudo mkdir -p /opt/cross-pi-gcc-bookworm && \
sudo chown -R "$USER":"$USER" /opt/cross-pi-gcc-bookworm && \
export PATH=/opt/cross-pi-gcc-bookworm/bin:$PATH
```

## 8. Build Binutils 2.40

Cấu hình:

```bash
cd ~/Qt6Cross_Bookworm_651/gcc_all && \
rm -rf build-binutils && \
mkdir build-binutils && \
cd build-binutils && \
../binutils-2.40/configure \
--prefix=/opt/cross-pi-gcc-bookworm \
--target=aarch64-linux-gnu \
--with-sysroot=$HOME/Qt6Cross_Bookworm_651/rpi-sysroot \
--disable-multilib \
--disable-nls
```

Build và cài:

```bash
cd ~/Qt6Cross_Bookworm_651/gcc_all/build-binutils && \
make -j"$(nproc)" && \
make install
```

Kiểm tra:

```bash
/opt/cross-pi-gcc-bookworm/bin/aarch64-linux-gnu-as --version | head -1
```

Kết quả:

```text
GNU assembler (GNU Binutils) 2.40
```

## 9. Cấu hình và build GCC bootstrap

Cấu hình:

```bash
cd ~/Qt6Cross_Bookworm_651/gcc_all && \
rm -rf build-gcc && \
mkdir build-gcc && \
cd build-gcc && \
../gcc-12.2.0/configure \
--prefix=/opt/cross-pi-gcc-bookworm \
--target=aarch64-linux-gnu \
--with-sysroot=$HOME/Qt6Cross_Bookworm_651/rpi-sysroot \
--enable-languages=c,c++ \
--disable-multilib \
--disable-nls \
--disable-shared \
--disable-threads \
--disable-libatomic \
--disable-libgomp \
--disable-libquadmath \
--disable-libssp \
--disable-libvtv \
--without-headers
```

Build compiler bootstrap:

```bash
cd ~/Qt6Cross_Bookworm_651/gcc_all/build-gcc && \
make -j"$(nproc)" all-gcc
```

Cài compiler bootstrap:

```bash
cd ~/Qt6Cross_Bookworm_651/gcc_all/build-gcc && \
make install-gcc
```

Kiểm tra:

```bash
/opt/cross-pi-gcc-bookworm/bin/aarch64-linux-gnu-gcc --version | head -1
```

---

# PHẦN C — CHUẨN BỊ RASPBERRY PI VÀ SYSROOT

## 10. Kiểm tra Raspberry Pi

Chạy trên **RASPBERRY PI**:

```bash
cd ~ && \
echo "===== HE DIEU HANH =====" && \
grep PRETTY_NAME /etc/os-release && \
echo "===== KIEN TRUC =====" && \
uname -m && \
echo "===== GLIBC =====" && \
ldd --version | head -1 && \
echo "===== IP =====" && \
hostname -I
```

Kết quả đã kiểm thử:

```text
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
aarch64
ldd (...) 2.36
192.168.137.227
```

Bật SSH:

```bash
cd ~ && \
sudo systemctl enable --now ssh && \
systemctl is-active ssh
```

Kết quả:

```text
active
```

## 11. Kiểm tra thư viện phát triển trên Pi

Có thể kiểm tra an toàn trước bằng `--simulate`:

```bash
cd ~ && \
sudo apt-get install --simulate \
libboost-all-dev libudev-dev libinput-dev libts-dev libmtdev-dev \
libjpeg-dev libfontconfig1-dev libssl-dev libdbus-1-dev \
libglib2.0-dev libxkbcommon-dev libegl1-mesa-dev libgbm-dev \
libgles2-mesa-dev mesa-common-dev libasound2-dev libpulse-dev \
libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev \
gstreamer1.0-alsa libvpx-dev libsrtp2-dev libsnappy-dev libnss3-dev \
flex bison libxslt-dev ruby gperf libbz2-dev libcups2-dev \
libatkmm-1.6-dev libxi6 libxcomposite1 libfreetype6-dev \
libicu-dev libsqlite3-dev libxslt1-dev
```

Nhóm thứ hai:

```bash
cd ~ && \
sudo apt-get install --simulate \
libavcodec-dev libavformat-dev libswscale-dev libx11-dev freetds-dev \
libsqlite3-dev libpq-dev libiodbc2-dev firebird-dev libxext-dev \
libxcb1 libxcb1-dev libx11-xcb1 libx11-xcb-dev \
libxcb-keysyms1 libxcb-keysyms1-dev libxcb-image0 libxcb-image0-dev \
libxcb-shm0 libxcb-shm0-dev libxcb-icccm4 libxcb-icccm4-dev \
libxcb-sync1 libxcb-sync-dev libxcb-render-util0 \
libxcb-render-util0-dev libxcb-xfixes0-dev libxrender-dev \
libxcb-shape0-dev libxcb-randr0-dev libxcb-glx0-dev libxi-dev \
libdrm-dev libxcb-xinerama0 libxcb-xinerama0-dev libatspi2.0-dev \
libxcursor-dev libxcomposite-dev libxdamage-dev libxss-dev libxtst-dev \
libpci-dev libcap-dev libxrandr-dev libdirectfb-dev libaudio-dev \
libxkbcommon-x11-dev gdbserver
```

Kết quả an toàn:

```text
0 upgraded, 0 newly installed, 0 to remove
```

> Không chạy `sudo apt autoremove` trong quá trình này.

## 12. Đồng bộ sysroot từ Pi sang Host

Chạy trên **HOST Ubuntu**.

Header:

```bash
cd ~/Qt6Cross_Bookworm_651 && \
rsync -avz --delete --rsync-path="sudo rsync" \
pi@192.168.137.227:/usr/include/ \
rpi-sysroot/usr/include/
```

Thư viện `/usr/lib`:

```bash
cd ~/Qt6Cross_Bookworm_651 && \
rsync -avz --delete --rsync-path="sudo rsync" \
pi@192.168.137.227:/usr/lib/ \
rpi-sysroot/usr/lib/
```

Thư viện `/lib`:

```bash
cd ~/Qt6Cross_Bookworm_651 && \
rsync -avz --delete --rsync-path="sudo rsync" \
pi@192.168.137.227:/lib/ \
rpi-sysroot/lib/
```

Pkg-config dùng chung:

```bash
cd ~/Qt6Cross_Bookworm_651 && \
mkdir -p rpi-sysroot/usr/share/pkgconfig && \
rsync -avz --delete --rsync-path="sudo rsync" \
pi@192.168.137.227:/usr/share/pkgconfig/ \
rpi-sysroot/usr/share/pkgconfig/
```

## 13. Sửa symbolic link trong sysroot

```bash
cd ~/Qt6Cross_Bookworm_651 && \
rm -f sysroot-relativelinks.py && \
wget https://raw.githubusercontent.com/abhiTronix/rpi_rootfs/master/scripts/sysroot-relativelinks.py && \
python3 sysroot-relativelinks.py rpi-sysroot
```

Kiểm tra:

```bash
cd ~/Qt6Cross_Bookworm_651 && \
du -sh rpi-sysroot && \
ls rpi-sysroot/usr/include >/dev/null && \
ls rpi-sysroot/usr/lib >/dev/null && \
ls rpi-sysroot/lib >/dev/null && \
echo "SYSROOT OK"
```

## 14. Sửa header multiarch ARM64 trong sysroot

Trong quá trình build `libgcc`, GCC có thể báo:

```text
fatal error: bits/libc-header-start.h: No such file or directory
fatal error: sys/cdefs.h: No such file or directory
```

Cách sửa cuối cùng đã chạy thành công:

```bash
cd ~/Qt6Cross_Bookworm_651/rpi-sysroot/usr/include && \
ln -s aarch64-linux-gnu/bits bits && \
ln -s aarch64-linux-gnu/gnu gnu && \
ln -s aarch64-linux-gnu/asm asm
```

Nếu `sys` là thư mục thật và thiếu `sys/cdefs.h`, giữ lại bằng cách đổi tên rồi tạo symlink:

```bash
cd ~/Qt6Cross_Bookworm_651/rpi-sysroot/usr/include && \
mv sys sys.backup && \
ln -s aarch64-linux-gnu/sys sys
```

Kiểm tra:

```bash
cd ~/Qt6Cross_Bookworm_651/rpi-sysroot/usr/include && \
ls -ld bits gnu asm sys && \
ls -l bits/libc-header-start.h sys/cdefs.h
```

---

# PHẦN D — HOÀN TẤT GCC/G++ 12.2.0

## 15. Build và cài libgcc bootstrap

```bash
cd ~/Qt6Cross_Bookworm_651/gcc_all/build-gcc && \
make -j"$(nproc)" all-target-libgcc && \
make install-target-libgcc
```

Kiểm tra:

```bash
find /opt/cross-pi-gcc-bookworm \
\( -name 'libgcc.a' -o -name 'libgcc_s.so*' \) \
-print
```

## 16. Cấu hình GCC hoàn chỉnh

```bash
cd ~/Qt6Cross_Bookworm_651/gcc_all && \
rm -rf build-gcc-final && \
mkdir build-gcc-final && \
cd build-gcc-final && \
../gcc-12.2.0/configure \
--prefix=/opt/cross-pi-gcc-bookworm \
--target=aarch64-linux-gnu \
--with-sysroot=$HOME/Qt6Cross_Bookworm_651/rpi-sysroot \
--with-build-sysroot=$HOME/Qt6Cross_Bookworm_651/rpi-sysroot \
--enable-languages=c,c++ \
--enable-threads=posix \
--enable-shared \
--disable-multilib \
--disable-nls
```

Build:

```bash
cd ~/Qt6Cross_Bookworm_651/gcc_all/build-gcc-final && \
make -j"$(nproc)" \
all-gcc \
all-target-libgcc \
all-target-libstdc++-v3
```

Cài:

```bash
cd ~/Qt6Cross_Bookworm_651/gcc_all/build-gcc-final && \
make \
install-gcc \
install-target-libgcc \
install-target-libstdc++-v3
```

Kiểm tra:

```bash
/opt/cross-pi-gcc-bookworm/bin/aarch64-linux-gnu-gcc --version | head -1
/opt/cross-pi-gcc-bookworm/bin/aarch64-linux-gnu-g++ --version | head -1
```

Kết quả:

```text
aarch64-linux-gnu-gcc (GCC) 12.2.0
aarch64-linux-gnu-g++ (GCC) 12.2.0
```

## 17. Test compiler ARM64 bằng C++ đơn giản

```bash
cd ~/Qt6Cross_Bookworm_651 && \
cat > test_arm64.cpp << 'EOF2'
#include <iostream>

int main()
{
    std::cout << "Cross compile ARM64 OK" << std::endl;
    return 0;
}
EOF2

/opt/cross-pi-gcc-bookworm/bin/aarch64-linux-gnu-g++ \
--sysroot=$HOME/Qt6Cross_Bookworm_651/rpi-sysroot \
test_arm64.cpp \
-o test_arm64

file test_arm64
```

Kết quả:

```text
ELF 64-bit LSB pie executable, ARM aarch64
```

---

# PHẦN E — BUILD QT 6.5.1 CHO HOST

## 18. Tải và giải nén QtBase 6.5.1

```bash
cd ~/Qt6Cross_Bookworm_651/qt6/src && \
rm -f qtbase-everywhere-src-6.5.1.tar.xz && \
wget https://download.qt.io/official_releases/qt/6.5/6.5.1/submodules/qtbase-everywhere-src-6.5.1.tar.xz
```

Giải nén:

```bash
cd ~/Qt6Cross_Bookworm_651/qt6/src && \
rm -rf qtbase-everywhere-src-6.5.1 && \
tar xf qtbase-everywhere-src-6.5.1.tar.xz
```

## 19. Cấu hình Qt Host

```bash
cd ~/Qt6Cross_Bookworm_651/qt6/host-build && \
rm -rf ./* && \
cmake ../src/qtbase-everywhere-src-6.5.1/ \
-GNinja \
-DCMAKE_BUILD_TYPE=Release \
-DQT_BUILD_EXAMPLES=OFF \
-DQT_BUILD_TESTS=OFF \
-DCMAKE_INSTALL_PREFIX=$HOME/Qt6Cross_Bookworm_651/qt6/host
```

## 20. Build và cài Qt Host

```bash
cd ~/Qt6Cross_Bookworm_651/qt6/host-build && \
cmake --build . --parallel "$(nproc)" && \
cmake --install .
```

Kiểm tra:

```bash
~/Qt6Cross_Bookworm_651/qt6/host/bin/qtpaths6 --qt-version
~/Qt6Cross_Bookworm_651/qt6/host/bin/qtpaths6 --query QT_INSTALL_PREFIX
```

Kết quả:

```text
6.5.1
/home/<USER>/Qt6Cross_Bookworm_651/qt6/host
```

---

# PHẦN F — CROSS-COMPILE QT 6.5.1 CHO RASPBERRY PI

## 21. Tạo file toolchain.cmake cuối cùng

Chạy trên **HOST Ubuntu**:

```bash
cd ~/Qt6Cross_Bookworm_651/qt6/pi-build && \
cat > toolchain.cmake << 'EOF2'
cmake_minimum_required(VERSION 3.18)
include_guard(GLOBAL)

set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(TARGET_SYSROOT "$ENV{HOME}/Qt6Cross_Bookworm_651/rpi-sysroot")
set(TARGET_ARCHITECTURE aarch64-linux-gnu)

set(CMAKE_SYSROOT "${TARGET_SYSROOT}")

set(CMAKE_C_COMPILER
    /opt/cross-pi-gcc-bookworm/bin/aarch64-linux-gnu-gcc)

set(CMAKE_CXX_COMPILER
    /opt/cross-pi-gcc-bookworm/bin/aarch64-linux-gnu-g++)

set(CMAKE_FIND_ROOT_PATH
    "${TARGET_SYSROOT}"
    "$ENV{HOME}/Qt6Cross_Bookworm_651/qt6/pi")

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)

set(ENV{PKG_CONFIG_SYSROOT_DIR} "${TARGET_SYSROOT}")
set(ENV{PKG_CONFIG_LIBDIR}
    "${TARGET_SYSROOT}/usr/lib/aarch64-linux-gnu/pkgconfig:${TARGET_SYSROOT}/lib/aarch64-linux-gnu/pkgconfig:${TARGET_SYSROOT}/usr/lib/pkgconfig:${TARGET_SYSROOT}/usr/share/pkgconfig")

set(CMAKE_C_FLAGS_INIT "-march=armv8-a")
set(CMAKE_CXX_FLAGS_INIT "-march=armv8-a")

# Linker tìm dependency gián tiếp trong sysroot
set(CMAKE_EXE_LINKER_FLAGS_INIT
    "-Wl,-rpath-link,${TARGET_SYSROOT}/usr/lib/aarch64-linux-gnu -Wl,-rpath-link,${TARGET_SYSROOT}/lib/aarch64-linux-gnu")

set(CMAKE_SHARED_LINKER_FLAGS_INIT
    "-Wl,-rpath-link,${TARGET_SYSROOT}/usr/lib/aarch64-linux-gnu -Wl,-rpath-link,${TARGET_SYSROOT}/lib/aarch64-linux-gnu")

set(CMAKE_MODULE_LINKER_FLAGS_INIT
    "-Wl,-rpath-link,${TARGET_SYSROOT}/usr/lib/aarch64-linux-gnu -Wl,-rpath-link,${TARGET_SYSROOT}/lib/aarch64-linux-gnu")

# Ưu tiên Qt 6.5.1 staging, tránh lấy Qt 6.4 có sẵn trong sysroot
set(QT_TARGET_LIB_DIR "$ENV{HOME}/Qt6Cross_Bookworm_651/qt6/pi/lib")

set(CMAKE_EXE_LINKER_FLAGS_INIT
    "-Wl,-rpath-link,${QT_TARGET_LIB_DIR} -Wl,-rpath-link,${TARGET_SYSROOT}/usr/lib/aarch64-linux-gnu -Wl,-rpath-link,${TARGET_SYSROOT}/lib/aarch64-linux-gnu")

set(CMAKE_SHARED_LINKER_FLAGS_INIT
    "-Wl,-rpath-link,${QT_TARGET_LIB_DIR} -Wl,-rpath-link,${TARGET_SYSROOT}/usr/lib/aarch64-linux-gnu -Wl,-rpath-link,${TARGET_SYSROOT}/lib/aarch64-linux-gnu")

set(CMAKE_MODULE_LINKER_FLAGS_INIT
    "-Wl,-rpath-link,${QT_TARGET_LIB_DIR} -Wl,-rpath-link,${TARGET_SYSROOT}/usr/lib/aarch64-linux-gnu -Wl,-rpath-link,${TARGET_SYSROOT}/lib/aarch64-linux-gnu")
EOF2
```

## 22. Kiểm tra pkg-config xkbcommon-x11

```bash
cd ~/Qt6Cross_Bookworm_651 && \
PKG_CONFIG_SYSROOT_DIR="$HOME/Qt6Cross_Bookworm_651/rpi-sysroot" \
PKG_CONFIG_LIBDIR="$HOME/Qt6Cross_Bookworm_651/rpi-sysroot/usr/lib/aarch64-linux-gnu/pkgconfig:$HOME/Qt6Cross_Bookworm_651/rpi-sysroot/lib/aarch64-linux-gnu/pkgconfig:$HOME/Qt6Cross_Bookworm_651/rpi-sysroot/usr/lib/pkgconfig:$HOME/Qt6Cross_Bookworm_651/rpi-sysroot/usr/share/pkgconfig" \
pkg-config --modversion xkbcommon-x11
```

Kết quả đã kiểm thử:

```text
1.5.0
```

## 23. Cấu hình Qt ARM64

```bash
cd ~/Qt6Cross_Bookworm_651/qt6/pi-build && \
rm -rf CMakeCache.txt CMakeFiles && \
cmake ../src/qtbase-everywhere-src-6.5.1/ \
-GNinja \
-DCMAKE_BUILD_TYPE=Release \
-DQT_BUILD_EXAMPLES=OFF \
-DQT_BUILD_TESTS=OFF \
-DQT_HOST_PATH=$HOME/Qt6Cross_Bookworm_651/qt6/host \
-DCMAKE_STAGING_PREFIX=$HOME/Qt6Cross_Bookworm_651/qt6/pi \
-DCMAKE_INSTALL_PREFIX=/usr/local/qt6 \
-DCMAKE_TOOLCHAIN_FILE=$HOME/Qt6Cross_Bookworm_651/qt6/pi-build/toolchain.cmake \
-DQT_QMAKE_TARGET_MKSPEC=devices/linux-rasp-pi4-aarch64 \
-DINPUT_opengl=es2 \
-DQT_FEATURE_xcb=ON \
-DFEATURE_xcb_xlib=ON \
-DQT_FEATURE_xlib=ON
```

Kết quả cần có:

```text
-- Configuring done
-- Generating done
```

## 24. Build và cài staging Qt ARM64

```bash
cd ~/Qt6Cross_Bookworm_651/qt6/pi-build && \
cmake --build . --parallel "$(nproc)" && \
cmake --install .
```

Kiểm tra:

```bash
cd ~/Qt6Cross_Bookworm_651 && \
file qt6/pi/lib/libQt6Core.so.6.5.1 && \
du -sh qt6/pi
```

Kết quả:

```text
ELF 64-bit LSB shared object, ARM aarch64
```

---

# PHẦN G — CHÉP QT 6.5.1 SANG RASPBERRY PI

## 25. Sao lưu Qt cũ trên Pi

Chạy trên **HOST Ubuntu**:

```bash
cd ~/Qt6Cross_Bookworm_651 && \
ssh pi@192.168.137.227 \
'sudo cp -a /usr/local/qt6 /usr/local/qt6_backup_before_bookworm_651'
```

Kiểm tra:

```bash
cd ~/Qt6Cross_Bookworm_651 && \
ssh pi@192.168.137.227 \
'ls -ld /usr/local/qt6 /usr/local/qt6_backup_before_bookworm_651'
```

## 26. Đồng bộ Qt mới sang Pi

```bash
cd ~/Qt6Cross_Bookworm_651 && \
rsync -avz --delete \
--rsync-path="sudo rsync" \
qt6/pi/ \
pi@192.168.137.227:/usr/local/qt6/
```

Kiểm tra:

```bash
cd ~/Qt6Cross_Bookworm_651 && \
ssh pi@192.168.137.227 \
'file /usr/local/qt6/lib/libQt6Core.so.6.5.1 && du -sh /usr/local/qt6'
```

---

# PHẦN H — TEST CROSS-COMPILE ỨNG DỤNG QT

## 27. Tạo project test

Chạy trên **HOST Ubuntu**:

```bash
cd ~/Qt6Cross_Bookworm_651 && \
rm -rf test_qt_arm64 && \
mkdir -p test_qt_arm64 && \
cd test_qt_arm64 && \
cat > CMakeLists.txt << 'EOF2'
cmake_minimum_required(VERSION 3.18)

project(TestQtArm64 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Qt6 REQUIRED COMPONENTS Widgets)

qt_add_executable(test_qt_arm64
    main.cpp
)

target_link_libraries(test_qt_arm64 PRIVATE Qt6::Widgets)
EOF2

cat > main.cpp << 'EOF2'
#include <QApplication>
#include <QLabel>

int main(int argc, char *argv[])
{
    QApplication app(argc, argv);

    QLabel label("Qt 6.5.1 cross-compile ARM64 OK");
    label.resize(420, 100);
    label.show();

    return app.exec();
}
EOF2
```

## 28. Cấu hình chương trình test bằng đúng Qt 6.5.1

```bash
cd ~/Qt6Cross_Bookworm_651/test_qt_arm64 && \
rm -rf build && \
mkdir build && \
cd build && \
cmake .. \
-GNinja \
-DCMAKE_BUILD_TYPE=Release \
-DCMAKE_TOOLCHAIN_FILE=$HOME/Qt6Cross_Bookworm_651/qt6/pi-build/toolchain.cmake \
-DQt6_DIR=$HOME/Qt6Cross_Bookworm_651/qt6/pi/lib/cmake/Qt6 \
-DQt6Core_DIR=$HOME/Qt6Cross_Bookworm_651/qt6/pi/lib/cmake/Qt6Core \
-DQt6Gui_DIR=$HOME/Qt6Cross_Bookworm_651/qt6/pi/lib/cmake/Qt6Gui \
-DQt6Widgets_DIR=$HOME/Qt6Cross_Bookworm_651/qt6/pi/lib/cmake/Qt6Widgets \
-DQt6DBus_DIR=$HOME/Qt6Cross_Bookworm_651/qt6/pi/lib/cmake/Qt6DBus
```

## 29. Build chương trình test

```bash
cd ~/Qt6Cross_Bookworm_651/test_qt_arm64/build && \
cmake --build . --parallel "$(nproc)"
```

Kiểm tra kiến trúc:

```bash
cd ~/Qt6Cross_Bookworm_651/test_qt_arm64/build && \
file test_qt_arm64
```

Kết quả:

```text
ELF 64-bit LSB pie executable, ARM aarch64
```

## 30. Chép và chạy thủ công trên Pi

Chép từ Host:

```bash
cd ~/Qt6Cross_Bookworm_651/test_qt_arm64/build && \
scp test_qt_arm64 pi@192.168.137.227:/home/pi/
```

Chạy trực tiếp trên **Raspberry Pi**:

```bash
cd /home/pi && \
chmod +x test_qt_arm64 && \
LD_LIBRARY_PATH=/usr/local/qt6/lib:$LD_LIBRARY_PATH \
QT_PLUGIN_PATH=/usr/local/qt6/plugins \
QT_QPA_PLATFORM=xcb \
./test_qt_arm64
```

Kết quả: cửa sổ Qt hiện trên màn hình Pi.

> Không bắt buộc lưu `LD_LIBRARY_PATH` và `QT_PLUGIN_PATH` vào `~/.bashrc`. Với Pi có nhiều chương trình Qt khác nhau, nên đặt biến riêng trong lệnh hoặc script chạy ứng dụng để tránh nạp nhầm phiên bản Qt.

---

# PHẦN I — SCRIPT BUILD, DEPLOY VÀ CHẠY TỰ ĐỘNG TRÊN MÀN HÌNH PI

## 31. Script cuối cùng đã chạy thành công

Tạo trên **HOST Ubuntu**:

```bash
cd ~/Qt6Cross_Bookworm_651/test_qt_arm64 && \
cat > build_and_run_pi.sh << 'EOF2'
#!/bin/bash
set -e

PROJECT_DIR="$HOME/Qt6Cross_Bookworm_651/test_qt_arm64"
BUILD_DIR="$PROJECT_DIR/build"

PI_USER="pi"
PI_IP="192.168.137.227"
PI_APP="/home/pi/test_qt_arm64"

echo "===== 1. BUILD TREN HOST ====="
cd "$BUILD_DIR"
cmake --build . --parallel "$(nproc)"

echo "===== 2. KIEM TRA FILE ARM64 ====="
file "$BUILD_DIR/test_qt_arm64"

echo "===== 3. CHEP SANG RASPBERRY PI ====="
scp "$BUILD_DIR/test_qt_arm64" \
"$PI_USER@$PI_IP:$PI_APP"

echo "===== 4. CHAY TREN MAN HINH RASPBERRY PI ====="
ssh "$PI_USER@$PI_IP" '
chmod +x /home/pi/test_qt_arm64

pkill -x test_qt_arm64 2>/dev/null || true

export DISPLAY=:0
export XAUTHORITY=/home/pi/.Xauthority
export XDG_RUNTIME_DIR=/run/user/$(id -u)
export LD_LIBRARY_PATH=/usr/local/qt6/lib:$LD_LIBRARY_PATH
export QT_PLUGIN_PATH=/usr/local/qt6/plugins
export QT_QPA_PLATFORM=xcb

nohup /home/pi/test_qt_arm64 \
>/home/pi/test_qt_arm64.log 2>&1 </dev/null &

sleep 2
pgrep -a test_qt_arm64 || true
'

echo "Da gui lenh chay sang Raspberry Pi."
echo "Giao dien se hien tren man hinh Raspberry Pi."
EOF2

chmod +x build_and_run_pi.sh
```

Chạy:

```bash
cd ~/Qt6Cross_Bookworm_651/test_qt_arm64 && \
./build_and_run_pi.sh
```

Script sẽ:

```text
Build trên Host
→ kiểm tra file ARM64
→ chép file sang Pi
→ dừng đúng tiến trình cũ bằng pkill -x
→ đặt biến môi trường riêng cho ứng dụng
→ chạy nền bằng nohup
→ hiển thị trực tiếp trên màn hình Raspberry Pi
```

Nếu giao diện không hiện, xem log trên Pi:

```bash
cat /home/pi/test_qt_arm64.log
```

Kiểm tra môi trường desktop trực tiếp trên Pi:

```bash
cd ~ && \
echo "DISPLAY=$DISPLAY" && \
echo "WAYLAND_DISPLAY=$WAYLAND_DISPLAY" && \
echo "XDG_SESSION_TYPE=$XDG_SESSION_TYPE" && \
echo "XAUTHORITY=$XAUTHORITY"
```

Cấu hình Pi đã kiểm thử:

```text
DISPLAY=:0
WAYLAND_DISPLAY=wayland-0
XDG_SESSION_TYPE=wayland
XAUTHORITY=/home/pi/.Xauthority
```

---

# PHẦN J — CÁC LỖI ĐÃ GẶP VÀ CÁCH SỬA CUỐI CÙNG

## 32. `bits/libc-header-start.h` không tồn tại

Lỗi:

```text
fatal error: bits/libc-header-start.h: No such file or directory
```

Sửa:

```bash
cd ~/Qt6Cross_Bookworm_651/rpi-sysroot/usr/include && \
ln -s aarch64-linux-gnu/bits bits && \
ln -s aarch64-linux-gnu/gnu gnu && \
ln -s aarch64-linux-gnu/asm asm
```

## 33. `sys/cdefs.h` không tồn tại

Lỗi:

```text
fatal error: sys/cdefs.h: No such file or directory
```

Sửa:

```bash
cd ~/Qt6Cross_Bookworm_651/rpi-sysroot/usr/include && \
mv sys sys.backup && \
ln -s aarch64-linux-gnu/sys sys
```

## 34. OpenGL test lỗi `__glDispatch...`

Lỗi:

```text
libEGL.so: undefined reference to __glDispatch...
libGLESv2.so: undefined reference to __glDispatch...
```

Nguyên nhân: linker không tìm thấy dependency gián tiếp `libGLdispatch.so.0` trong sysroot.

Sửa bằng `-Wl,-rpath-link` trong `toolchain.cmake`.

## 35. Thiếu `PkgConfig::XKB_COMMON_X11`

Lỗi:

```text
Target links to target PkgConfig::XKB_COMMON_X11 but the target was not found
```

Nguyên nhân: chưa đồng bộ `/usr/share/pkgconfig`, dẫn đến thiếu `xproto.pc` và `kbproto.pc`.

Sửa:

```bash
cd ~/Qt6Cross_Bookworm_651 && \
rsync -avz --delete --rsync-path="sudo rsync" \
pi@192.168.137.227:/usr/share/pkgconfig/ \
rpi-sysroot/usr/share/pkgconfig/
```

## 36. Linker lấy nhầm Qt 6.4 trong sysroot

Lỗi có đường dẫn dạng:

```text
rpi-sysroot/usr/lib/aarch64-linux-gnu/libQt6DBus.so.6
```

trong khi project cần Qt 6.5.1 mới.

Sửa:

- Chỉ định trực tiếp các biến `Qt6_DIR`, `Qt6Core_DIR`, `Qt6Gui_DIR`, `Qt6Widgets_DIR`, `Qt6DBus_DIR`.
- Đặt thư mục staging Qt 6.5.1 đứng trước sysroot trong `rpath-link`.

## 37. Giao diện hiện trên Host thay vì trên Pi

Nguyên nhân: dùng `ssh -X`, X11 forwarding sẽ đưa cửa sổ từ Pi về Host.

Không dùng:

```bash
ssh -X ...
```

Dùng SSH thường và đặt:

```bash
DISPLAY=:0
XAUTHORITY=/home/pi/.Xauthority
XDG_RUNTIME_DIR=/run/user/$(id -u)
```

## 38. `pkill -f` làm ngắt luôn phiên SSH

Không dùng:

```bash
pkill -f /home/pi/test_qt_arm64
```

Vì chuỗi lệnh SSH cũng chứa đường dẫn đó.

Dùng:

```bash
pkill -x test_qt_arm64 2>/dev/null || true
```

---

# 39. Kết quả cuối cùng

Môi trường đã hoạt động thành công:

```text
HOST Ubuntu x86_64
    ↓ GCC/G++ 12.2.0 cross-compiler
    ↓ Qt 6.5.1 ARM64
    ↓ Build ứng dụng Qt
    ↓ SCP + SSH
Raspberry Pi 4 Bookworm aarch64
    ↓ /usr/local/qt6
    ↓ plugin xcb
    ↓ giao diện hiển thị trực tiếp trên màn hình Pi
```

Các đường dẫn chính:

```text
Workspace Host:
~/Qt6Cross_Bookworm_651

Cross compiler:
/opt/cross-pi-gcc-bookworm

Sysroot:
~/Qt6Cross_Bookworm_651/rpi-sysroot

Qt Host:
~/Qt6Cross_Bookworm_651/qt6/host

Qt ARM64 staging:
~/Qt6Cross_Bookworm_651/qt6/pi

Qt runtime trên Pi:
/usr/local/qt6

Backup Qt cũ trên Pi:
/usr/local/qt6_backup_before_bookworm_651
```

