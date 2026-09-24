# COM2 Protocol

COM2 is a Samsung NonNASA communication bus used between indoor HVAC units
and wired controllers.

Support for COM2 in this integration has been developed through observation
and testing of a working Samsung installation. The details below describe
behaviour verified on that installation and should not yet be assumed to
apply to every Samsung COM2 system.

## COM2 quick start

### Wiring

COM2 uses the Samsung **F3/F4 communication bus**, rather than the F1/F2 bus
described by the existing NonNASA COM1 installation instructions.

Connect the Samsung F3/F4 communication pair to the RS-485 interface connected
to the ESP device.

> **Important:** F3/F4 RS-485 A/B polarity still needs to be documented here
> from a verified installation. Do not assume that the existing F1/F2 wiring
> convention necessarily applies to F3/F4.

Connect the RS-485 A terminal to Samsung F3 and the RS-485 B terminal to Samsung F4. 
The ESP32 TX and RX GPIOs connect to the transmit and receive pins of the RS-485 interface, not directly to F3/F4.
For the M5STACK configuration used by the example configuration:

```yaml example
uart:
  tx_pin: GPIO19
  rx_pin: GPIO22
  baud_rate: 2400
  parity: EVEN
  stop_bits: 1

```

The exact GPIO pins depend on the ESP32 and RS-485 hardware being used.

### ESPHome configuration

COM1 remains the default NonNASA protocol. A COM2 installation must explicitly
select COM2 by adding `non_nasa_bus: com2` to the existing `samsung_ac:`
configuration:

```yaml
samsung_ac:
  non_nasa_bus: com2
```

Do not create a second `samsung_ac:` block if one already exists.

The remainder of the device configuration, including the indoor-unit address
and climate entities, is configured in the normal way.

After flashing, monitor the ESPHome logs. The integration must receive a
complete CMD52 state from the indoor unit before it will transmit a COM2
control request.

## Current status

COM2 v1 supports:

- Reading indoor-unit climate state
- Power control
- HVAC mode control
- Target temperature control
- Fan speed control
- Home Assistant climate integration
- Operation alongside the existing Samsung wired controllers

The implementation has been tested with two wired controllers remaining
connected and operational.

Swing and alternative mode control are not currently implemented for COM2.

## Observed bus topology

On the system used for development:

| Address | Device |
|---|---|
| `20` | Indoor unit |
| `84` | Wired controller | Master Controller
| `85` | Wired controller / address used by ESPHome for control | Slave controller

Other installations may use different addresses. 

These addresses are observations from the development installation and should
not yet be treated as universal Samsung COM2 addresses.

## State discovery

For COM2, CMD52 messages from the indoor unit are treated as the authoritative
source of climate state.

The implementation currently obtains the following from CMD52:

- Power
- HVAC mode
- Target temperature
- Fan speed

CMD20 and CMD53 are not used as authoritative COM2 climate state.

Because an A0 control packet contains a complete control state, the integration
waits until a complete CMD52 state has been received before allowing control.

When Home Assistant changes a single property, such as temperature, the
integration starts with the most recently received complete CMD52 state and
changes only the requested property before constructing the A0 packet.

This prevents a partial Home Assistant command from unintentionally replacing
other climate settings with unknown or default values.

## Control packets

Climate control is performed using A0 packets.

Rather than transmitting a packet immediately when Home Assistant requests a
change, the request is queued until an appropriate position in the existing
COM2 bus traffic is observed.

On the development system, reliable operation was achieved by:

1. Observing an `85 -> 84` C4 packet.
2. Waiting approximately 180 ms.
3. Sending the queued A0 packet using source address `85`.

This places the ESPHome transmission in the bus slot normally associated with
controller 85.

Earlier experiments transmitting as controller 84 caused E607 communication
errors and disruption of the physical controller. The controller-85 timing
method has been tested with both physical controllers connected without
producing those errors.

The controller addresses and 180 ms timing are empirical observations from the
development system. They are not yet known to be universal COM2 requirements.

## Mode constraints

The physical Samsung controller was used to verify the available temperature
and fan settings for each operating mode.

| Mode | Temperature | Low | Mid | High | Turbo |
|---|---|---:|---:|---:|---:|
| Auto | 18-30 C | No | No | No | Yes |
| Cool | 18-30 C | Yes | Yes | Yes | Yes |
| Dry | 18-30 C | No | No | No | Yes |
| Fan | N/A | Yes | Yes | Yes | No |
| Heat | 16-30 C | Yes | Yes | Yes | Yes |

These constraints are enforced before a COM2 control packet is queued.

For example:

- Heat at 16 C -> Cool resolves to Cool at 18 C.
- Cool with Turbo fan -> Fan mode resolves to High fan.
- A manual fan setting carried into Auto or Dry resolves to Turbo.

This prevents Home Assistant from transmitting combinations that the physical
Samsung controller does not permit.

## Control request handling

A Home Assistant request does not directly become an A0 packet.

