# SLWF-08 HDMI-CEC bridge (ESPHome)

An ESPHome firmware for the [SMLIGHT SLWF-08](https://smlight.tech) HDMI-CEC adapter (ESP8266) that connects your HDMI-CEC bus to Home Assistant.

It does not need to know which device sits on which CEC address. It finds the devices itself, so you do not need new firmware when you add, remove or move a device.

Based on SMLIGHT's `slwf-08-Palakis.yaml` example and the [Palakis/esphome-hdmi-cec](https://github.com/Palakis/esphome-hdmi-cec) component.

## Features

- **Bus scan** on boot, every 10 minutes, and on demand (`CEC scan` button or `cec_scan` action). It polls logical addresses 0..14 and asks each device that answers for its physical address, OSD name, vendor ID and power status.
- **Passive discovery**: a message from an unknown address also adds that device.
- **Power poll** every 30 s for each device found.
- **One text sensor per logical address**, `CEC 0 (TV)` to `CEC 14 (Specific use)`, in the format `<power> | <OSD name> | <physical address> | <vendor ID>`, for example `on | SHIELD | 1.2.0.0 | 00044B`, or `absent`.
- Extra sensors: `Active source`, `Active source device`, `Last CEC message`, `Bridge physical address`, `Bridge logical address`, binary sensors `TV` and `Audio system`.
- Every received CEC message is sent to Home Assistant as the event `esphome.hdmi_cec`.
- The bridge answers requests from other devices itself (physical address, OSD name, power status, CEC version, Feature Abort).
- Buttons: CEC scan, TV on, TV standby, All devices standby, Request active source, Volume up, Volume down, Mute.
- Control any device by **name, logical address or physical address** with the `cec_command` action. This also works for devices behind an AV receiver.

## What you need

- An SMLIGHT SLWF-08 adapter.
- Home Assistant with the ESPHome integration (the firmware also works without it; it has a web page on port 80).
- A way to flash it the first time (see SMLIGHT's documentation for the adapter's connection).

## Install

### Option A: ready-made firmware (no Wi-Fi password inside)

1. Download `slwf-08-cec.factory.bin` from the [latest release](../../releases/latest).
2. Flash it for the first time with the [ESPHome web flasher](https://web.esphome.io) (Chrome or Edge, "Install" > choose file), or with esptool:
   ```
   esptool.py --port <your-port> write_flash 0x0 slwf-08-cec.factory.bin
   ```
3. On first boot the bridge opens an **open** Wi-Fi network called `HDMI-AP`. Connect to it with your phone or laptop; the captive portal opens (or go to `192.168.4.1`). Choose your Wi-Fi network and enter its password.
4. The bridge joins your network as `hdmi-control-palakis-<mac suffix>`. Home Assistant should discover it under Settings > Devices & services > ESPHome.

For later updates, use `slwf-08-cec.ota.bin` (upload it on the bridge's web page, or from the ESPHome dashboard). Wi-Fi settings are kept.

### Option B: build it yourself

```
git clone https://github.com/dllz/slwf-08.git
cd slwf-08
cp secrets.yaml.example secrets.yaml      # then edit: Wi-Fi name, Wi-Fi password, AP password
pip install esphome
esphome run slwf-08-cec.yaml
```

`secrets.yaml` is gitignored. Do not commit it. To build the credential-free variant yourself, use `slwf-08-cec-release.yaml` (it still needs a `secrets.yaml` file to exist, with any values, but they are not compiled in).

## Set the physical address

The SLWF-08 has no DDC (EDID) connection, so it cannot find its own HDMI port. It uses `3.0.0.0` (TV HDMI 3) on first boot. If you plug it into another TV port, set the address once. It is kept in flash.

In Home Assistant, Developer tools > Actions, use the action of your device (see "Action names" below) `set_physical_address` with `address: "2.0.0.0"` for TV HDMI 2. A conflict with another device is shown on the `Bridge physical address` sensor.

The logical address is chosen at boot (the first free one, preferring the value in flash, default 10). Change it with `set_logical_address` (`1`..`14`, not `5`), or `-1` to select again automatically.

## Actions

| Action | Variables | What it does |
|---|---|---|
| `cec_command` | `target`, `command` | Control a device. See below. |
| `cec_scan` | none | Scan the CEC bus now. |
| `hdmi_cec_send` | `cec_destination` (int), `cec_data` (int list) | Raw CEC send. |
| `set_physical_address` | `address` (string, e.g. `"3.0.0.0"`) | Set the bridge physical address. |
| `set_logical_address` | `address` (int) | Set the bridge logical address, `-1` = automatic. |

**`target`**: an OSD name (`SHIELD`, not case sensitive, exact match first then "contains"), a logical address (`4`), a physical address (`1.2.0.0`, also for an AV receiver input that has no CEC), or `TV`, `AUDIO`, `ALL`.

**`command`**:

- `on`: wake the device. For the TV: Text View On + Image View On.
- `standby`: put it in standby (`ALL` = broadcast standby).
- `toggle`: send the Power key, for devices that ignore `on`.
- `select`: wake the TV and switch the whole HDMI path to this device (Set Stream Path).

Example Home Assistant automation action:

```yaml
action: esphome.hdmi_control_palakis_xxxxxx_cec_command   # your device name
data:
  target: "SHIELD"
  command: "select"
```

### Action names

The action name in Home Assistant contains the bridge's name with its MAC suffix, for example `esphome.hdmi_control_palakis_fb09b2_cec_command`. Your suffix is different. Find it under Developer tools > Actions by searching for `cec_command`.

### Events

Each CEC message is fired as `esphome.hdmi_cec`. Use it as an automation trigger, and look at the event data in Developer tools > Events (listen to `esphome.hdmi_cec`).

## Notes

- Fixed by the CEC standard: logical address 0 = TV, logical address 5 = audio system. Only one audio system is permitted.
- Keep `name` and `name_add_mac_suffix` unchanged if you edit the YAML. Then OTA updates, the Home Assistant device and the action names stay the same.
- During an OTA upload the bridge stops all CEC activity, because a CEC send blocks the processor and can break the upload.
- The Home Assistant API and OTA have no password or encryption key in this config. Add `api: encryption:` and `ota: password:` if your network is not trusted.

## Troubleshooting

- **No devices found**: check the physical address (see above) and that CEC is enabled in the TV and other devices (names vary: Anynet+, Bravia Sync, Simplink, ...). Press `CEC scan` and look at the logs.
- **Cannot find the bridge on Wi-Fi**: it falls back to the `HDMI-AP` network if it cannot join your Wi-Fi.
- **Flashing fails**: retry with a lower baud rate, and see SMLIGHT's documentation for how to put the adapter in flash mode.

## License

MIT, see [LICENSE](LICENSE). The ESPHome component [Palakis/esphome-hdmi-cec](https://github.com/Palakis/esphome-hdmi-cec) has its own license.
