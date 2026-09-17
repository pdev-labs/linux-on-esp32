# Linux on ESP32-S3: A Detailed Roadmap

> [!WARNING]
> Running Linux on an ESP32-S3 is a research/hobbyist project. The ESP32-S3 lacks a Memory Management Unit (MMU), meaning it cannot run standard desktop-class Linux distributions (like Debian or Ubuntu) or applications requiring process isolation. It runs a minimal "NOMMU" Linux kernel (often with BusyBox).

This roadmap will guide you through the process of building and running a functional, stripped-down Linux environment natively on the Xtensa cores of your ESP32-S3.

---

## Phase 1: Preparation & Prerequisites

Before starting, you must ensure you have the correct hardware and software setup. 

### Hardware Requirements
Not all ESP32-S3 boards can run Linux. The Linux kernel requires significant memory.
- **Microcontroller**: ESP32-S3 (Dual-core Xtensa LX7)
- **PSRAM**: Minimum **8 MB** (Required for the kernel and RAM).
- **Flash Memory**: Minimum **8 MB** (16 MB recommended for storing the filesystem).
- **Recommended Board Config**: ESP32-S3-WROOM-N16R8 or N8R8.
- **Connection**: USB cable capable of data transfer (connected to the UART/Console port of the ESP32).

### Software Requirements (Host PC)
Building a cross-compiled Linux kernel from scratch requires dozens of dependencies. The community standard approach is to use **Docker** to encapsulate the build environment.
- **OS**: Linux, macOS, or Windows (via WSL2).
- **Tools installed**: `git`, `docker`.

> [!TIP]
> Ensure Docker is running and your user has permissions to run docker commands (e.g., added to the `docker` group on Linux) before proceeding.

---

## Phase 2: Building the Linux Image

We will use the popular community Docker builder based on `jcmvbkbc`'s original Xtensa Linux port, typically wrapped by community members (like `hpsaturn/esp32s3-linux`) for ease of use.

### 1. Clone the Build Environment
Open your terminal and clone the Docker builder repository:
```bash
git clone --recursive https://github.com/hpsaturn/esp32s3-linux.git
cd esp32s3-linux
```

### 2. Build the Docker Container
Build the base Docker image that contains the Xtensa toolchains and buildroot environments. This step prepares the compiler.
```bash
docker build --build-arg DOCKER_USER=$USER --build-arg DOCKER_USERID=$UID -t esp32linuxbase .
```

### 3. Configure the Build Settings
Copy the default settings configuration file:
```bash
cp settings.cfg.default settings.cfg
```
*You can edit `settings.cfg` if you need to tweak board configurations, but the defaults usually work for standard N8R8/N16R8 boards.*

### 4. Compile Linux
Run the compilation script inside the Docker container. This will download the Linux kernel source, patch it for ESP32-S3 NOMMU, compile the kernel, and build the root filesystem.

```bash
docker run --rm -it --name esp32s3linux --user="$(id -u):$(id -g)" \
  -v ./esp32-linux-build:/app --env-file settings.cfg \
  esp32linuxbase ./rebuild-esp32s3-linux-wifi.sh
```
> [!NOTE]
> This process will download several gigabytes of data and compile the Linux kernel from scratch. It typically takes **30 to 45 minutes** depending on your CPU speed and internet connection.

---

## Alternative Phase 2: Building via GitHub Actions (Low Disk Space)

If you are low on local disk space (the build requires around 15-20 GB), you can leverage GitHub Actions to compile the firmware and download the resulting binaries.

### 1. Create a New GitHub Repository
1. Go to GitHub and create a new repository (e.g., `esp32s3-linux-builder`).
2. Clone it locally (you will only need a few kilobytes of space for this).

### 2. Create the GitHub Actions Workflow
In your new repository, create a directory structure `.github/workflows/` and add a file named `build.yml`:

