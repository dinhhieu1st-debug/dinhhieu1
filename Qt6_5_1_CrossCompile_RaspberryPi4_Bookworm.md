# Qt 6.5.1 Cross-Compile cho Raspberry Pi 4 Bookworm

Tài liệu này tổng hợp **quy trình cuối cùng đã chạy thành công** để:

- Biên dịch Qt 6.5.1 trên máy ảo Ubuntu x86_64.
- Tạo chương trình ARM64 bằng GCC 12.2.0.
- Chép chương trình và thư viện Qt sang Raspberry Pi 4.
- Chạy giao diện Qt thành công trên Raspberry Pi OS Bookworm 64-bit.

> Tài liệu tham khảo ban đầu:  
> `https://github.com/dinhquanghaICTU/DATN_dashboard_car/blob/main/doccument/cross_compiler.MD`
>
> Điểm khác quan trọng: Pi thật sử dụng **Debian 12 Bookworm, glibc 2.36**, nên bộ GCC 10.3.0 + glibc 2.31 trong tài liệu Bullseye không phù hợp. Quy trình dưới đây dùng **GCC 12.2.0 + sysroot Bookworm**.

---

## 1. Kiến trúc hệ thống

```text
Máy ảo Ubuntu x86_64
├── Viết và sửa mã nguồn
├── Chạy CMake + Ninja
├── Dùng GCC 12.2.0 cross-compiler
├── Build Qt 6.5.1 cho ARM64
└── Tạo file thực thi ARM aarch64
             │
             │ SCP / rsync
             ▼
Raspberry Pi 4 Bookworm ARM64
├── Nhận file đã biên dịch
├── Chạy Qt 6.5.1 trong /usr/local/qt6
├── Hiển thị giao diện
└── Thực thi logic, GPIO, UART, camera, cảm biến
```

Máy ảo chịu trách nhiệm **biên dịch**. Raspberry Pi chịu trách nhiệm **chạy toàn bộ chương trình**.

---

## 2. Cấu hình đã kiểm thử thành công

### Máy host

- Ubuntu 22.04.5 LTS amd64 trên VMware.
- CPU máy ảo: 6 lõi.
- RAM máy ảo: 8 GB.
- Ổ đĩa: 60 GB, phân vùng `/` đã mở rộng gần toàn bộ ổ.
- Network: NAT.

### Raspberry Pi

- Raspberry Pi 4.
- Debian GNU/Linux 12 Bookworm 64-bit.
- glibc 2.36.
- IP trong quá trình cài đặt: `192.168.137.227`.
- Tên đăng nhập: `pi`.

### Bộ công cụ

- Binutils 2.40.
- GCC/G++ 12.2.0.
- Sysroot lấy trực tiếp từ Pi Bookworm.
- QtBase 6.5.1.
- CMake + Ninja.

---

## 3. Chuẩn bị Raspberry Pi thật

### 3.1. Cập nhật hệ thống

Chạy trên **Raspberry Pi**:

```bash
sudo apt update
```

### 3.2. Cài nhóm thư viện phát triển thứ nhất

```bash
sudo apt-get install -y \
libboost-all-dev libudev-dev libinput-dev libts-dev libmtdev-dev \
libjpeg-dev libfontconfig1-dev libssl-dev libdbus-1-dev \
libglib2.0-dev libxkbcommon-dev libegl1-mesa-dev libgbm-dev \
libgles2-mesa-dev mesa-common-dev libasound2-dev libpulse-dev \
gstreamer1.0-omx libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev \
gstreamer1.0-alsa libvpx-dev libsrtp2-dev libsnappy-dev libnss3-dev \
"^libxcb.*" flex bison libxslt-dev ruby gperf libbz2-dev libcups2-dev \
libatkmm-1.6-dev libxi6 libxcomposite1 libfreetype6-dev libicu-dev \
libsqlite3-dev libxslt1-dev
```

### 3.3. Cài nhóm thư viện phát triển thứ hai

