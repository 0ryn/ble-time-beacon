# BTB registries (draft)

Registry assignments in draft 0.1 are provisional until protocol version 1.0 is frozen. IDs are transmitted little-endian.

## Time zones

`timezone_id` maps to an IANA Time Zone Database name.

| ID | IANA name |
|---:|---|
| `0x0000` | Unknown / not supplied |
| `0x0001` | `Etc/UTC` |
| `0x0002` | `Europe/London` |
| `0x0003` | `Europe/Paris` |
| `0x0004` | `Europe/Berlin` |
| `0x0005` | `America/New_York` |
| `0x0006` | `America/Chicago` |
| `0x0007` | `America/Los_Angeles` |
| `0x0008` | `Asia/Tokyo` |
| `0x0009` | `Asia/Kolkata` |
| `0x000A` | `Australia/Sydney` |
| `0xFFFE` | Private-use assignment; deployment-specific |
| `0xFFFF` | Reserved |

The signed UTC offset and DST flag describe the sample instant even when the receiver does not recognize the time-zone ID. A receiver needs current IANA time-zone rules to predict future transitions reliably.

## Display locales

`locale_id` maps to a BCP 47 language tag.

| ID | BCP 47 tag |
|---:|---|
| `0x00` | Unspecified; use receiver setting |
| `0x01` | `en-GB` |
| `0x02` | `en-US` |
| `0x03` | `en-CA` |
| `0x04` | `fr-FR` |
| `0x05` | `de-DE` |
| `0x06` | `es-ES` |
| `0x07` | `ja-JP` |
| `0x08` | `zh-CN` |
| `0xFE` | Private-use assignment; deployment-specific |
| `0xFF` | Reserved |

Do not allocate an ID already listed here. Registry changes should be proposed as a pull request with the time-zone database or BCP 47 reference name.
