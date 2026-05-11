# SpareParts Project — Claude Code Context

## User

Toni Kurvinen — Senior Product Manager at Swappie. Was a coder in the 90s but hasn't coded since — prefers simple, jargon-free explanations. GitHub: tokurvin.

## Rules

- **NEVER** read, display, or reference contents of files containing tokens, secrets, or credentials. If debugging a token issue, ask Toni to verify it himself — troubleshoot from error messages only.

## Project: SP QR Code & ASN Generator

Swappie spare part QR codes used for LP (lot/batch) traceability in repair operations. Scanned in Hemuli during phone repair, synced to NetSuite.

### Version history

| Version | Why it exists |
|---------|--------------|
| **v1** | Original code for battery traceability. Started on the outer packaging box only; expanded to also cover individual batteries and their cardboard boxes. No check digit — scanning errors go undetected. |
| **v2** | Name intentionally skipped. |
| **v3** | Display SPs started shipping in multiple shipments per PO, requiring a shipment number field. Added a check letter (mod 29) at the same time to catch scanning errors. Stayed within QR V2-M capacity by tightening field widths. |
| **v4** | EU Battery Regulation 2023/1542 requires a scannable QR code on each battery linking to a public landing page — so the code must be a URL. Moving to URL format required a larger QR version (V4-Q), which brought extra capacity; used that headroom to upgrade to two check characters (mod 37) and reorder fields into a more human-readable structure. |

### Format details

**v1 (in production since Dec 2025):** `1234LTAPLP98765POFI-12345678-99-11` — no check character, no validation. QR Version 2, ECC M (15%), alphanumeric mode.
Regex: `/^(?<qty>\d{1,5})(?<SKU>smaplp\d{1,5})(?<PO>pofi-\d{8})(?<totalBoxes>\d{1,4})-(?<boxId>\d{1,4})$/i`

**v3 (rollout paused):** `1234LTAPLP98765POFI-12345678-01-99-11C` — adds shipment number field and check letter (A-Z or $%*). QR Version 2, ECC M (15%), alphanumeric mode. Field sizes tightened vs v1 (totalBoxes and boxId capped at 2 digits) to stay within 38-char capacity while adding shipment # and check letter.
Check letter uses rolling hash: accumulator = (acc * 31 + char_value) mod 29, mapped to A-Z plus $%*. Prime modulus 29 guarantees detection of all single-char errors and adjacent transpositions.
Regex: `^\d{4}[A-Z][A-Z0-9]{8,10}PO[A-Z]{2}-\d{8}-\d{2}-\d{2}-\d{2}[A-Z$%*]$`

**v4 (battery sticker, EU compliance):** `HTTPS://QR.SWAPPIE.COM/POFI-12345678-LTAPLP98765-01-99-01-1234:XY` — URL QR on the battery sticker, encodes LP batch data (PO + SKU). Links directly to battery info landing page showing EU Battery Regulation required information. Must be URL format so it works for recyclers at end-of-device-lifecycle with no packaging present. Domain TBD: `qr.swappie.com` (23-char prefix) or `qr.swapp.ie` (20-char prefix). QR Version 4, ECC Q (25%), 67-char capacity. See spec-eu-qr-v4.md.

**v4 no-URL (proposed, internal traceability only):** `POFI-12345678-LTAPLP98765-01-99-01-1234:XY` — same as v4 but without URL. Suitable only where the code is scanned by Hemuli (not by end users or recyclers). 38–42 chars (max with 11-char SKU). QR Version 3, ECC Q (25%), alphanumeric mode, 29×29 modules, capacity 47 chars (5 chars headroom at max SKU length).