```bash
sudo apt-get install -y \
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

### 3.4. Bật SSH

```bash
sudo systemctl enable --now ssh
systemctl is-active ssh
```

Kết quả mong đợi:

```text
active
```

### 3.5. Kiểm tra hệ điều hành và glibc

```bash
ldd --version | head -1
grep PRETTY_NAME /etc/os-release
```

Kết quả đã kiểm thử:

```text
ldd (Debian GLIBC 2.36-9+rpt2+deb12u14) 2.36
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
```

---

## 4. Chuẩn bị máy ảo Ubuntu

### 4.1. Tắt sleep và tắt màn hình tự động

```bash
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
gsettings set org.gnome.desktop.session idle-delay 0
```

### 4.2. Cài các gói cần thiết

```bash
sudo apt update
sudo apt install -y \
build-essential make ninja-build git wget file rsync \
bison flex gawk texinfo python3 pkg-config \
libclang-dev libssl-dev gdb gdb-multiarch gdbserver \
libfontconfig1-dev libfreetype6-dev \
libx11-dev libx11-xcb-dev libxext-dev libxfixes-dev libxi-dev \
libxrender-dev libxcb1-dev libxcb-glx0-dev libxcb-keysyms1-dev \
libxcb-image0-dev libxcb-shm0-dev libxcb-icccm4-dev \
libxcb-sync-dev libxcb-xfixes0-dev libxcb-shape0-dev \
libxcb-randr0-dev libxcb-render-util0-dev libxcb-util-dev \
libxcb-xinerama0-dev libxcb-xkb-dev libxcb-cursor-dev \
libxkbcommon-dev libxkbcommon-x11-dev libatspi2.0-dev \
libgl1-mesa-dev libglu1-mesa-dev freeglut3-dev
```

### 4.3. Tạo thư mục làm việc

```bash
mkdir -p ~/Qt6Cross
```

---

## 5. Build cross-compiler GCC 12.2.0 cho ARM64

### 5.1. Tải mã nguồn

```bash
mkdir -p ~/Qt6Cross/gcc_all && cd ~/Qt6Cross/gcc_all

wget https://ftp.gnu.org/gnu/binutils/binutils-2.40.tar.xz
wget https://ftp.gnu.org/gnu/gcc/gcc-12.2.0/gcc-12.2.0.tar.xz
wget https://ftp.gnu.org/gnu/glibc/glibc-2.36.tar.xz
```

Giải nén và tải các phụ thuộc của GCC:

```bash
cd ~/Qt6Cross/gcc_all && \
tar xf binutils-2.40.tar.xz && \
tar xf gcc-12.2.0.tar.xz && \
tar xf glibc-2.36.tar.xz && \
cd gcc-12.2.0 && \
contrib/download_prerequisites
```

> Trong quy trình cuối cùng, glibc 2.36 không được cài đè lên Ubuntu.  
> Header và thư viện runtime của target được lấy qua sysroot từ Pi thật.

### 5.2. Tạo thư mục cài toolchain

```bash
sudo mkdir -p /opt/cross-pi-gcc
sudo chown -R "$USER":"$USER" /opt/cross-pi-gcc
export PATH=/opt/cross-pi-gcc/bin:$PATH
```

### 5.3. Build Binutils 2.40

```bash
cd ~/Qt6Cross/gcc_all && \
mkdir -p build-binutils && \
cd build-binutils && \
../binutils-2.40/configure \
--prefix=/opt/cross-pi-gcc \
--target=aarch64-linux-gnu \
--with-sysroot=$HOME/Qt6Cross/rpi-sysroot \
--disable-multilib \
--disable-nls
```

```bash
cd ~/Qt6Cross/gcc_all/build-binutils && make -j6
```

```bash
cd ~/Qt6Cross/gcc_all/build-binutils && make install
```

Kiểm tra:

```bash
export PATH=/opt/cross-pi-gcc/bin:$PATH
aarch64-linux-gnu-as --version | head -1
```

Kết quả:

```text
GNU assembler (GNU Binutils) 2.40
```

### 5.4. Cấu hình GCC bootstrap

```bash
cd ~/Qt6Cross/gcc_all && \
mkdir -p build-gcc && \
cd build-gcc && \
../gcc-12.2.0/configure \
--prefix=/opt/cross-pi-gcc \
--target=aarch64-linux-gnu \
--with-sysroot=$HOME/Qt6Cross/rpi-sysroot \
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

Build và cài compiler bootstrap:

