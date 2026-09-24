To use the ESPHome Samsung HVAC Bus component, you will need specific hardware
to connect your ESP device to the communication bus of your Samsung AC unit.
The connection depends on whether your system uses COM1 or COM2.

## Required Hardware

### ESP Device

It is recommended to use an **ESP32** for better performance, as it offers more
CPU power and RAM compared to an ESP8266. The ESP device acts as the controller
for communication between your Samsung HVAC system and Home Assistant.

### RS-485 to TTL Adapter

You will need an RS-485 to TTL serial adapter between the Samsung communication
bus and the ESP device.

The ESP UART TX and RX pins connect to the TTL side of the RS-485 adapter.
They do **not** connect directly to the Samsung F1/F2 or F3/F4 terminals.

### Recommended Setup: M5STACK ATOM Lite with RS-485 Base

For the simplest setup, we recommend using the **M5STACK ATOM Lite** combined
with the **M5STACK RS-485 Base**. This combination is affordable, comes with a
compact case that fits inside most indoor units, and allows direct use of the
12V provided by the V1/V2 lines on some AC units.

- **M5STACK ATOM Lite** - [Aliexpress](https://a.aliexpress.com/_mO88aeK), [M5STACK Store](https://shop.m5stack.com/products/atom-lite-esp32-development-kit), [Documentation](https://docs.m5stack.com/en/core/ATOM%20Lite)
- **M5STACK ATOM RS-485 Base** - [Aliexpress](https://a.aliexpress.com/_mLhOZQA), [M5STACK Store](https://shop.m5stack.com/products/atomic-rs485-base), [Documentation](https://docs.m5stack.com/en/atom/atomic485)

## Choosing the Communication Bus

NonNASA Samsung systems may use either COM1 or COM2. These use different
terminals and different communication behaviour.

### COM1 - F1/F2

COM1 uses the Samsung F1/F2 communication bus.

Connect:

- **F1 (AC Unit)** → **A (RS-485 adapter)**
- **F2 (AC Unit)** → **B (RS-485 adapter)**

COM1 is the default NonNASA protocol in the ESPHome component.

### COM2 - F3/F4

COM2 uses the Samsung F3/F4 wired-controller communication bus.

Connect:

- **F3 (AC Unit)** → **A (RS-485 adapter)**
- **F4 (AC Unit)** → **B (RS-485 adapter)**

The ESP UART is connected to the TTL side of the RS-485 adapter:

- **ESP TX** → **RS-485 transmit input (DI/TX)**
- **ESP RX** ← **RS-485 receive output (RO/RX)**

For the M5STACK configuration used by the example configuration, the UART pins
are:

```yaml
uart:
  tx_pin: GPIO19
  rx_pin: GPIO22
```

For the M5STACK ATOM Tail485, the example configuration instead uses GPIO26
for TX and GPIO32 for RX.

COM2 must also be explicitly selected in the `samsung_ac:` configuration:

```yaml
samsung_ac:
  non_nasa_bus: com2
```

COM1 remains the default when `non_nasa_bus` is not specified.

For more information about COM2 behaviour, current support and known
limitations, see [COM2 Protocol](COM2-Protocol.md).

## Power Connection

If V1/V2 power is available on your AC unit and is suitable for your hardware,
connect:

- **V1 (AC Unit)** → **DC (M5STACK RS-485 Base)**
- **V2 (AC Unit)** → **GND (M5STACK RS-485 Base)**

> **Note:** Some AC units provide power through the V1/V2 lines, which you can
> use to power your ESP device. If your AC unit does not have these lines, you
> will need to use an external power source.

> **Note:** If your AC unit does not have V1/V2 lines and you are using a
> NASA-based device, you can obtain 12V power internally from the unit's
> connector as demonstrated in
> [this discussion post](https://github.com/lanwin/esphome_samsung_ac/discussions/39#discussioncomment-8383733).

## Existing M5STACK Wiring Diagram

The following diagram shows the existing M5STACK F1/F2 installation:

![M5STACK Wiring Diagram](https://github.com/omerfaruk-aran/esphome_samsung_hvac_bus/assets/32042186/42a6757d-bfcf-4a29-be87-cf1b204e248a)

> **Important:** The diagram above shows a **COM1 F1/F2 installation**. For
> COM2, connect RS-485 A/B to **F3/F4** as described above instead.

## Important Considerations

- **Choose the correct bus:** COM1 uses F1/F2. COM2 uses F3/F4.
- **Use an RS-485 interface:** Do not connect ESP UART TX/RX directly to the
  Samsung communication terminals.
- **Check polarity:** COM1 uses F1 → A and F2 → B. The tested COM2 installation
  uses F3 → A and F4 → B.
- **Secure connections:** Loose connections can cause unreliable communication
  or communication errors.
- **COM2 configuration:** Remember to set `non_nasa_bus: com2` for a COM2
  installation. Without this setting the component defaults to COM1.

For additional wiring tips and information, visit our
[Troubleshooting](Troubleshooting.md) page.