**Packaging QR (fulfillment, EU compliance):** URL QR printed on device packaging at fulfillment. Encodes device identifier type and value in URL path. Links to same battery info landing page as v4 battery sticker, resolved via IMEI→battery batch lookup in Hemuli. Format: `HTTPS://QR.SWAPPIE.COM/{type}/{identifier}`. Three identifier types:
- `HTTPS://QR.SWAPPIE.COM/IMEI/350521064534770` — iPhones (IMEI, 15 digits) → 43 chars, V3-Q fits
- `HTTPS://QR.SWAPPIE.COM/SN/C02XL0LHJGH5` — devices with serial number → ~37 chars, V3-Q fits
- `HTTPS://QR.SWAPPIE.COM/SDID/22F142AD-2A7F-449E-B3BA-53A38CFA8D1C` — Swappie Device ID (assigned at activation) → 64 chars, V4-Q required
SDID is the binding constraint: packaging QR must be V4-Q (67-char capacity) to cover all device types (iPhones, iPads, MacBooks).

**QR capacity reference:** `QR-code_v1-10_capacity.csv` — alphanumeric capacities for QR Versions 1–10, all ECC levels, sourced from qrcode.com. Always use this file when evaluating QR version/ECC fit. Do not rely on training data for QR capacities.

### Tools (this repo, hosted on tokurvin.github.io/sp-qr-tool)

**v3 tools (production):**
- `index.html` — landing page; shows v3 and v4 tool groups separately with format reference cards
- `qr-checker.html` — v3 checker/generator: validate/generate individual codes with check letter + QR preview
- `asn-generator.html` — v3 bulk ASN generator: PO + SKU + quantities → Excel/CSV with QR codes
- `asn-generator-v3.0.html` — shadow copy of v3.0 for download-bug comparison

**v4 tools (draft, EU Battery Regulation):**
- `qr-checker-v4.html` — combined v4 checker/generator: shows v4.0 (URL) and v4.1 (no URL) side by side with live QR images. Paste any v4 code to validate check chars; fill fields to generate. Includes dynamic link below the v4.0 output: opens `battery.html` with the generated path in the URL hash.
- `asn-generator-v4.0.html` — v4.0 bulk ASN generator: URL format (EU compliant), Excel/CSV
- `asn-generator-v4.1.html` — v4.1 bulk ASN generator: no-URL format (internal traceability only), Excel/CSV
- `battery.html` — demo EU Battery Regulation landing page. Reads the code from the URL hash: `battery.html#{path}:{checks}` e.g. `battery.html#POFI-00088285-SMAPLP1003-01-05-03-0010:BK`. Strips URL prefix if present, validates check chars, shows EU-required battery info by SKU type (SM/TB/LT). Has demo banner. Note: still references old domain `QR.SWAPPIE.CLOUD` internally (harmless since path is passed without prefix).

**GitHub MCP note:** The MCP server is named `github-swappie` but accesses Toni's personal GitHub account (tokurvin). The name is just a label — all pushes go to `tokurvin/sp-qr-tool`.

## Rollout Status (as of 2026-04-14)

**PAUSED** — v3 rollout on hold as of 2026-04-14 until EU Battery Regulation (2023/1542) and legal requirements are better understood. Decision: no further rollout steps until clarity on how EU QR/labelling obligations affect the LP QR code design.

- **Stakeholder OKs:** All received (Apr 7 meeting) — Iaroslav/procurement, Arto/finance, Vineet/warehousing, Allar+Eric/repair, Marko/tech
- **Vendor validation:** DONE — Main batteries vendor (China) confirmed tool works. Download bug in WeChat/in-app browsers fixed in v3.1.
- **Supplier can print v3 codes:** YES — format fits QR Version 2 limits
- **Supplier labeling guideline:** UPDATED by Iaroslav
- **NetSuite/WMS (Arto + Greenstep):** IN PROGRESS — Regex changes made, pending testing. WMS changes requested Mar 27.
- **Hemuli validation:** NOT STARTED — Needs regex + check letter verification + LP SKU must match phone model
- **Scanner fix:** DONE — Markus deployed 3x consecutive read requirement (live since Mar 27)
- **Repair line config:** DONE — Kevin enabled "Installs battery" check on Support Line 1.0
- **Official hosting:** NOT STARTED — currently on Toni's personal GitHub Pages; tech team to implement official version
- **Parallel support needed:** Old and new code formats must both be supported during transition
- **Data quality:** Xiaolan found 88/571 erroneous device-LP SKU pairs in BQ, only 101 Hemuli events affected — low severity

