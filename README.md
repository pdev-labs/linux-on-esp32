# Linux on ESP32-S3

This repository contains everything you need to compile, flash, and run a fully functional NOMMU Linux kernel on the **ESP32-S3** microcontroller. It is based on the amazing port by [jcmvbkbc](https://github.com/jcmvbkbc).

## 🚀 Features

- **Linux Kernel 6.11 (NOMMU)** running on Xtensa architecture
- **Buildroot Environment**: Extremely lightweight root filesystem
- **Wi-Fi Connectivity**: Full WPA2 support via `wpa_supplicant` and the ESP-Hosted network adapter
- **Hardware Control**: Includes full suite of I2C utilities (`i2cdetect`, `i2cget`, `i2cset`) to interact directly with sensors
- **Scripting Ready**: Includes `vi`, `awk`, `sed`, `wget`, `ping`, and more
- **Persistent Storage**: Configured with a dedicated JFFS2 partition for `/etc`, meaning Wi-Fi passwords and startup scripts survive reboots

---

## 📦 Download the Binaries

If you do not want to compile the kernel from scratch, you can download the latest pre-compiled binaries from the **Releases** tab of this repository. The release contains all necessary files:

1. `bootloader.bin`
2. `partition-table.bin`
3. `network_adapter.bin`
4. `xipImage`
5. `rootfs.cramfs`
6. `etc.jffs2`

---

## ⚡ Flashing Instructions

To flash the downloaded binaries to your ESP32-S3, you will need the official `esptool.py` utility. Ensure your board is connected via USB.

```bash
pip install esptool

# Flash the binaries (Ensure you are in the directory containing the files)
esptool.py -p /dev/ttyACM0 -b 460800 --before default_reset --after hard_reset --chip esp32s3 write_flash \
  0x0 bootloader.bin \
  0x8000 partition-table.bin \
  0x10000 network_adapter.bin \
  0xb0000 etc.jffs2 \
  0x120000 xipImage \
  0x480000 rootfs.cramfs
```
*(Replace `/dev/ttyACM0` with your actual serial port if on Windows or macOS, e.g., `COM3`)*

---

## 💻 Connecting and Using Linux

Once flashed, the ESP32-S3 will immediately boot Linux. Connect to the serial console to access the terminal:

```bash
picocom -b 115200 /dev/ttyACM0
```
*Note: The terminal might appear blank upon connection. Press `Enter` to prompt the login screen. You can log in by typing `root` (no password required).*

### Connecting to Wi-Fi
To connect to an open network:
```bash
echo 'network={' > /etc/wpa.conf
echo '  ssid="YOUR_NETWORK_NAME"' >> /etc/wpa.conf
echo '  key_mgmt=NONE' >> /etc/wpa.conf
echo '}' >> /etc/wpa.conf

killall wpa_supplicant
wpa_supplicant -B -i espsta0 -c /etc/wpa.conf
udhcpc -i espsta0
```

*(For WPA2 protected networks, replace `key_mgmt=NONE` with `psk="YOUR_PASSWORD"`)*

---

## 🏗️ Building from Source

This repository includes a highly optimized GitHub Actions workflow (`.github/workflows/build.yml`) that automatically compiles the Linux kernel in the cloud, completely bypassing the massive disk space requirements needed on a local machine.

### Triggering the Build
1. Go to the **Actions** tab of this repository.
2. Select **Build ESP32-S3 Linux** on the left.
3. Click **Run workflow**.

The workflow uses a Docker container based on Ubuntu to securely compile the cross-compilation toolchain, Buildroot environment, ESP-Hosted firmware, and the kernel itself. The entire process takes approximately 45-60 minutes, after which a ZIP file containing the binaries will be available for download in the Artifacts section.

---

## ⚖️ License
This project relies on the open-source Linux kernel and Buildroot environments and is licensed under the **GNU General Public License v3.0**. See the `LICENSE` file for details.
