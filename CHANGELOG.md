# Changelog — SP QR Tool

All significant changes to the QR code format, tools, and rollout decisions, newest first.

---

## 2026-06-09

### OPS scanner testing results documented
- Vineet Kulkarni (warehousing) + Eric Oja (repair) tested QR V4-Q at multiple sizes and print methods
- Laser print: passes at all tested sizes (9×9, 10×10, 16×16, 19×19 mm)
- Thermal print: fails at 10×10 and 15×15 mm on handheld scanners; passes at 19×19 mm
- Toni tested DYMO LabelWriter 450 (thermal, 300 DPI): iPhone 13 Pro Max read 10×10 mm successfully — phone cameras more tolerant than handheld scanners
- **Decision confirmed: 20×20 mm sticker spec** — round-number safe margin above 19 mm thermal pass; standard label stock; 0.49 mm/module at QR V4-Q

### GitHub history cleanup
- Full 66-commit git history pulled to local repo (Dropbox project folder)
- GitHub `main` branch replaced with single clean commit — only the 13 tool HTML files, no history
- `.gitignore` added: internal context files permanently excluded from git
- Local git retains full history; GitHub contains only current tool files

---

## 2026-05-19

### index.html restructured — current tools first, deprecated to bottom
- Moved v4 tools and format reference to top of page
- Added v1 format reference card (production format, no tooling)
- Moved v3 tools to clearly labelled "Deprecated" section
- Removed ASN Generator v3.0 shadow card (file did not exist; link was broken)
- Restored missing tools dropped in restructure: QR Code Checker v4, Battery Data Registration
- Fixed Battery Info Page Demo link to include example code hash so page shows content on open
- Battery Data Registration back-link updated to point to index.html instead of ASN generator

### asn-generator-v4.0.html — UUID gate changed to warning-only
- UUID was blocking download in the 2026-05-12 version
- Changed to warning-only: download proceeds but shows a warning if UUID is missing
- Rationale: UUID is optional until the Feb 2027 EU deadline; blocking it now prevents suppliers from generating codes for POs before they’ve registered battery data

### qr-checker-v4.html — domain corrected
- URL prefix updated from `HTTPS://QR.SWAPPIE.CLOUD/` to `HTTPS://SP.SWAPPIE.COM/`

---

## 2026-05-12

