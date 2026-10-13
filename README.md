# BLE Time Beacon

An open, connectionless BLE protocol for broadcasting a UTC clock sample and enough metadata for small receivers to display local civil time. Receivers can tolerate missed packets, filter timing variation, and compare multiple independent sources.

**Status: draft 0.1.** The packet layout and initial registries are proposed for review; implementations should identify themselves as experimental until version 1.0 is frozen.

## Contents

- [Protocol draft](docs/protocol.md): packet layout, time semantics, flags, integrity check, timezone and locale registries, and receiver guidance.
- [Registry](docs/registry.md): initial timezone and locale IDs.
- [License](LICENSE): MIT.

The protocol carries UTC as the canonical instant. A receiver derives the local date and clock from the UTC sample, timezone ID, and offset. The locale ID is presentation metadata; it does not change the instant.

## Design goals

- Work without a BLE connection or request/response exchange.
- Keep sending when a receiver is absent; a receiver accepts the next valid fresh packet.
- Detect accidental packet corruption and tolerate missed or duplicated packets.
- Carry quality and uncertainty so receivers do not mistake timestamp resolution for clock accuracy.
- Support multiple transmitters with stable IDs and receiver-side outlier handling.

CRC detects accidental corruption. It does not authenticate the transmitter or protect against deliberate spoofing.

## Contributions

Protocol changes should be discussed before changing the version 1 packet layout or assigning registry values. See [CONTRIBUTING.md](CONTRIBUTING.md).