## Project: EU Battery Regulation 2023/1542

Raised by Iaroslav (Apr 13, 2026). Separate from but parallel to the LP QR code rollout.

- **18 Aug 2026 — Labelling deadline:** Batteries must carry labels with capacity, lifespan, performance, chemical composition, collection symbol, hazardous substance markings
- **18 Feb 2027 — QR code deadline:** All batteries must bear a QR code linking to: chemical composition, carbon footprint, lifespan, recycled content, recycling instructions, traceability data, EU declaration of conformity, waste management info
- **Battery category:** Swappie's spare part batteries are "portable batteries" (under 5 kg). No full digital battery passport required, but physical labels + QR code are mandatory.
- **Applicability:** Yes — covers all batteries placed on EU market. Swappie is an importer.
- **Separate from LP QR:** LP QR = internal supply chain traceability; EU QR = consumer-facing regulatory data. Could coexist on same sticker.
- **Open questions:** Who is responsible economic operator? Do suppliers already comply? What data does Swappie have vs. need? Who hosts the landing page? Does it apply to batteries in refurbished phones or only standalone spare parts?

## Stakeholders

- **Toni Kurvinen** (U01FNGUF77G) — Senior PM, project owner
- **Iaroslav Ivanov / Slava** (U01QXKFTA4C) — Supply chain / procurement, vendor interface, maintains supplier labeling guideline
- **Arto Leinonen** (U03M9CRRA10) — Finance/NetSuite/WMS, works with Greenstep on NS changes
- **Markus Tammeoja** (U026GCMUKC0) — Hemuli/scanner tech (DTF), implemented 3x read confirmation
- **Marko Rasa** (UQ868QKV5) — Tech lead / code reviewer, approved v3 code
- **Vineet Kulkarni** (U03CX8CS6NS) — Warehousing/ops
- **Xiaolan Geng** (U05SZK24HDE) — Data analyst, BigQuery
- **Kevin Lillend** (U01RXRZCXNJ) — Repair line configurator
- **Allar Satsi** (U0362CC7PSS) — Repair department
- **Eric Oja** — Repair
- **Hannele Ruse** — Meeting doc owner
- **Nicolas Spadaro** — Meeting attendee
- **Sudarshan Karki** (U02NGCSJ7FD) — Team member
- **Reijo Ojassoo** (U01HQS2C05C) — Stakeholder
- **DTF** (S02GB81AR7Y) — Tech implementation team for Hemuli changes
- **Greenstep** — External partner for NetSuite/WMS changes

## Key Documents & Links

- **Weekly meeting doc:** [Spare parts related Discovery](https://docs.google.com/document/d/1I5FSMLdCnnekXGmGwRF_CfzHOd94ynX1xDyD63Dgn0g/edit) — owned by Hannele Ruse
- **Supplier labeling guideline:** [Battery QR Code and LOT Labeling Guide](https://docs.google.com/document/d/1ZnZQTLEFzU5HdVVTeLm9-4m2MQRwCW19jKD2QQkrJO8/edit) — maintained by Iaroslav
- **Battery QR Code & Lot Tracking SOP:** [Link](https://docs.google.com/document/d/16tZwcAmquG1B8lTonB-EloRk4NrcYADxi55ksEAB570/edit)
- **DFD: Serialized Spare Parts:** [Link](https://docs.google.com/document/d/1HaP8Fif0u32kHq3RpfmS4vWec2ssJlSHXe5E5NK-weM/edit)
- **Label drafts:** [Presentation](https://docs.google.com/presentation/d/1eBZahcOcBeLKcHTKVgSn1sIiieN1KUusmGG22YcLvD0/edit)
- **Slack channel:** #spare-part-related-discovery (C042E9RDW3W)
- **QR tool (prototype):** https://tokurvin.github.io/sp-qr-tool/
- **GitHub repo:** tokurvin/sp-qr-tool
- **GitHub MCP:** A custom github-pat MCP server is configured for write access to Toni's repos