```bash
cd ~/Qt6Cross/gcc_all/build-gcc && make -j6 all-gcc
```

```bash
cd ~/Qt6Cross/gcc_all/build-gcc && make install-gcc
```

Build và cài `libgcc`:

```bash
cd ~/Qt6Cross/gcc_all/build-gcc && make -j6 all-target-libgcc
```

```bash
cd ~/Qt6Cross/gcc_all/build-gcc && make install-target-libgcc
```

### 5.5. Build GCC hoàn chỉnh

```bash
cd ~/Qt6Cross/gcc_all && \
rm -rf build-gcc-final && \
mkdir build-gcc-final && \
cd build-gcc-final && \
../gcc-12.2.0/configure \
--prefix=/opt/cross-pi-gcc \
--target=aarch64-linux-gnu \
--with-sysroot=$HOME/Qt6Cross/rpi-sysroot \
--with-build-sysroot=$HOME/Qt6Cross/rpi-sysroot \
--enable-languages=c,c++ \
--enable-threads=posix \
--enable-shared \
--disable-multilib \
--disable-nls
```

```bash
cd ~/Qt6Cross/gcc_all/build-gcc-final && \
make -j6 all-gcc all-target-libgcc all-target-libstdc++-v3
```

```bash
cd ~/Qt6Cross/gcc_all/build-gcc-final && \
make install-gcc install-target-libgcc install-target-libstdc++-v3
```

Kiểm tra:

```bash
export PATH=/opt/cross-pi-gcc/bin:$PATH && \
aarch64-linux-gnu-gcc --version | head -1 && \
aarch64-linux-gnu-g++ --version | head -1
```

Kết quả:

```text
aarch64-linux-gnu-gcc (GCC) 12.2.0
aarch64-linux-gnu-g++ (GCC) 12.2.0
```

---

## 6. Build Qt 6.5.1 cho máy host

### 6.1. Tạo cấu trúc thư mục

```bash
cd ~/Qt6Cross && \
mkdir -p qt6/host qt6/pi qt6/host-build qt6/pi-build qt6/src
```

### 6.2. Tải QtBase 6.5.1

```bash
cd ~/Qt6Cross/qt6/src && \
wget https://download.qt.io/official_releases/qt/6.5/6.5.1/submodules/qtbase-everywhere-src-6.5.1.tar.xz
```

```bash
cd ~/Qt6Cross/qt6/src && \
tar xf qtbase-everywhere-src-6.5.1.tar.xz
```

### 6.3. Cấu hình, build và cài Qt Host

```bash
cd ~/Qt6Cross/qt6/host-build && \
cmake ../src/qtbase-everywhere-src-6.5.1/ \
-GNinja \
-DCMAKE_BUILD_TYPE=Release \
-DQT_BUILD_EXAMPLES=OFF \
-DQT_BUILD_TESTS=OFF \
-DCMAKE_INSTALL_PREFIX=$HOME/Qt6Cross/qt6/host
```

```bash
cd ~/Qt6Cross/qt6/host-build && \
cmake --build . --parallel 6
```

```bash
cd ~/Qt6Cross/qt6/host-build && \
cmake --install .
```

---

## 7. Tạo sysroot từ Raspberry Pi thật

### 7.1. Tạo thư mục sysroot

```bash
cd ~/Qt6Cross && mkdir -p rpi-sysroot/usr
```

### 7.2. Copy header và thư viện từ Pi

```bash
cd ~/Qt6Cross && \
rsync -avz --delete --rsync-path="sudo rsync" \
pi@192.168.137.227:/usr/include/ \
rpi-sysroot/usr/include/
```

```bash
cd ~/Qt6Cross && \
rsync -avz --delete --rsync-path="sudo rsync" \
pi@192.168.137.227:/usr/lib/ \
rpi-sysroot/usr/lib/
```

```bash
cd ~/Qt6Cross && \
rsync -avz --delete --rsync-path="sudo rsync" \
pi@192.168.137.227:/lib/ \
rpi-sysroot/lib/
```

### 7.3. Sửa symbolic link trong sysroot

```bash
cd ~/Qt6Cross && \
wget https://raw.githubusercontent.com/abhiTronix/rpi_rootfs/master/scripts/sysroot-relativelinks.py
```

