# BLE Time Beacon Protocol (BTB)

**Draft 0.1 — 2026-10-13**

This document defines a best-effort, one-way BLE time beacon. The beacon reports a sampled UTC time; it does not claim that the sample is the exact time the radio transmitted the packet.

## 1. Time model

The packet's timestamp is an unsigned count of POSIX seconds since `1970-01-01 00:00:00 UTC`, plus a binary fractional second. POSIX time does not encode a leap second as a distinct second. The UTC-valid and leap-warning flags describe the transmitter's status.

The sample time is the UTC instant used when the transmitter built the payload. The transmitter refreshes the payload on a best-effort schedule, nominally every 100 ms. BLE may transmit that payload later; the receiving scan schedule and host callback add more delay. Implementations MUST NOT interpret the timestamp as the exact RF transmit time.

`sample_uncertainty_ms` describes the transmitter's uncertainty in the UTC sample at payload construction. A receiver MUST additionally account for payload age, advertising scheduling, scan timing, and callback latency. If it cannot estimate those terms, it should report a suitably conservative total uncertainty.

## 2. Packet

The application payload is exactly 24 bytes. All multi-byte integers are little-endian unless stated otherwise.

| Offset | Size | Field | Encoding |
|---:|---:|---|---|
| 0 | 1 | `version` | `0x01` for this draft packet layout |
| 1 | 1 | `flags` | Bit assignments below |
| 2 | 3 | `source_id` | Stable provisioned unsigned 24-bit ID; little-endian; `0xFFFFFF` is unassigned and MUST be ignored by quorum logic |
| 5 | 6 | `utc_seconds` | Unsigned 48-bit POSIX seconds from the epoch |
| 11 | 2 | `utc_fraction` | Unsigned Q0.16 fraction of a second; time fraction is `utc_fraction / 65536` seconds |
| 13 | 2 | `sample_uncertainty_ms` | Unsigned milliseconds; `0xFFFF` means unknown |
| 15 | 2 | `sequence` | Per-source unsigned counter, incremented for every newly built packet and wrapping modulo 65536 |
| 17 | 2 | `utc_offset_minutes` | Signed 16-bit two's-complement offset east of UTC at the sample instant; e.g. `+60` for UTC+01:00, `-300` for UTC−05:00 |
| 19 | 2 | `timezone_id` | Registry ID for the transmitter's IANA time-zone name; `0x0000` means unknown |
| 21 | 1 | `locale_id` | Registry ID for a BCP 47 display locale; `0x00` means unspecified |
| 22 | 2 | `crc16` | CRC-16/CCITT-FALSE over bytes 0–21; little-endian representation |

The Q0.16 fraction has a resolution of about 15.26 microseconds. This is only a representation resolution; it is not a claim of 15-microsecond radio or clock accuracy.

### 2.1 Flags

| Bit | Name | Meaning when set |
|---:|---|---|
| 0 | `UTC_VALID` | Transmitter has a usable UTC estimate |
| 1 | `GNSS_LOCK` | Transmitter currently has a valid GNSS navigation solution |
| 2 | `PPS_VALID` | The GNSS PPS reference is currently considered valid |
| 3 | `HOLDOVER` | UTC is being maintained without a current primary reference lock |
| 4 | `LEAP_WARNING` | Transmitter knows of a leap-second event relevant to its UTC conversion |
| 5 | `TIMEZONE_VALID` | Offset and time-zone ID describe the sample instant |
| 6 | `DST_ACTIVE` | The transmitter's local civil time is observing daylight-saving time at the sample instant |
| 7 | Reserved | MUST be transmitted as zero; receivers MUST ignore it |

`GNSS_LOCK` and `HOLDOVER` are normally mutually exclusive. `UTC_VALID` may remain set during holdover. A receiver MUST reject packets without `UTC_VALID` for clock discipline, but may still use the source ID for diagnostics.

## 3. BLE transport

The initial interoperable transport uses a 128-bit Service Data AD structure (AD type `0x21`) with the BTB service UUID:

`08b658aa-8dfa-40fa-bdf6-b69291ffbec3`

UUID bytes in the advertising AD structure use Bluetooth's least-significant-byte-first UUID encoding. The 24-byte payload follows the 16-byte UUID. Because the Service Data structure plus UUID and payload exceeds the 31-byte legacy advertising limit, this transport profile requires **extended advertising** and extended scanning. A future legacy-compatible profile may use a legitimately assigned 16-bit company identifier or service UUID and a reduced payload; implementations MUST NOT invent or appropriate an assigned Bluetooth identifier.

