# Contributing

This repository is an open draft. Please open an issue before changing packet fields, CRC parameters, the Service UUID, or an assigned registry value.

- Clarify whether a proposal is wire-incompatible.
- Include an encoded packet example and expected decoded values for codec changes.
- Keep UTC semantics separate from local display metadata.
- Never reuse a registry ID after publication.
- Do not claim authentication from the CRC or the BLE advertiser address.

Draft changes may be made freely before version 1.0, but implementations should pin the draft revision they support.