```bash
cd ~/Qt6Cross && \
python3 sysroot-relativelinks.py rpi-sysroot
```

Kiểm tra:

```bash
cd ~/Qt6Cross && \
du -sh rpi-sysroot && \
ls rpi-sysroot/usr/include >/dev/null && \
ls rpi-sysroot/usr/lib >/dev/null && \
echo "SYSROOT OK"
```

Kết quả:

```text
SYSROOT OK
```

---

## 8. Toolchain CMake cuối cùng đã build Qt thành công

Tạo file:

```bash
cd ~/Qt6Cross/qt6/pi-build && \
rm -f toolchain.cmake && \
nano toolchain.cmake
```

Nội dung:

```cmake
cmake_minimum_required(VERSION 3.18)
include_guard(GLOBAL)

set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(TARGET_SYSROOT "$ENV{HOME}/Qt6Cross/rpi-sysroot")
set(TARGET_ARCHITECTURE aarch64-linux-gnu)

set(CMAKE_SYSROOT "${TARGET_SYSROOT}")

set(CMAKE_C_COMPILER
    /opt/cross-pi-gcc/bin/${TARGET_ARCHITECTURE}-gcc
)

set(CMAKE_CXX_COMPILER
    /opt/cross-pi-gcc/bin/${TARGET_ARCHITECTURE}-g++
)

set(CMAKE_C_COMPILER_TARGET ${TARGET_ARCHITECTURE})
set(CMAKE_CXX_COMPILER_TARGET ${TARGET_ARCHITECTURE})

set(ENV{PKG_CONFIG_SYSROOT_DIR} "${TARGET_SYSROOT}")

set(
    ENV{PKG_CONFIG_LIBDIR}
    "${TARGET_SYSROOT}/usr/lib/${TARGET_ARCHITECTURE}/pkgconfig:${TARGET_SYSROOT}/usr/lib/pkgconfig:${TARGET_SYSROOT}/usr/share/pkgconfig"
)

set(ENV{PKG_CONFIG_PATH} "")

set(
    CMAKE_FIND_ROOT_PATH
    "${TARGET_SYSROOT}"
    "$ENV{HOME}/Qt6Cross/qt6/pi"
)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)

set(COMMON_INCLUDE_FLAGS
    "-march=armv8-a
    -isystem ${TARGET_SYSROOT}/usr/include/at-spi-2.0
    -isystem ${TARGET_SYSROOT}/usr/include/gtk-3.0
    -isystem ${TARGET_SYSROOT}/usr/include/pango-1.0
    -isystem ${TARGET_SYSROOT}/usr/include/cairo
    -isystem ${TARGET_SYSROOT}/usr/include/gdk-pixbuf-2.0
    -isystem ${TARGET_SYSROOT}/usr/include/glib-2.0
    -isystem ${TARGET_SYSROOT}/usr/lib/${TARGET_ARCHITECTURE}/glib-2.0/include
    -isystem ${TARGET_SYSROOT}/usr/include/harfbuzz
    -isystem ${TARGET_SYSROOT}/usr/include/atk-1.0
")

string(REPLACE "\n" " " COMMON_INCLUDE_FLAGS "${COMMON_INCLUDE_FLAGS}")

set(CMAKE_C_FLAGS_INIT "${COMMON_INCLUDE_FLAGS}")
set(CMAKE_CXX_FLAGS_INIT "${COMMON_INCLUDE_FLAGS}")

set(CMAKE_C_FLAGS_RELEASE_INIT "-O2 -pipe")
set(CMAKE_CXX_FLAGS_RELEASE_INIT "-O2 -pipe")

set(
    CMAKE_EXE_LINKER_FLAGS_INIT
    "-Wl,-O1 -Wl,--hash-style=gnu -Wl,--as-needed -Wl,-rpath-link,${TARGET_SYSROOT}/usr/lib/${TARGET_ARCHITECTURE} -Wl,-rpath-link,${TARGET_SYSROOT}/lib/${TARGET_ARCHITECTURE}"
)

set(
    CMAKE_SHARED_LINKER_FLAGS_INIT
    "-Wl,-O1 -Wl,--hash-style=gnu -Wl,--as-needed -Wl,-rpath-link,${TARGET_SYSROOT}/usr/lib/${TARGET_ARCHITECTURE} -Wl,-rpath-link,${TARGET_SYSROOT}/lib/${TARGET_ARCHITECTURE}"
)

set(
    CMAKE_MODULE_LINKER_FLAGS_INIT
    "-Wl,-O1 -Wl,--hash-style=gnu -Wl,--as-needed -Wl,-rpath-link,${TARGET_SYSROOT}/usr/lib/${TARGET_ARCHITECTURE} -Wl,-rpath-link,${TARGET_SYSROOT}/lib/${TARGET_ARCHITECTURE}"
)

set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

set(
    CMAKE_LIBRARY_PATH
    "${TARGET_SYSROOT}/usr/lib/${TARGET_ARCHITECTURE}"
    "${TARGET_SYSROOT}/lib/${TARGET_ARCHITECTURE}"
)

set(
    CMAKE_PREFIX_PATH
    "${TARGET_SYSROOT}/usr/lib/${TARGET_ARCHITECTURE}/cmake"
    "$ENV{HOME}/Qt6Cross/qt6/pi"
)

set(GL_INC_DIR "${TARGET_SYSROOT}/usr/include")

set(EGL_INCLUDE_DIR "${TARGET_SYSROOT}/usr/include")
set(
    EGL_LIBRARY
    "${TARGET_SYSROOT}/usr/lib/${TARGET_ARCHITECTURE}/libEGL.so"
)

set(OPENGL_INCLUDE_DIR "${TARGET_SYSROOT}/usr/include")
set(
    OPENGL_opengl_LIBRARY
    "${TARGET_SYSROOT}/usr/lib/${TARGET_ARCHITECTURE}/libOpenGL.so"
)

set(GLESv2_INCLUDE_DIR "${TARGET_SYSROOT}/usr/include")
set(
    GLESv2_LIBRARY
    "${TARGET_SYSROOT}/usr/lib/${TARGET_ARCHITECTURE}/libGLESv2.so"
)

set(gbm_INCLUDE_DIR "${TARGET_SYSROOT}/usr/include")
set(
    gbm_LIBRARY
    "${TARGET_SYSROOT}/usr/lib/${TARGET_ARCHITECTURE}/libgbm.so"
)

set(Libdrm_INCLUDE_DIR "${TARGET_SYSROOT}/usr/include")
set(
    Libdrm_LIBRARY
    "${TARGET_SYSROOT}/usr/lib/${TARGET_ARCHITECTURE}/libdrm.so"
)

set(XCB_XCB_INCLUDE_DIR "${TARGET_SYSROOT}/usr/include")
set(
    XCB_XCB_LIBRARY
    "${TARGET_SYSROOT}/usr/lib/${TARGET_ARCHITECTURE}/libxcb.so"
)
```