The advertiser should be non-connectable and non-scannable unless a product has a separate reason to accept connections. The nominal payload refresh period is 100 ms. BLE event timing is not exact; the controller and RF environment can delay, jitter, or drop events.

## 4. Time zone and locale

UTC is the authoritative instant. `utc_offset_minutes` and `DST_ACTIVE` report local civil-time state **at the packet's sample instant**. `timezone_id` identifies the time-zone rule set so a receiver with a time-zone database can calculate future transitions. The offset is supplied as well so a small receiver can display the current local time without storing the complete rule database.

`locale_id` controls presentation conventions only (language, date order, separators, and related formatting). It MUST NOT alter UTC interpretation or time-zone arithmetic. A receiver may ignore locale metadata and use its own configured display locale.

Timezone and locale IDs are listed in [registry.md](registry.md). Unknown IDs MUST be preserved or treated as unknown, not reinterpreted as another zone or locale.

## 5. CRC

CRC-16/CCITT-FALSE parameters:

- Width: 16
- Polynomial: `0x1021`
- Initial value: `0xFFFF`
- Reflected input/output: no
- Final XOR: `0x0000`
- Covered bytes: offsets 0 through 21 inclusive
- Wire order: low byte, then high byte

The standard check string `123456789` produces CRC `0x29B1` (wire bytes `B1 29`).

Receivers MUST validate the CRC before using any field. A failed CRC is a dropped packet; it must not update the clock or source statistics used for consensus.

### 5.1 Example packet

This example represents `2026-10-13 12:34:56.500 UTC`, source ID `0xD311A5`, 25 ms sample uncertainty, sequence `0x1234`, local offset UTC+01:00, `Europe/London`, and locale `en-GB`. Flags `0x67` mean UTC valid, GNSS locked, PPS valid, timezone valid, and DST active.

```text
01 67 A5 11 D3 70 25 CE 6A 00 00 00 80 19 00 34 12 3C 00 02 00 01 58 27
```

The CRC is `0x2758` and is transmitted low byte first (`58 27`). As a 128-bit Service Data AD structure, the UUID's little-endian bytes are:

```text
C3 BE FF 91 92 B6 F6 BD FA 40 8D FA AA 58 B6 08
```

The extended-advertising AD element begins with length and type bytes `29 21`, followed by the 16 UUID bytes and the 24-byte packet above.

## 6. Receiver guidance

Receivers should timestamp each accepted report using a monotonic clock and keep source state keyed by `source_id`, not by a potentially private or rotating BLE address. State should include the receive monotonic timestamp, decoded UTC sample, sequence, reported uncertainty, RSSI, and quality flags.

For each source:

1. Reject bad CRCs, unsupported versions, invalid UTC, unassigned IDs, duplicates, and stale sequence values.
2. Form an observed offset between advertised UTC and receiver monotonic time. This is a noisy observation: payload age and receive-side delay are non-negative and vary.
3. Use a robust sliding estimator (for example, a bounded sample window with outlier rejection and a high percentile of the observed offset) and estimate drift slowly. Because `advertised_utc - receive_monotonic` becomes smaller as packet age grows, a low percentile selects more-delayed observations; the high side is closer to the least-delayed observations. Do not treat this as an exact one-way-delay measurement.
4. Discipline the local clock gradually. Avoid stepping it on each packet. Grow uncertainty during holdover and when packets stop arriving.
5. Combine only distinct provisioned source IDs. With three or more independent sources, a mutually agreeing cluster can identify an outlier. With two disagreeing sources, report ambiguity or request a trust choice. Multiple beacons derived from the same upstream clock are not independent votes.

RSSI may help rank link quality, but it is not a time-quality metric. CRC is not authentication. Deployments needing adversarial protection require an authenticated packet extension or a trusted transport/security layer.

## 7. Compatibility and versioning

Version 1.0 MUST freeze field meaning, byte order, CRC parameters, UUID, and registry assignments. Draft 0.1 is not yet frozen. A decoder MUST reject an unknown major packet version and MUST ignore reserved flag bits. A future incompatible layout requires a new version; fields must not be silently repurposed.