```yaml
name: Build ESP32-S3 Linux

on:
  workflow_dispatch: # Allows manual trigger from the Actions tab

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Free Disk Space
        uses: jlumbroso/free-disk-space@main
        with:
          tool-cache: false
          android: true
          dotnet: true
          haskell: true
          large-packages: true
          docker-images: true
          swap-storage: true

      - name: Checkout Builder Repo
        run: |
          git clone --recursive https://github.com/hpsaturn/esp32s3-linux.git

      - name: Build Docker Image
        run: |
          cd esp32s3-linux
          docker build --build-arg DOCKER_USER=$USER --build-arg DOCKER_USERID=$UID -t esp32linuxbase .

      - name: Compile Linux
        run: |
          cd esp32s3-linux
          cp settings.cfg.default settings.cfg
          # Patch the build script to remove interactive flashing loops.
          # This deletes everything from the 'ready to flash' prompt to the end of the file.
          sed -i '/read -p .ready to flash/,$d' esp32-linux-build/rebuild-esp32s3-linux-wifi.sh
          # Run the build.
          docker run --rm --name esp32s3linux --user="$(id -u):$(id -g)" \
            -v ./esp32-linux-build:/app --env-file settings.cfg \
            esp32linuxbase ./rebuild-esp32s3-linux-wifi.sh

      - name: Upload Binaries
        uses: actions/upload-artifact@v4
        with:
          name: esp32s3-firmware
          path: |
            esp32s3-linux/esp32-linux-build/**/*.bin
            esp32s3-linux/esp32-linux-build/**/xipImage
            esp32s3-linux/esp32-linux-build/**/*.cramfs
            esp32s3-linux/esp32-linux-build/**/*.jffs2
```

### 3. Run and Download
1. Push this file to your GitHub repository.
2. Go to the **Actions** tab of your repository on GitHub.
3. Select **Build ESP32-S3 Linux** and click **Run workflow**.
4. Once the action completes (after about 30-40 minutes), download the `esp32s3-firmware` artifact zip from the workflow summary page. 
5. Extract it locally, and proceed directly to **Phase 3** to flash your board.

---


## Phase 3: Flashing the ESP32-S3

Once the build successfully completes, the compiled binaries (Bootloader, Partition Table, Linux Kernel, and Root Filesystem) will be ready in the output directory.

### 1. Connect your Board
Plug your ESP32-S3 into your computer. Identify its serial port:
- **Linux**: Usually `/dev/ttyUSB0` or `/dev/ttyACM0`
- **Windows/macOS**: Check Device Manager or `/dev/cu.usbserial-*`

### 2. Flash using the Docker environment
If you are on Linux, you can pass the device directly to the docker run command to flash it. Alternatively, you can use `esptool.py` locally on your host machine to flash the generated binaries. 

To use `esptool.py` directly from your host:
```bash
pip install esptool
esptool.py -p /dev/ttyACM0 -b 460800 --before default_reset --after hard_reset --chip esp32s3 write_flash \
  0x0 build/esp-hosted/esp_hosted_ng/esp/esp_driver/network_adapter/build/bootloader/bootloader.bin \
  0x8000 build/esp-hosted/esp_hosted_ng/esp/esp_driver/network_adapter/build/partition_table/partition-table.bin \
  0x10000 build/esp-hosted/esp_hosted_ng/esp/esp_driver/network_adapter/build/network_adapter.bin \
  0xb0000 build/build-buildroot-esp32s3_devkit_c1_8m/images/etc.jffs2 \
  0x120000 build/build-buildroot-esp32s3_devkit_c1_8m/images/xipImage \
  0x480000 build/build-buildroot-esp32s3_devkit_c1_8m/images/rootfs.cramfs
```
*(Note: Check the build output logs; the exact memory addresses and filenames like `xipImage` might vary slightly based on the script's output).*

---

## Phase 4: Booting and Usage

### 1. Connect via Serial Console
Because there is no display output, you must interact with the Linux system via the serial port using a terminal emulator like `picocom`, `minicom`, or `screen`.

```bash
picocom -b 115200 /dev/ttyACM0
```

### 2. The Boot Process
Press the **RST / EN** button on your ESP32-S3. 
You should see the ESP-IDF bootloader start, followed by the familiar Linux Kernel boot logs (`dmesg` style output). It takes a few seconds.

### 3. Login
Eventually, you will be greeted by a prompt:
```
Welcome to Buildroot
esp32s3 login: 
```
Log in as `root` (usually no password is required by default).

### 4. What can you do?
You are now running Linux! You have access to a basic BusyBox environment.
- Run `ls`, `cd`, `pwd`, `top`, `free`.
- Write simple C programs (if compiled into the image).
- If you used the WiFi build script, you can configure `wpa_supplicant` to connect to your local network and use tools like `ping` or `wget`.

---

## Phase 5: Next Steps & Exploration

- **Customizing the Kernel**: You can drop into the Buildroot `menuconfig` or Linux `menuconfig` to add drivers, change filesystem types (like switching to `jffs2` for a read/write filesystem instead of read-only `cramfs`), or add custom packages.
- **Networking**: Explore setting up a lightweight web server (like `httpd` from BusyBox) running directly on the ESP32-S3's Linux.
- **Understanding Architecture**: Research how the ESP-IDF running on one core handles WiFi hardware, while the Linux kernel running on the other core communicates with it via shared memory.