Kiểm tra cú pháp:

```bash
cd ~/Qt6Cross/qt6/pi-build && cmake -P toolchain.cmake
```

---

## 9. Build Qt 6.5.1 cho Raspberry Pi

Xóa cache cấu hình cũ:

```bash
cd ~/Qt6Cross/qt6/pi-build && \
rm -rf CMakeCache.txt CMakeFiles config.tests
```

Cấu hình:

```bash
cd ~/Qt6Cross/qt6/pi-build && \
cmake ../src/qtbase-everywhere-src-6.5.1 \
-GNinja \
-DCMAKE_BUILD_TYPE=Release \
-DINPUT_opengl=es2 \
-DQT_BUILD_EXAMPLES=OFF \
-DQT_BUILD_TESTS=OFF \
-DQT_HOST_PATH=$HOME/Qt6Cross/qt6/host \
-DCMAKE_STAGING_PREFIX=$HOME/Qt6Cross/qt6/pi \
-DCMAKE_INSTALL_PREFIX=/usr/local/qt6 \
-DCMAKE_TOOLCHAIN_FILE=$HOME/Qt6Cross/qt6/pi-build/toolchain.cmake \
-DQT_QMAKE_TARGET_MKSPEC=devices/linux-rasp-pi4-aarch64 \
-DQT_FEATURE_xcb=ON \
-DFEATURE_xcb_xlib=ON \
-DQT_FEATURE_xlib=ON
```

