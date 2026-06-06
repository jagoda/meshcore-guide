# Flash Your First Device

MeshCore ships pre-built firmware and a browser-based flasher — no compiler or
Python toolchain required for standard use.

**→ [meshcore.io/flasher](https://meshcore.io/flasher)**

!!! note "Browser requirement"
    The flasher uses the Web Serial API. Use **Google Chrome** or another
    Chromium-based browser. Firefox and Safari cannot access the serial port.

---

## Step 1 — Choose your firmware type

Before you flash, decide what role this device will play:

| Firmware type | What it does | How you connect |
|---------------|-------------|-----------------|
| **Companion Radio (BLE)** | Your personal radio; pairs with the mobile app over Bluetooth | BLE → Android / iOS / desktop app |
| **Companion Radio (USB)** | Same as BLE Companion but connects over USB serial | USB → web app or Node.js client |
| **Companion Radio (Wi-Fi)** | Connects over Wi-Fi (requires compiling with custom SSID/password) | Wi-Fi → web app |
| **Repeater** | Relays packets, extends range; no app interface | Configured via USB console or remote admin over LoRa |
| **Room Server** | Shared message store (BBS); retains history for offline users | Configured via USB console or remote admin over LoRa |

!!! tip "For your first device: choose BLE Companion"
    Flash your first device as **Companion Radio (BLE)**. You'll pair it with
    the mobile app over Bluetooth and have a working messaging interface in
    minutes. Once you understand the system, add a second device as a
    **Repeater** to extend range.

---

## Step 2 — Flash an ESP32-based device

Heltec V3, Heltec V4, Lilygo T3S3, Station G2/G3, and similar devices use an
ESP32 chip. Flashing is fully browser-based:

1. Open [meshcore.io/flasher](https://meshcore.io/flasher) in Chrome.
2. Connect your device to your computer via USB.
3. Select your device from the list.
4. Select the firmware type (e.g. **Companion Radio (BLE)**).
5. Click **Flash**.
6. When prompted, select the serial port for your device (it usually appears as
   `USB Serial` or similar).
7. Wait for the flash to complete (typically 1–2 minutes).

The device reboots automatically when flashing is done.

---

## Step 3 — Flash an nRF52-based device

RAK WisBlock (RAK4631), Heltec T114, Seeed Xiao nRF52, and T1000-E devices use
an nRF52 chip. These appear as a USB mass-storage drive when in DFU mode:

1. Put the device into DFU mode (see [DFU modes](#dfu-modes) below).
2. A drive will appear on your computer (e.g. `FTHR840BOOT`, `RAK4631`).
3. On [meshcore.io/flasher](https://meshcore.io/flasher), download the
   firmware **ZIP** for your device and firmware type.
4. Drag and drop the ZIP onto the DFU drive.
5. The device flashes and reboots automatically. The drive will disappear when
   it is done.

Alternatively, the flasher site can walk you through nRF flashing interactively.

---

## DFU modes

Some devices need a manual trigger to enter DFU (Device Firmware Update) mode
before flashing can begin:

=== "Lilygo T-Deck"
    1. Turn the device off.
    2. Connect the USB cable.
    3. Hold the **trackball button** (keep holding).
    4. Turn the device on.
    5. Wait for the USB connection sound on your computer.
    6. Release the trackball.
    7. The device is now in DFU mode — proceed with flashing.

=== "RAK WisBlock"
    Double-click the **Reset button**. The DFU drive appears on your computer.

=== "Heltec T114"
    Double-click the **lower Reset button**. The DFU drive appears.

=== "Seeed T1000-E"
    Quickly disconnect and reconnect the magnetic USB cable **twice** in
    succession. The DFU drive appears.

=== "Seeed Xiao nRF52"
    Single-click the Reset button. If that doesn't work, double-click it.
    If the device still doesn't enter DFU mode, disconnect it from USB and
    reconnect.

---

## Step 4 — Verify the flash

After flashing completes:

- **ESP32 devices**: the device reboots automatically. Open the flasher's
  **Console** tab (or any serial monitor at 115200 baud) and press Enter. You
  should see MeshCore boot output including the firmware version.
- **nRF52 devices**: the DFU drive disappears when the flash is done — the
  device is running MeshCore. Connect the device via the console on the flasher
  or the config tool to confirm.

If you flashed BLE Companion firmware, your device is now ready to pair with the
mobile app.

---

## Troubleshooting

**Linux: "Failed to open serial port" error in the browser**

The browser user needs access to the serial device:
```
sudo setfacl -m u:$USER:rw /dev/ttyUSB0
```
Replace `ttyUSB0` with the actual device path (check `ls /dev/tty*` before and
after plugging in to identify it).

**Device doesn't appear as a serial port at all**

Try a different USB cable — some cables carry power only and have no data wires.
On Linux, also check `dmesg | tail` after plugging in to confirm the device is
recognised by the kernel.

**nRF52 device appears corrupted**

Download the `flash_erase*.uf2` file for your device from
[meshcore.io/flasher](https://meshcore.io/flasher), drag it onto the DFU drive,
then re-flash the firmware from scratch. Starting with firmware v1.7.0, nRF
devices with a user button also support a CLI rescue mode: hold the user button
within 8 seconds of boot, then use the flasher Console to recover.

---

!!! tip "Next step"
    Device flashed? [Connect a client app →](connect-a-client.md)
