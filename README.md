# Mitsubishi CN105 ESPHome airconditioning guide

## 1. Update Home Assistant, ESPHome and your add-ons

Before starting, make sure **Home Assistant, the ESPHome Device Builder add-on, and your other installed add-ons are fully updated to the latest stable versions**. Let the updates finish and complete any requested restarts before continuing. Use the updated Device Builder to compile and flash the controller.

## Why this guide exists

This guide helps you take back control of the technology you own. It connects a compatible Mitsubishi Electric airconditioning unit to Home Assistant through an ESP32-S3 and the unit's CN105 connector. Everyday commands and readings stay local, without depending on a manufacturer's cloud service to process them.

The configuration uses the external [MitsubishiCN105ESPHome component](https://github.com/echavet/MitsubishiCN105ESPHome). Internet access is needed to download software and updates during setup and maintenance.

## Safety, responsibility and warranty

This is a DIY project undertaken at your own risk. Working inside an airconditioning unit can expose you to dangerous voltages. Incorrect wiring or installation can cause electric shock, fire, serious injury, death or damage to your home and equipment. I accept no responsibility for injury, loss or damage arising from attempting this project.

Opening or modifying your unit may void its manufacturer's warranty. Check the warranty terms before proceeding. Isolate the electrical supply before accessing the electronics, and use a qualified technician for work involving exposed mains wiring. Switching the unit off with its remote does not isolate the electrical supply.

With control and freedom comes responsibility. You are responsible for future maintenance, especially security updates and the security of your Home Assistant installation and network. Encryption helps protect communication, but I do not have a crystal ball. No connected device can be guaranteed secure forever. Future vulnerabilities may require you to update, reconfigure or replace parts of this setup.

Treat the controller as an IoT device and follow current [IoT security best practices](https://esphome.io/guides/security_best_practices/).

## What you need

- Soldering iron, flux, solder
- Home Assistant with ESPHome Device Builder.
- An ESP32-S3 board that matches the configuration, such as the board shown below.
- A compatible Mitsubishi Electric indoor unit with a CN105 connector.
- A five-position JST PA harness, commonly sold as a PAP-05V-S cable.
- A UART interface with signal levels compatible with the ESP32-S3.
- A USB data cable and access to your Wi-Fi network.

![Example ESP32-S3 development board](images/esp32-s3-board.png)

The [JST PA family](https://www.jst-mfg.com/product/index.php?series=189) uses a 2 mm pitch. Check the exact connector and board before buying parts, as similar-looking connectors are not always interchangeable.

## 2. Add the device configuration

Download [Mitsibishi esp home.yaml](Mitsibishi%20esp%20home.yaml). Open ESPHome Device Builder, create a device and paste the configuration into its YAML editor. You can also use **Import from File** if your version offers it.

For an existing controller, edit its existing Device Builder entry. Keep its current device name, friendly name, Wi-Fi details and API key while updating the rest of the configuration.

For a new controller, change the substitutions to suit its room. For example:

```yaml
substitutions:
  device_name: living-room-ac
  friendly_name: Living Room Airconditioning
```

Use lowercase letters and hyphens for `device_name`, and give each controller a different name. The friendly name is the readable label shown in Home Assistant.

![ESPHome Device Builder showing the ESP32-S3 board and YAML editor](images/esphome-device-builder.png)

The supplied YAML selects `esp32-s3-devkitc-1` with the ESP-IDF framework. Confirm that this is suitable for your board. ESPHome downloads the external CN105 component as part of the build when needed. You do not need to upload the component's source files separately.

## 3. Set up secrets and encryption

### How secret references work

The device configuration reads private values from ESPHome's local `secrets.yaml` file. For example:

```yaml
key: !secret homeassistant_api_encryption
```

means "look up the entry named `homeassistant_api_encryption` and use its value." ESPHome resolves this when it processes the configuration. It is a local file lookup, not a call to an online secrets service.

Open **Secrets** in ESPHome Device Builder and add these entries, or keep the existing values for a controller that already works:

```yaml
wifi_ssid: "YOUR_WIFI_NAME"
wifi_password: "YOUR_WIFI_PASSWORD"
homeassistant_api_encryption: "YOUR_BASE64_API_ENCRYPTION_KEY"
```

Replace the placeholders before compiling. The secret names must match the references in the device YAML exactly.

| Secret | Used for |
|---|---|
| `wifi_ssid` | Your Wi-Fi network name |
| `wifi_password` | Your Wi-Fi password |
| `homeassistant_api_encryption` | The API key, also used by encrypted OTA in this configuration |

Keep `secrets.yaml` and compiled firmware binaries private. The secrets file is plain text, and its values are included in the firmware. Share only the device YAML, documentation and intended images. See [ESPHome's secrets guidance](https://esphome.io/guides/security_best_practices/#using-secretsyaml).

### Generate an API key for a new controller

The API key contains 32 random bytes encoded as Base64. On a Mac, generate one with:

```bash
openssl rand -base64 32
```

Copy the entire output into the `homeassistant_api_encryption` value, inside quotation marks. When Home Assistant asks for the encryption key, enter that value, not the secret's name.

Keep the current key when updating an existing working controller. Each new controller should have its own key. If multiple controllers share one secrets file, use a different secret name for each key and update that controller's `!secret` reference to match. See the [ESPHome API documentation](https://esphome.io/components/api/#configuration-variables).

### API and OTA encryption

The API carries sensor readings, operating states and commands between Home Assistant and the ESP32-S3. OTA means "over the air" and is the wireless firmware update connection.

The supplied YAML uses the encrypted configuration shown in this screenshot:

![API secret reference and encrypted OTA configuration](images/api-and-ota-encryption.png)

```yaml
api:
  encryption:
    key: !secret homeassistant_api_encryption
  reboot_timeout: 0s

ota:
  - platform: esphome
    encryption:
```

The API uses the saved key to protect its network connection. With supported ESPHome versions, the OTA block inherits the same key, encrypts firmware uploads and authenticates the uploader. The empty `encryption:` line is intentional. This configuration does not need a separate `ota_password` secret. See the [ESPHome OTA documentation](https://esphome.io/components/ota/esphome/#encryption).

`api.reboot_timeout: 0s` disables restarting because no API client connects. It does not disable other recovery settings, such as the Wi-Fi reboot timeout.

These settings protect network communication. They do not encrypt the physical UART connection to CN105.

**Updating a controller that still uses an OTA password:** follow [ESPHome's migration instructions](https://esphome.io/components/ota/esphome/#enabling-encryption-on-an-existing-device) before replacing its OTA settings. Older firmware may reject an encrypted upload, and password-based firmware still needs its current password during the transition. A first installation over USB can use the encrypted configuration from the start.

## 4. Compile and flash over USB

For a new controller, do the first flash before connecting it to the airconditioning unit.

1. Leave the CN105 harness disconnected from the ESP32-S3.
2. Connect the board to your computer with a USB data cable.
3. Save the YAML and use **Validate** to check the configuration.
4. Select **Install** and choose the USB installation method offered by Device Builder. If needed, follow the [ESPHome physical connection instructions](https://esphome.io/guides/physical_device_connection/).
5. Wait for compilation and flashing to finish. The first build can take longer while ESPHome downloads dependencies.
6. Check the logs for a successful Wi-Fi connection, then disconnect USB before installing the controller in the unit.

Saving the YAML alone does not change the device. You must compile and install it. Later updates can use **Install** and the wireless option once the controller is online and its OTA settings match the running firmware. See [ESPHome's setup guide](https://esphome.io/install/getting-started/).

## 5. Find CN105 and connect the controller

Isolate the airconditioning unit's electrical supply before opening it. These photos show one example unit. Cover arrangements and board locations vary, so check your model's service documentation.

### Find the electronics compartment

In this example, the electronics compartment is on the right-hand side of the indoor unit.

![Example indoor unit with the right-hand electronics compartment visible](images/indoor-unit-electronics-location.png)

The wider view below shows the green control board beneath the other electronics. Avoid disturbing the existing wiring.

![Wider view of the indoor unit electronics and green control board](images/indoor-unit-control-board.png)

### Identify the CN105 connector

CN105 is the accessory communication connector used to exchange commands and readings with the indoor unit. In this photo, it is the red five-position socket beside the CN105 marking. Confirm the printed label on your own board. Colour alone is not enough to identify it.

![Close-up of the red CN105 socket and the adjacent board label](images/cn105-board-connector.png)

### Check the electrical interface

A PAP harness is a connector and cable. It does not provide voltage conversion. The ESP32-S3 uses 3.3 V GPIO and is not specified as 5 V tolerant. The upstream project describes a level-shifted CN105 interface.

Use a suitable UART level shifter unless compatible signal levels or built-in conversion have been verified for your exact hardware. A working connection by itself does not establish that a board contains a converter. Check the [upstream wiring guide](https://github.com/echavet/MitsubishiCN105ESPHome#step-1-building-the-control-circuit) and [Espressif's electrical specifications, Table 5-4](https://www.espressif.com/sites/default/files/documentation/esp32-s3_datasheet_en.pdf#page=65).

### Use GPIO17 and GPIO16

This configuration assigns **GPIO17 to transmit** and **GPIO16 to receive**. Use the numbered headers shown in the pinout. On the board pictured, the headers printed TX and RX are different pins.

![ESP32-S3 header pinout showing GPIO16, GPIO17, 5V and GND](images/esp32-s3-pinout.png)

### Check the PAP harness wiring

The reference image labels the Mitsubishi-side signals. Cable colours vary, and viewing the connector from the opposite side reverses its apparent left-to-right order. Verify the contact positions using the unit's documentation before connecting the harness.

![CN105 connector reference showing RX, TX, 5V and GND](images/cn105-connector-reference.png)

![Example five-position PAP harness; wire colours may differ](images/pap-harness-example.png)

| Mitsubishi CN105 signal | ESP32-S3 connection |
|---|---|
| TX from the indoor unit | GPIO16, configured as RX, through any required voltage conversion |
| RX into the indoor unit | GPIO17, configured as TX, through any required voltage conversion |
| GND | Common GND for the board and interface |
| Verified +5 V supply | The board's documented 5 V input, if the supply can power the board reliably |
| Unused contact | Leave disconnected and insulated; it may carry 12 V on some units |

Keep 12 V away from the board's 5 V input and GPIO. Use only one board power source at a time unless the board explicitly supports combining them. The CN105 5 V supply has limited current on some units, so confirm that it can support your board and interface.

The UART settings must match the wiring:

```yaml
uart:
  id: ac_uart
  tx_pin: GPIO17
  rx_pin: GPIO16
  baud_rate: 2400
  data_bits: 8
  parity: EVEN
  stop_bits: 1
```

### Fit the harness and secure the controller

Once the wiring has been checked and USB is disconnected, align the harness's keyed housing and seat it gently in CN105. Do not force it. Keep the electrical supply isolated while making the connection.

![Example PAP harness being fitted to the CN105 socket](images/cn105-harness-connected.png)

Insulate exposed connections and secure the board and cable away from moving parts, wet surfaces and exposed electrical connections. Refit the covers before restoring power.

Open the ESPHome logs and check for both a network connection and a successful reply from the indoor unit. An ESP32-S3 being online does not, by itself, confirm CN105 communication.

## 6. Add the controller to Home Assistant

Open **Settings > Devices & services** and look for the discovered ESPHome device. Add it and enter its API encryption key if requested. If discovery does not find it, add the ESPHome integration manually using the controller's IP address.

Use the entity IDs actually created in your installation when building dashboard cards or automations. If older entries still exist, a replacement entity may receive an `_2` suffix.

## Controls and readings

The configuration creates the following entities. Available controls and valid readings depend on the indoor unit. A configured sensor may remain unknown or report an unsupported value if the unit does not supply that information.

| Entity | What it shows or controls |
|---|---|
| Air Conditioner | Supported operating modes, target temperature, fan speed and swing settings |
| Vertical Vane | Supported vertical vane positions and swing |
| Horizontal Vane | Supported horizontal vane positions and swing |
| Room Temperature | The climate entity's reported indoor temperature |
| Target Temperature | The climate entity's target temperature |
| Outside Air Temperature | Outdoor temperature reported by supported units |
| Compressor Frequency | Reported compressor frequency |
| Input Power | Electrical power reported by the unit |
| Energy Usage | The unit's reported energy counter |
| Runtime Hours | The unit's reported operating hours |
| Operating Stage | Reported indoor fan stage |
| Sub Mode | Additional operating state, such as standby or defrost |
| Auto Sub Mode | The reported operating state within automatic mode |
| I-See Status | Reported I-See status, not a separate occupancy reading |
| Controller Connection | ESPHome API connectivity, not confirmation of a CN105 connection |
| Controller Uptime | Time since the ESP32-S3 last started |
| Wi-Fi Signal | The controller's Wi-Fi signal strength |
| Firmware Version | The ESPHome firmware version |
| Restart Controller | Manual restart button, disabled by default in Home Assistant |
| Restart in Safe Mode | Recovery button, disabled by default in Home Assistant |

Room Temperature and Target Temperature copy values from the climate entity. They do not add physical temperature sensors. The connection sensor tracks the [ESPHome API connection](https://esphome.io/components/binary_sensor/status/). See the [upstream project](https://github.com/echavet/MitsubishiCN105ESPHome) for model-specific features and limitations.

There is no scheduled weekly restart in this configuration. Wi-Fi and safe-mode recovery settings still apply.

## Example Home Assistant dashboard

![Example Home Assistant airconditioning dashboard with device names blacked out](images/home-assistant-dashboard-redacted.png)

This example shows thermostat controls, individual readings and controller diagnostics. Device names have been blacked out for privacy.

The schedule panels shown at the bottom are a separate Home Assistant feature. This ESPHome YAML provides the airconditioning entities but does not create those panels or a scheduler. They require their own Home Assistant configuration.

## Troubleshooting

| Problem | What to check |
|---|---|
| Secret not defined | Check that the entry exists in ESPHome's `secrets.yaml` and matches the `!secret` reference exactly. |
| Controller is online but the unit does not respond | Check the logs for CN105 replies. Isolate power before checking signal wiring, common ground and voltage compatibility. |
| Dashboard entities are unavailable | Find the current live entity IDs in Home Assistant. Check for old entries and replacements with an `_2` suffix. |
| Room temperature is unavailable | Check `current_temperature` on the climate entity. The separate Room Temperature sensor copies that value. |
| Some readings are missing | Check whether the entity is enabled and whether the indoor unit supports it. |
| OTA upload fails | Check the address, current API key, installed firmware version and whether the device requires password-based or encrypted OTA. |
| The controller repeatedly resets or loses Wi-Fi | Check the board's power supply and the logs. Some CN105 5 V supplies cannot provide enough current for every development board. |

## Ongoing maintenance

Keep Home Assistant, ESPHome and your add-ons updated. Rebuild and install firmware when a relevant fix is released, as updating Device Builder alone does not update firmware already running on the controller.

Keep a private backup of the YAML, secrets and working configuration. Review upstream changes before installing new firmware, and follow [ESPHome's security guidance](https://esphome.io/guides/security_best_practices/) for protecting IoT devices and their network access.

## External inspiration

- **[Video walkthrough](https://www.youtube.com/watch?v=I7jC5Xi063E)**: a recommended visual guide to the build and setup process.
- **[rwarner / ESP-CN105-Config](https://github.com/rwarner/ESP-CN105-Config)**: the example configuration accompanying the video and an inspiration for this guide.
- **[echavet / MitsubishiCN105ESPHome](https://github.com/echavet/MitsubishiCN105ESPHome)**: the main project behind the CN105 component used here, with further information on features, wiring and troubleshooting.
