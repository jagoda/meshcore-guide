# Connect a Client

With firmware flashed, the next step is connecting a companion app to your radio
so you can configure it and send messages.

---

## Step 1 — Install the app

Pick the client that fits your platform:

| Platform | App | Connection type |
|----------|-----|----------------|
| Android | [Play Store → MeshCore](https://play.google.com/store/apps/details?id=com.liamcottle.meshcore.android) | BLE or USB |
| iOS / iPadOS | [App Store → MeshCore](https://apps.apple.com/us/app/meshcore/id6742354151) | BLE |
| Windows / Mac | [files.liamcottle.net/MeshCore](https://files.liamcottle.net/MeshCore) | BLE or USB |
| Browser (Chrome) | [app.meshcore.nz](https://app.meshcore.nz) | USB Serial |

---

## Step 2 — Connect to your radio

=== "Bluetooth (BLE)"
    Use this if you flashed **Companion Radio (BLE)** firmware.

    1. Enable Bluetooth on your phone or computer.
    2. Open the MeshCore app.
    3. Tap **Add Radio** (or the `+` button) and select **Bluetooth**.
    4. The app scans for nearby MeshCore devices. Tap your device when it
       appears.
    5. Enter the pairing code when prompted: **`123456`** (factory default).
    6. The app connects and shows your node identity.

    !!! note "BLE antenna range"
        Some devices (notably the Heltec V3) have a small PCB coil antenna for
        Bluetooth with limited range — a few metres. Keep your phone near the
        radio during pairing. If Bluetooth drops repeatedly at short range, see
        [Troubleshooting](#troubleshooting) below.

=== "USB Serial"
    Use this if you flashed **Companion Radio (USB)** firmware, or if you want
    to manage the device from a laptop browser.

    1. Connect your device to your computer via USB.
    2. Open [app.meshcore.nz](https://app.meshcore.nz) in Chrome (or open the
       desktop app and choose USB).
    3. Click **Connect** → **USB Serial**.
    4. Select the serial port for your device from the browser's port picker.
    5. The app connects.

    !!! warning "Data cable required"
        Some USB cables are charge-only. If the port does not appear in the
        picker, try a different cable.

=== "Wi-Fi"
    Wi-Fi companion firmware requires you to compile the firmware yourself with
    your SSID and password baked in (edit `WIFI_SSID` and `WIFI_PWD` in the
    variant's `platformio.ini`). This is an advanced option; for getting started,
    use BLE or USB instead.

---

## Step 3 — Set your region and frequency

!!! warning "Do this before transmitting"
    A freshly flashed device may not be set to the correct frequency for your
    country. Transmitting on the wrong frequency may be illegal in your region.

After connecting:

1. In the app, open **Settings** (gear icon or side menu).
2. Find **Radio Settings** or **Frequency**.
3. Choose the **preset for your region** — for example *USA/Canada (Recommended)*,
   *EU 868*, or *AU/NZ 915*.
4. Save / apply.

The preset configures frequency, spreading factor (SF), bandwidth (BW), and
coding rate (CR) in one step. You do not need to set these manually.

!!! tip "Ask your community"
    If other MeshCore users in your area use a specific sub-preset (e.g. a
    particular channel number or narrow-band setting), ask on the
    [MeshCore Discord](https://meshcore.gg) so your radios are compatible.

---

## Step 4 — Set your node name

1. Go to **Identity** or **My Node** settings in the app.
2. Enter a name or callsign — this is what others will see when you advertise
   yourself on the mesh.
3. Save.

---

## Troubleshooting

**Device doesn't appear in the Bluetooth scan**

- Confirm you flashed *BLE* Companion firmware, not USB-only firmware.
- Reboot the device (unplug power, replug).
- On Android: ensure the MeshCore app has **Location** permission (Android
  requires location access for BLE scanning, even if you don't use GPS).
- On iOS: ensure the app has **Bluetooth** permission.

**Paired but won't connect / keeps disconnecting**

Go to your phone's system Bluetooth settings and **Forget** the device, then
re-pair from within the MeshCore app (not from the system settings).

**Heltec V3 drops Bluetooth constantly**

The Heltec V3 has a tiny PCB coil antenna that limits Bluetooth range to a few
metres. Replacing it with a 31 mm wire antenna significantly improves stability.
This is a hardware modification — search the MeshCore Discord for guides.

**USB port not listed in browser**

Ensure you are using Chrome or a Chromium-based browser. Check that the cable
has data lines (try a different cable). On Linux, check serial port permissions
(see [Flash troubleshooting](flash-your-first-device.md#troubleshooting)).

---

!!! tip "Next step"
    Connected and frequency set? [Send your first message →](send-your-first-message.md)