The current process is:

1. Start with the most recent complete CMD52 state.
2. Apply only the fields explicitly changed by Home Assistant.
3. Resolve the resulting state against the destination mode's capabilities.
4. Validate the complete state.
5. Queue the A0 request.
6. Wait for the controller-85 transmission opportunity.
7. Transmit the A0 packet using source address 85.

The COM2 encoder is therefore intended to receive a complete, valid control
state rather than deciding what values are safe to transmit.

## Fan modes

The COM2 fan values observed in CMD52 correspond to:

| COM2 value | Fan mode |
|---|---|
| `1` | Low |
| `2` | Mid |
| `3` | High |
| `4` | Turbo |

A separate COM2 `Auto` fan wire value has not been observed.

Auto and Dry operating modes instead restrict the selectable fan setting to
the value represented by Turbo in the protocol.

The integration therefore does not treat Home Assistant `FanMode::Auto` as a
valid COM2 wire fan state.

Earlier versions of the COM2 implementation could manufacture an Auto fan
request when changing HVAC mode. This behaviour has been removed.

## Safety resolver

COM2 v1 contains a mode capability model describing:

- Whether temperature is meaningful for the mode
- Minimum target temperature
- Maximum target temperature
- Available fan speeds

Before an A0 packet is queued, the complete requested state is passed through
a resolver.

If a value inherited from the previous operating mode is not legal in the new
mode, the resolver substitutes a safe supported value.

The resolved request is then validated again before it can enter the
transmission queue.

This is particularly important because Samsung's permitted settings differ
between operating modes. A state that is valid in Heat, for example, is not
necessarily valid after changing to Cool or Fan.

## v1 validation

The initial COM2 implementation has been tested with both original Samsung
wired controllers connected.

Successful control tests include:

- Target temperature changes
- Fan speed changes
- Cool -> Heat
- Fan -> Cool
- Heat at 16 C -> Cool, resolving safely to 18 C
- Cool with Turbo fan -> Fan, resolving safely to High

These tests completed without the E607 communication errors seen during
earlier controller-84 transmission experiments.

The physical wired controllers continued to operate normally during these
tests.

## Known limitations and TODO

COM2 v1 is intentionally a working first implementation rather than a complete
reverse engineering of the protocol.

### Home Assistant state display

Some mode changes can cause a temporary Home Assistant state/display "bobble"
while subsequent state packets arrive.

This is particularly noticeable during Fan -> Cool.

The physical HVAC state and controllers remain correct, so this is currently
considered a display/state-publishing issue rather than a COM2 transmission
failure.

### Per-mode state memory

Samsung's physical controller remembers target temperature and fan settings
independently for different operating modes.

COM2 v1 does not yet maintain an equivalent per-mode cache.

A future implementation should remember the last observed temperature and fan
setting for each mode and use those values when returning to that mode.

The current safety resolver remains necessary even after per-mode memory is
implemented.

### Mode constraints in YAML

The current COM2 capability matrix is defined in code using the behaviour
verified on the development system.

A future version should allow mode constraints to be overridden from ESPHome
YAML while retaining safe defaults.

This will make it possible to support Samsung systems whose available modes or
limits differ from the development installation.

### Heat 16 C support

Heat mode has been verified on the physical controller to support a target
temperature of 16 C.

There are still older global 18 C assumptions elsewhere in the integration,
including the COM2 encoder and Home Assistant climate temperature limits.

As a result, full direct 16 C Heat control is not yet exposed correctly even
though the COM2 capability model records Heat's actual 16-30 C range.

These global assumptions should eventually be replaced with mode-aware
behaviour.

### Encoder fan fallback

The COM2 encoder still contains legacy fallback behaviour for unsupported fan
values.

The safety resolver and validator are intended to prevent an invalid fan mode
from reaching the encoder, but the encoder should eventually reject invalid
input rather than silently translating it to another fan speed.

### Bus timing and addressing

The current transmission strategy is based on observations from one
installation:

- Indoor unit at address 20
- Physical controllers at addresses 84 and 85
- ESPHome transmitting as controller 85
- `85 -> 84` C4 used to identify the transmission opportunity
- Approximately 180 ms delay before transmission

This configuration is stable on the development installation, but more COM2
systems need to be observed before these values can be considered generally
applicable.

Future work should determine whether controller addressing and transmission
timing can be discovered dynamically.

### Additional protocol reverse engineering

Further work may include:

- Additional COM2 commands and packet fields
- Swing control
- Alternative/special operating modes
- Controller discovery
- More robust bus-slot discovery
- Testing with other Samsung indoor units and controller combinations

## Implementation status

COM2 v1 should be considered an experimentally validated implementation.

The primary climate-control path is functional and has been exercised with the
original wired controllers connected. Safety handling for the mode constraints
observed on the development system is implemented.

The remaining work is primarily compatibility, fidelity, Home Assistant
presentation, configuration flexibility, and broader hardware validation,
rather than basic climate control.