### Decision: v4.0 format adopted — two QR codes
- **v4.0 QR on the battery sticker** (supplier-printed): LP traceability data encoded in URL (`HTTPS://SP.SWAPPIE.COM/…`). Scanned by Hemuli at repair; also opens EU compliance landing page for anyone who scans it.
- **UUID QR on the sales box** (Swappie-printed at Grading): a sticker directly on the box or a leaflet with a sticker. UUID links to battery spec data registered by supplier. New UUID issued on battery replacement.
- **Domain confirmed:** `SP.SWAPPIE.COM` (`.com` = customer-facing per Swappie convention; specific subdomain needed for QR capacity reasons)
- Stakeholders present: Toni, Iaroslav, Arto, Markus, Marko, Vineet, Kevin, Allar, Eric + others
- See [meeting notes](https://docs.google.com/document/d/1I5FSMLdCnnekXGmGwRF_CfzHOd94ynX1xDyD63Dgn0g/edit)

### battery-upload.html added
- Prototype supplier portal for submitting battery spec data per SKU
- Fields: SKU, device type, capacity, voltage, chemistry, carbon footprint, recycled content, EU declaration of conformity URL, manufacturer
- Submits return a fixed demo UUID (no real backend yet)
- UUID flow: supplier registers data → gets UUID → enters UUID in ASN generator → UUID embedded in packaging QR

### asn-generator-v4.0.html — UUID gate added
- Added Battery Data UUID field per SKU row
- UUID validation gate added — initially blocking (required before download)
- Link to battery-upload.html for suppliers who need to register data first
- **Note:** UUID gate changed to warning-only in 2026-05-19 update (see above) — UUID optional until Feb 2027 EU deadline

---

## 2026-04-14

### v3 rollout stopped — meeting decision
- Team meeting: decided to stop all further v3 rollout steps pending clarity on EU Battery Regulation implications
- Decision: no further rollout steps until the implications are understood
- Outcome (resolved 2026-05-12): v4.0 format designed to serve both purposes simultaneously

### spec-eu-qr-v4.md created (draft)
- Initial v4 spec written exploring how to embed the LP code in a URL
- Domain in this draft (`QR.SWAPPIE.CLOUD`) was later superseded by `SP.SWAPPIE.COM` (confirmed 2026-05-12)

## 2026-04-13

### EU Battery Regulation 2023/1542 raised
- Iaroslav raised EU Battery Regulation requirements; team learned for the first time that batteries placed on the EU market require a QR code linking to compliance information by Feb 2027
- Key open questions: labelling deadline (18 Aug 2026), QR code deadline (18 Feb 2027), whether Swappie as EU importer is the responsible party, whether the LP QR and the EU QR could be the same code

---

## 2026-03-27

### Scanner fix deployed
- Markus (DTF) deployed 3× consecutive read requirement to Hemuli scanner
- Prevents accidental single-scan false positives

### NetSuite/WMS regex change requested
- Arto requested WMS changes to accept v4 format (in progress with Greenstep)

---

## 2026-03 (approximate)

### v4.0 ASN Generator built
- `asn-generator-v4.0.html`: bulk QR code generation for full shipments
- URL-based format: `HTTPS://QR.SWAPPIE.CLOUD/…` (domain later changed to SP.SWAPPIE.COM)
- Dual check characters, MOD 37
- QR Version 4-Q (25% error correction, 67 alphanumeric capacity)
- Excel and CSV export
- WeChat/in-app browser download bug found and fixed (v3.1 of tool)

### v4.1 prototype built (then deprecated)
- `asn-generator-v4.1.html`: no-URL variant of v4 format
- Deprecated in favour of v4.0 — URL format required for EU compliance

### EU Battery Regulation landing page demo built
- `battery.html`: consumer-facing demo showing what compliance info page will look like
- Parses v4 QR code from URL hash, displays battery chemistry, capacity, carbon footprint, recycling info

### QR Code Checker v4 built
- `qr-checker-v4.html`: validate or generate individual v4 codes
- Supports both v4.0 (URL) and v4.1 (no URL) — same check char logic for both
- Preview link to battery.html with live-generated code

---

## 2026-02 (approximate)

### v3 tools completed
- `qr-checker.html`: validate/generate v3 format codes
- `asn-generator.html`: bulk ASN generation for v3 format
- Supplier validation done — main batteries vendor (China) confirmed tool works

### v3 rollout planned but blocked
- All stakeholder approvals received (Iaroslav, Arto, Vineet, Allar/Eric, Marko)
- NetSuite/WMS regex update in progress
- Hemuli validation not yet started when rollout was paused

---

## 2025 (original v3 development)

### v3 format designed
- New format replacing v1: added single check letter (MOD 29, A–Z + $%*)
- Removed hyphen between qty and SKU vs v1
- Format: `1234LTAPLP12345POFI-87654321-01-98-12K`
- Rationale: v1 had no error detection; check letter catches all single-char errors and transpositions

### v1 format (original, still in production)
- Format: `1234-LTAPLP12345POFI-87654321-01-98-12`
- Plain text, no check digit, hyphen between qty and SKU
- Placed on batteries by suppliers since original spare parts traceability rollout
- Must continue to be accepted by Hemuli and NetSuite/WMS throughout any transition

---

## Format comparison summary

| Version | Format example | Check | Status |
|---------|---------------|-------|--------|
| v1 | `1234-LTAPLP12345POFI-87654321-01-98-12` | None | **In production** — Hemuli + NetSuite must accept |
| v3 | `1234LTAPLP12345POFI-87654321-01-98-12K` | 1 char, MOD 29 | **Deprecated** — never deployed |
| v4.1 | `POFI-00088285-SMAPLP1003-01-05-03-0010:BK` | 2 chars, MOD 37 | **Deprecated** — intermediate prototype |
| v4.0 | `HTTPS://SP.SWAPPIE.COM/POFI-00088285-SMAPLP1003-01-05-03-0010:BK` | 2 chars, MOD 37 | **Current standard** |
