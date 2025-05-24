# Viture Multi-Screen Demo (Arch Linux / KDE Plasma)

A reference application that creates multiple virtual monitors inside your VITURE Pro XR glasses using the VITURE SDK for Linux and native kernel libraries—no paid wrappers required.

---

## 📖 Overview

This demo shows how to:

- Use the VITURE One SDK for Linux to enumerate and control VITURE XR displays.
- Create and manage multiple virtual framebuffers with Kernel DRM/KMS and `libdrm`.
- Integrate with an X11+Qt application on KDE Plasma 6.3.5.
- Render separate OpenGL/Qt windows into each virtual display inside your glasses.

---

## 🚀 Prerequisites

1. **Hardware & OS**  
   - Dell XPS 13 9300 (Intel® Core™ i7-1065G7, 16 GiB RAM, Mesa Intel® Iris® Plus)
   - Arch Linux (64-bit), Kernel 6.14.6-zen1-1-zen  
   - KDE Plasma 6.3.5, KDE Frameworks 6.14.0, Qt 6.9.0  
   - X11 graphics platform

2. **Kernel & Libraries**  
   ```bash
   sudo pacman -Syu \
     linux-zen linux-zen-headers \
     libdrm xf86-video-intel \
     qt6-base qt6-declarative \
     mesa glu \
     libevdev \
     git cmake make pkg-config
````

3. **VITURE One SDK for Linux**
   Clone and install the official SDK:

   ```bash
   git clone https://first.viture.com/developer/viture-sdk-for-linux.git
   cd viture-sdk-for-linux
   mkdir build && cd build
   cmake .. \
     -DCMAKE_BUILD_TYPE=Release \
     -DBUILD_SHARED_LIBS=ON
   make -j$(nproc)
   sudo make install
   ```

4. **Udev Rules**
   Ensure your user can access the VITURE device:

   ```bash
   sudo tee /etc/udev/rules.d/99-viture.rules <<EOF
   SUBSYSTEM=="video4linux", ATTRS{idVendor}=="1d50", ATTRS{idProduct}=="6018", MODE="0666"
   EOF
   sudo udevadm control --reload
   sudo udevadm trigger
   ```

---

## 🏗️ Building the Demo

1. **Clone this repo**

   ```bash
   git clone https://github.com/your-username/viture-multi-screen.git
   cd viture-multi-screen
   ```

2. **Configure with CMake**

   ```bash
   mkdir build && cd build
   cmake .. \
     -DCMAKE_BUILD_TYPE=Release \
     -DVITURE_SDK_ROOT=/usr/local \
     -DQT6_DIR=/usr/lib/cmake/Qt6
   ```

3. **Compile**

   ```bash
   make -j$(nproc)
   ```

---

## ▶️ Running the Demo

```bash
./viture_multi_screen \
  --displays 9 \
  --width 1024 --height 768 \
  --spacing 40 \
  --layout 3x3
```

* `--displays`: Total virtual screens (max supported by your GPU).
* `--width`/`--height`: Resolution of each virtual display.
* `--spacing`: Pixel gap between logical screens.
* `--layout`: Rows×Columns grid (e.g. `3x3` for nine screens).

Each virtual framebuffer is exposed as a Linux DRM connector. The demo automatically:

1. Allocates `N` framebuffers via `libdrm` + KMS.
2. Attaches them to the VITURE device handles via the SDK.
3. Creates one Qt window per framebuffer, rendering a colored test pattern.

---

## 📂 Project Structure

```
viture-multi-screen/
├── CMakeLists.txt
├── src/
│   ├── main.cpp       # app entry & CLI parsing
│   ├── VitureManager.h/.cpp  # wraps VITURE SDK + libdrm
│   └── WindowRenderer.h/.cpp # Qt OpenGL widget per display
└── README.md
```

---

## 📝 Example Code Snippets

### Initializing VITURE & DRM

```cpp
// VitureManager.cpp
#include <viture/sdk.h>
#include <xf86drm.h>
#include <xf86drmMode.h>

bool VitureManager::initialize(int count, int w, int h) {
    // 1. SDK init
    if (!viture::init()) return false;

    // 2. Open device handle
    device_ = viture::openDevice("/dev/video0");
    // 3. Create DRM framebuffers
    for (int i = 0; i < count; ++i) {
        drmModeAddFB(drm_fd_, w, h, 24, 32, pitch, bo_handle, &fb_id_);
        connectors_.push_back(fb_id_);
    }
    // 4. Attach framebuffers to SDK
    for (auto fb : connectors_) {
        viture::attachFramebuffer(device_, fb);
    }
    return true;
}
```

### Spawning Qt Windows

```cpp
// main.cpp
int main(int argc, char** argv) {
    QGuiApplication app(argc, argv);
    VitureManager vm;
    vm.initialize(dispCount, width, height);

    // Create one window per framebuffer
    for (int i = 0; i < dispCount; ++i) {
        auto window = new WindowRenderer(vm.fbId(i), width, height);
        window->show();
    }
    return app.exec();
}
```

---

## 🛠️ Troubleshooting

* **No device found**: Check `ls /dev/video*` and udev rules.
* **Insufficient DRM connectors**: Ensure your Intel GPU supports enough framebuffers.
* **Qt rendering issues**: Verify your `QT_QPA_PLATFORM=wayland` is **not** set; use X11.

---

## 📚 References

* [VITURE One SDK for Linux documentation](https://first.viture.com/developer/viture-sdk-for-linux)
* [libdrm & KMS API Guide](https://01.org/linuxgraphics/documentation/drm)
* [Qt 6 Graphics and OpenGL Integration](https://doc.qt.io/qt-6/opengl-index.html)
