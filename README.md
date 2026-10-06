[![hacs_badge](https://img.shields.io/badge/HACS-Default-orange.svg?style=for-the-badge)](https://github.com/hacs/default)

# Jablotron 100+

Home Assistant custom component for JABLOTRON 100+ alarm system.

Tested with JA-100K, JA-101K, JA-101K-LAN, JA-103K, JA-103KRY, JA-106K-3G, JA-107K and JA-14K.

## Temporary diagnostic build: issue 174

**Version `3.34.1.dev1742` is the second capture-only build based on 3.34.1, not a fix or a production release.**
It investigates [kukulich/home-assistant-jablotron100#174](https://github.com/kukulich/home-assistant-jablotron100/issues/174).

The [physical v1 test](https://github.com/kukulich/home-assistant-jablotron100/issues/174#issuecomment-6004918883) strongly supports interpreting the two request parameters as **start and inclusive end**, rather than start and count: `(3, 4)` returned one data byte for two positions, not four. The unanswered high-position v1 requests had their second parameter below their start under this interpretation; they do not establish that high-position queries are unsupported. V2 tests ordered intervals instead. The high-position replies remain unverified on hardware.

When the highest configured, non-ignored device position exceeds 122, startup runs a bounded section-map probe instead of normal device discovery. It does this even if the device cache is already complete, without clearing or updating that cache. **Setup deliberately ends with `ServiceUnavailable`; entities and automations depending on this integration will be unavailable during the test.** Do not leave this build installed. Configurations with no non-ignored positions above 122 follow the normal startup path.

For the reported highest position 219, the probe sends the existing authorisation command followed by these seven `0x3a` read requests, once each:

| Window | Intended inclusive interval | Request packet | Expected reply bytes |
| --- | --- | --- | --- |
| 1 | 1-122 | `3a02017a` | 64 |
| 2 | 3-6 | `3a020306` | 5 |
| 3 | 121-124 | `3a02797c` | 5 |
| 4 | 123-219 | `3a027bdb` | 52 |
| 5 | 199-200 | `3a02c7c8` | 4 |
| 6 | 213-214 | `3a02d5d6` | 4 |
| 7 | 219-219 | `3a02dbdb` | 4 |

Expected lengths assume the inclusive-end interpretation; actual replies are logged even when their length differs. Every interval is ordered, starts at an odd position and spans at most 122 positions so the expected reply fits in 64 bytes. Window 2 further checks the end-position interpretation, window 3 crosses the old 122-position boundary, and windows 4-7 target the missing map and known F-Link references. Replies are recorded, not interpreted as section assignments. The probe waits two seconds after each request, records all observed `0x3b` packets (including malformed ones), and reports reply counts even if no map arrives. Window labels describe timing, not proven request/reply correlation. There are no probe retries within a startup, no arm/disarm or PG commands, and no changes to panel configuration. Stopping Home Assistant interrupts the capture.

### One-off test and rollback

1. Back up the currently installed `custom_components/jablotron100` directory outside `custom_components`. Keep the existing integration configuration and cache; do not remove/re-add the integration, renumber devices or mark occupied positions as Empty.
2. Choose a short maintenance window in which Home Assistant alarm entities and related automations may be unavailable. Keep the alarm's normal keypad/application available.
3. Download the diagnostic branch/source ZIP linked in the issue comment. Stop Home Assistant Core, replace only `custom_components/jablotron100` with that directory from the ZIP, and start Home Assistant Core again. Do not replace the rest of your configuration.
4. Capture the startup log through **`Section-map probe v2 capture finished`**, normally about 15 seconds after the probe starts for position 219. Confirm that the log says **v2**, not v1. The final setup error is intentional. An I/O error is reported as **`Section-map probe v2 failed`** instead. If neither marker appears, stop the test and share the available startup log rather than repeatedly restarting.
5. All probe messages are at warning level and include the exact map requests/replies. The default Home Assistant logging level is sufficient; no broad packet logging is required. Remove any custom logging filter that suppresses warnings from `custom_components.jablotron100` for the test. Review the log for access codes, serial numbers and other identifying information before sharing; no configuration/storage files or 3.33.5 packet logs are needed.
6. Stop Home Assistant Core, restore the backed-up integration directory, and start Home Assistant Core. Alternatively, redownload a known-working regular version through HACS and restart. For the reporter, 3.33.5 is the confirmed working rollback; 3.34.1 only improves diagnostics and is not a fix for this issue. Confirm that the expected entities and automations work again, and restore your usual logging settings.

Return the sanitized probe log to the issue. Compare any future fix against the supplied F-Link references (200: section 1; 214 and 219: section 2); a successful startup alone will not establish correctness.


## Features

### Sections

- States are reported to Home Assistant.
- You can arm/disarm all sections. Supported states are `arm_away` (= armed) and `arm_night`/`arm_home` (choose in options what means "armed partially" for you).
- Event `jablotron100_wrong_code` is triggered when wrong code is inserted in Home Assistant.
- Problem in a section is reported in specific "problem" sensor.

### Devices

- Devices with two states (on/off, active/inactive, open/closed etc.) are supported.
- Sabotage or problem of the device is supported in specific "problem" sensor.
- Temperature is reported for thermostats, thermometers and smoke detectors.
- Pulses are reported for electricity meters with pulse output.
- Signal strength is reported for wireless devices.
- Battery level is reported for devices with battery.
- Model, hardware and firmware versions are shown in device information when reported by a device.

### PG outputs

- States are reported to Home Assistant.
- It's possible to turn on/off all PG outputs.

### Central unit

- Power supply state and overall problem are reported as binary sensors.
- BUS voltage and BUS devices current are reported per detected BUS.
- Battery presence, battery level, standby and load voltages are reported when the central unit has a backup battery.
- LAN connection state and (when available) the LAN IP address are reported for supported central units.
- GSM signal availability and signal strength are reported for supported central units.
- A `wrong code` event entity records failed authorisation attempts; the same condition also fires the `jablotron100_wrong_code` event on the bus.


## Before installation

Requires Home Assistant 2026.9.1 or newer.

1. Connect the USB cable to Jablotron central unit
2. Restart the Home Assistant OS

## Installation

- If you use code with a prefix, insert the code with the asterisk, e.g. `12*3456`.
- Use code of administrator to make devices work. If you cannot use code of administrator, or you don't want to use devices, set the number of devices to 0.
- You have to set devices in the same order as you see them in your J-Link/F-Link/mobile application. Ignore the central unit on position 0. The number of devices is the highest position to include, not the number of physical devices in use. Set unoccupied positions to **Empty** without renumbering the other devices.
- If you want to use PG outputs, the user of the code has to have rights to control the PG outputs. Set the number of PG outputs to 0 to ignore them.


Serial port should be automatically detected. If not, you can detect it manually and set it during integration installation.

The default value `auto` makes the integration probe `/sys/class/hidraw` on every start and pick the device exposing the Jablotron USB vendor/product ID (`16D6:0008`). Prefer `auto` over a fixed `/dev/hidrawN` path — when other USB HID peripherals are connected to the host, the kernel's hidraw numbering can change between reboots.

```
$ dmesg | grep usb
$ dmesg | grep hid
```

The cable should be connected as `/dev/hidraw[x]`, `/dev/ttyUSB0` or similar.


### HACS

1. Install the integration via [HACS](https://hacs.xyz/) (Home Assistant Community Store)  
    <small>*HACS is a third party community store and is not included in Home Assistant out of the box.*</small>
2. Restart Home Assistant
3. Jablotron integration should be available in the integrations UI

### Manual

1. [Download integration](https://github.com/kukulich/home-assistant-jablotron100/releases/)
2. Copy the folder `custom_components/jablotron100` from the zip to your config directory
3. Restart Home Assistant
4. Jablotron integration should be available in the integrations UI


### Reconfigure

To change the serial port, code, number of devices or PG outputs without losing your existing configuration, open the integration on the *Devices & Services* page and choose *Reconfigure*. Leave the password field empty to keep the previously stored code. Per-device type assignments are preserved across reconfiguration.

If device discovery times out, the error reports missing status replies and the received section-map coverage. If the map does not cover some configured positions, compare those positions with F-Link/J-Link and change only positions that are actually unoccupied to **Empty** using *Reconfigure*. The total number of positions can remain unchanged. There is no need to remove the integration or clear its cache to correct a position assignment.


## Check

1. Try to arm/disarm all sections
2. Try to activate all devices if possible (open/close door/window, move ahead of motion sensor etc.) and check if Home Assistant see the state changes
3. Check log - it should be empty when everything works
4. Does any problem occur? Report [issue](https://github.com/kukulich/home-assistant-jablotron100/issues) or join [Discord](https://discord.gg/bNmaB6n)


## Services

### `jablotron100.reset_problem`

Locally turns the selected `problem` binary sensor back to off without waiting for the central unit to clear the underlying condition. Useful for one-shot conditions that you have already acknowledged. Only entities with `device_class: problem` from this integration are accepted.

```yaml
service: jablotron100.reset_problem
target:
  entity_id: binary_sensor.section_1_problem_sensor
```

Even if everything works for you, you can join the [Discord](https://discord.gg/bNmaB6n).
We would be happy:
 - If you report model of you Jablotron central unit, so we know that integration works on another model
 - If you can test some things (e.g. LAN), so we can make the integration more robust

The communication in Discord is mostly in Czech or Slovak but don't be afraid - you can use English as well.


## Debugging
1. Enable debug logging for the Jablotron intergation via the [logger](https://www.home-assistant.io/integrations/logger/) integration by adding the following lines to the `configuration.yaml` file.
```
logger:
  default: info
  logs: 
    custom_components.jablotron100: debug
```

2. Enable debug logging in the Jablotron integration. Go to the Integration page of your Home Assistant and click on the `Configure` button belonging to the Jablotron integration and then select `Debugging` to specify specific debugging options, such as `Log all incoming packets`. Finish the configuration by pressing the `Submit` button.
3. After enabling 1. and 2., the home assistant log should contain debug log of the Jablotron integration, e.g.,
```
2022-02-17 10:57:19 DEBUG (ThreadPoolExecutor-2_0) [custom_components.jablotron100] Incoming: 801a0cffffffff010001002820010027ffffffffffffffffffffffff
2022-02-17 10:57:19 DEBUG (ThreadPoolExecutor-2_0) [custom_components.jablotron100] Incoming: 5203820113
```
4. Restart Home Assistant

## Development

See [Testing](tests/README.md) for packet tests, tests with real Home Assistant,
and the minimum/latest Home Assistant CI matrix.

## Credits

Big thanks to [plaksnor](https://github.com/plaksnor/), [Horsi70](https://github.com/Horsi70/) and [Shamshala](https://github.com/Shamshala/) for their work on previous integration.