Kết quả đúng:

```text
-- Configuring done
-- Generating done
-- Build files have been written to: /home/pi/Qt6Cross/qt6/pi-build
```

Build:

```bash
cd ~/Qt6Cross/qt6/pi-build && \
cmake --build . --parallel 6
```

Kết quả thành công cuối quá trình:

```text
[1201/1201] Linking CXX shared module ...
```

Cài vào staging:

```bash
cd ~/Qt6Cross/qt6/pi-build && \
cmake --install .
```

Qt target được cài tại:

```text
/home/pi/Qt6Cross/qt6/pi
```

---

## 10. Chép Qt sang Raspberry Pi

Chạy trên máy ảo:

```bash
cd ~/Qt6Cross && \
rsync -avz --delete --rsync-path="sudo rsync" \
qt6/pi/ \
pi@192.168.137.227:/usr/local/qt6/
```

Trên Pi, thêm thư viện Qt vào môi trường:

```bash
echo 'export LD_LIBRARY_PATH=/usr/local/qt6/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

Kiểm tra kiến trúc thư viện:

```bash
file -L /usr/local/qt6/lib/libQt6Core.so.6
```

Kết quả:

```text
ELF 64-bit LSB shared object, ARM aarch64
```

> `qmake6` trong thư mục target có thể là symbolic link tới Qt Host trên máy ảo.  
> Điều này không ảnh hưởng việc chạy ứng dụng trên Pi. Project được build trên máy ảo bằng CMake.

---

## 11. Demo ứng dụng biên dịch chéo

### 11.1. `main.cpp`

```cpp
#include <QApplication>
#include <QLabel>
#include <QWidget>
#include <QVBoxLayout>
#include <QFont>

int main(int argc, char *argv[])
{
    QApplication app(argc, argv);

    QWidget window;
    window.setWindowTitle("Qt 6.5.1 - Raspberry Pi");
    window.resize(500, 250);

    auto *layout = new QVBoxLayout(&window);

    auto *title = new QLabel("Qt 6.5.1 chạy thành công!");
    title->setAlignment(Qt::AlignCenter);

    QFont font;
    font.setPointSize(20);
    font.setBold(true);
    title->setFont(font);

    auto *info = new QLabel(
        "Ứng dụng được cross-compile trên máy ảo Ubuntu\n"
        "và chạy trên Raspberry Pi 4 ARM64."
    );
    info->setAlignment(Qt::AlignCenter);

    layout->addWidget(title);
    layout->addWidget(info);

    window.show();

    return app.exec();
}
```

### 11.2. `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.18)

