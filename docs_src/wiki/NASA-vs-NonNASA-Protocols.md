Samsung utilizes two distinct protocols for communication between its HVAC systems: **NASA** and **NonNASA**. These protocols dictate how information is exchanged between indoor and outdoor units. Understanding these protocols can help with troubleshooting and configuring your setup correctly.

## NASA Protocol

The **NASA protocol** is the newer communication standard Samsung uses in its modern HVAC systems. It is designed to be more flexible and capable of transporting more data compared to the older NonNASA protocol. It uses keys and values for each piece of data, allowing a more structured and detailed way to exchange information between units.

For instance, to retrieve room temperature data, you would need to know the specific key and wait for it to be transmitted. This protocol allows more advanced communication between units and supports a wider range of data types such as enums, integers, long values, and byte arrays.

For more technical details, check out the documentation by [Foxhill67](https://wiki.myehs.eu/wiki/NASA_Protocol) which offers an in-depth breakdown of this protocol.

### Key Characteristics:
- **Advanced Key-Value Pairing:** Transports variables using a unique key and associated value.
- **Supports Multiple Data Types:** Allows for the use of Enums, Integers, Longs, and Bytes.
- **More Data Capacity:** Transfers larger and more detailed sets of data.

## NonNASA Protocol

The **NonNASA protocol** is the older standard that Samsung HVAC systems used. It is more basic and primarily designed to transport essential data between the air conditioner units. Although simpler, it still shares some structural characteristics with the NASA protocol, like the usage of start and end bytes for each message.

If you're looking for a more detailed explanation of the NonNASA protocol, take a look at the efforts made by [DannyDeGaspari](https://github.com/DannyDeGaspari/Samsung-HVAC-buscontrol), who has documented the protocol from the wall controller's perspective.

### Key Characteristics:
- **Simple Data Transport:** Designed for straightforward communication of basic air conditioner data.
- **Fewer Data Types Supported:** Primarily supports basic values like temperatures, modes, and error codes.


- **Limited Data Capacity:** Transfers fewer types of data compared to NASA.

### COM1 and COM2

NonNASA systems may use different communication buses.

This integration supports:

- **COM1**, typically connected to the Samsung **F1/F2** communication bus.
- **COM2**, connected to the Samsung **F3/F4** wired-controller communication bus.

COM1 remains the default for backwards compatibility. COM2 installations must
explicitly select the COM2 implementation in their ESPHome configuration:

```yaml
samsung_ac:
  non_nasa_bus: com2

COM2 uses different communication behaviour from COM1 and should not be treated
as simply another pair of terminals carrying the same protocol.

For COM2 wiring, configuration, implementation details, verified mode
constraints, and current limitations, see the
[COM2 Protocol](COM2-Protocol.md) documentation.