project(QtPiTest LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(
    Qt6 6.5.1 EXACT REQUIRED
    COMPONENTS
        Core
        Gui
        Widgets
        DBus
)

qt_standard_project_setup()

qt_add_executable(QtPiTest
    main.cpp
)

target_link_libraries(
    QtPiTest
    PRIVATE
        Qt6::Core
        Qt6::Gui
        Qt6::Widgets
        Qt6::DBus
)
```

### 11.3. Cấu hình project mẫu

```bash
cd ~/Qt6Cross/test_app && \
rm -rf build && \
mkdir build && \
cd build && \
cmake .. \
-GNinja \
-DCMAKE_BUILD_TYPE=Release \
-DCMAKE_TOOLCHAIN_FILE=$HOME/Qt6Cross/qt6/pi-build/toolchain.cmake \
-DQT_HOST_PATH=$HOME/Qt6Cross/qt6/host \
-DQt6_DIR=$HOME/Qt6Cross/qt6/pi/lib/cmake/Qt6 \
-DQt6Core_DIR=$HOME/Qt6Cross/qt6/pi/lib/cmake/Qt6Core \
-DQt6Gui_DIR=$HOME/Qt6Cross/qt6/pi/lib/cmake/Qt6Gui \
-DQt6Widgets_DIR=$HOME/Qt6Cross/qt6/pi/lib/cmake/Qt6Widgets \
-DQt6DBus_DIR=$HOME/Qt6Cross/qt6/pi/lib/cmake/Qt6DBus \
-DCMAKE_PREFIX_PATH=$HOME/Qt6Cross/qt6/pi
```

### 11.4. Build

```bash
cd ~/Qt6Cross/test_app/build && \
cmake --build . --parallel 6
```

Kiểm tra kiến trúc file:

```bash
file ~/Qt6Cross/test_app/build/QtPiTest
```

Kết quả:

```text
ELF 64-bit LSB executable, ARM aarch64
```

Đây là bằng chứng biên dịch chéo:

- Máy build: `x86_64`.
- File được tạo: `ARM aarch64`.
- File không chạy trực tiếp trên máy ảo x86_64.
- File chạy trên Raspberry Pi 4 ARM64.

### 11.5. Chép chương trình sang Pi

```bash
scp ~/Qt6Cross/test_app/build/QtPiTest \
pi@192.168.137.227:/home/pi/
```

### 11.6. Chạy trên Pi

Mở Terminal **trực tiếp trên giao diện Desktop của Raspberry Pi**:

```bash
chmod +x ~/QtPiTest
~/QtPiTest
```

Nếu chạy qua SSH mà báo:

```text
qt.qpa.xcb: could not connect to display
```

thì chạy:

```bash
export DISPLAY=:0
export XAUTHORITY=/home/pi/.Xauthority
~/QtPiTest
```

Cách ổn định nhất vẫn là mở terminal trực tiếp trên Pi.

---

## 12. Quy trình làm việc hằng ngày

### Trên máy ảo Ubuntu

Sửa code rồi build:

```bash
cd ~/duong-dan-project/build
cmake --build . --parallel 6
```

Chép file sang Pi:

```bash
scp ten_chuong_trinh \
pi@192.168.137.227:/home/pi/
```

### Trên Raspberry Pi

```bash
chmod +x ~/ten_chuong_trinh
~/ten_chuong_trinh
```

---

## 13. Các lỗi quan trọng và kết luận cuối cùng

### Không trộn glibc 2.31 với sysroot Bookworm glibc 2.36

Bộ cũ:

```text
GCC 10.3.0 + glibc 2.31
```

không tương thích với:

```text
Pi Bookworm + glibc 2.36
```

Biểu hiện:

```text
__issignalingf was not declared
__iseqsigf was not declared
```

Cách xử lý cuối cùng:

```text
GCC 12.2.0 + sysroot lấy trực tiếp từ Pi Bookworm
```

### Không dùng header `/usr/include` của máy host cho target

Không thêm trực tiếp:

```text
/usr/include
/usr/local/include
```

vì đây là header amd64 của Ubuntu.

### Khi thêm include vào toolchain phải xóa cache CMake

```bash
rm -rf CMakeCache.txt CMakeFiles config.tests
```

sau đó cấu hình lại.

### Không trộn Qt trong sysroot với Qt 6.5.1 tự build

Khi build ứng dụng phải ép các module Qt về:

```text
~/Qt6Cross/qt6/pi
```

Đặc biệt:

```bash
-DQt6_DIR=$HOME/Qt6Cross/qt6/pi/lib/cmake/Qt6
-DQt6Core_DIR=$HOME/Qt6Cross/qt6/pi/lib/cmake/Qt6Core
-DQt6Gui_DIR=$HOME/Qt6Cross/qt6/pi/lib/cmake/Qt6Gui
-DQt6Widgets_DIR=$HOME/Qt6Cross/qt6/pi/lib/cmake/Qt6Widgets
-DQt6DBus_DIR=$HOME/Qt6Cross/qt6/pi/lib/cmake/Qt6DBus
```

Nếu không, linker có thể trộn:

```text
QtCore 6.5.1 tự build
QtDBus của hệ thống Pi
```

và báo undefined reference.

---

## 14. Kết quả cuối cùng

Quá trình đã được xác nhận thành công bằng ứng dụng giao diện hiển thị trên Raspberry Pi:

```text
Qt 6.5.1 chạy thành công!

Ứng dụng được cross-compile trên máy ảo Ubuntu
và chạy trên Raspberry Pi 4 ARM64.
```

Kết luận:

> Ubuntu x86_64 dùng GCC 12.2.0 và Qt 6.5.1 để tạo binary ARM64.  
> Raspberry Pi 4 Bookworm nhận binary qua SCP và chạy bằng Qt tại `/usr/local/qt6`.
