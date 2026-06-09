# SP QR Code v4 Specification — EU Battery Regulation Compliant

**Status:** ACTIVE  
**Decision date:** 2026-05-12  
**Author:** Toni Kurvinen  
**Last updated:** 2026-05-19

> **Change from draft:** Domain confirmed as `SP.SWAPPIE.COM` (was `QR.SWAPPIE.CLOUD`). Modified Option 3 (two QR codes) approved by all stakeholders 2026-05-12.

---

## 1. Goal

Design a QR code that serves two purposes simultaneously:

1. **Internal LP traceability** — scanned by Hemuli during repair, links battery lot to phone being repaired, synced to NetSuite/WMS
2. **EU Battery Regulation 2023/1542 compliance** — scanned by anyone, opens a Swappie-hosted landing page with required battery information

This replaces the planned v3 rollout. By embedding the LP code in a URL, we get EU compliance from day one without a second format change later.

---

## 2. Code Format

### Example

```
HTTPS://SP.SWAPPIE.COM/POFI-00088285-SMAPLP1003-01-05-03-0010:BK
```

### Structure

```
HTTPS://SP.SWAPPIE.COM/{po_country}-{po_number}-{lp}-{shp#}-{boxes}-{box#}-{qty}:{check1}{check2}
```

| Field | Example | Description |
|-------|---------|-------------|
| URL prefix | `HTTPS://SP.SWAPPIE.COM/` | Fixed; stripped by Hemuli & WMS before parsing |
| po_country | `POFI` | "PO" + 2-letter country code |
| po_number | `00088285` | 8-digit purchase order number |
| lp | `SMAPLP1003` | LP/lot identifier — SM/TB/LT + APLP + 1–5 digits |
| shp# | `01` | 2-digit shipment number |
| boxes | `05` | 2-digit total boxes in shipment |
| box# | `03` | 2-digit box number |
| qty | `0010` | 4-digit batch quantity (zero-padded) |
| check1 | `B` | First check character (A–Z, 0–9, or $) |
| check2 | `K` | Second check character (A–Z, 0–9, or $) |

### Separators

- Dashes `-` between all data fields
- Colon `:` before check characters — visually distinct, unambiguous, valid QR alphanumeric char

### Domain

`SP.SWAPPIE.COM` — confirmed 2026-05-12.

Rationale:
- `.com` = customer-facing per Swappie domain convention (Antti)
- Specific subdomain required: `/path` prefix approach not feasible within QR Version 4-Q capacity constraints (Matthieu)
- Domain must be ≤16 chars to keep total code within 67-char capacity — `SP.SWAPPIE.COM` = 14 chars ✓

### Regex (path part, after URL prefix, before `:`)

```
^PO[A-Z]{2}-\d{8}-(?:SM|TB|LT)APLP\d{1,5}-\d{2}-\d{2}-\d{2}-\d{4}:[A-Z0-9$]{2}$
```

### Full URL regex

```
^HTTPS://SP\.SWAPPIE\.COM/PO[A-Z]{2}-\d{8}-(?:SM|TB|LT)APLP\d{1,5}-\d{2}-\d{2}-\d{2}-\d{4}:[A-Z0-9$]{2}$
```

### Space budget (V4-Q, 67 alphanumeric capacity)

URL prefix `HTTPS://SP.SWAPPIE.COM/` = 23 chars (fixed)

| LP length | Path chars | Total URL chars | Headroom |
|-----------|------------|-----------------|----------|
| 7 chars min (e.g. SMAPLP1) | 38 | 61 | 6 spare |
| 10 chars (e.g. SMAPLP1003) | 41 | 64 | 3 spare |
| 11 chars max (e.g. SMAPLP12345) | 42 | 65 | 2 spare |

Fits comfortably within capacity for all valid LP lengths.

---

## 3. Check Characters

Two independent check characters computed over the **path part** of the URL — everything between the last `/` and the `:` separator. The URL prefix is excluded from the calculation.

```
Input example: POFI-00088285-SMAPLP1003-01-05-03-0010
```

### Alphabet: 37 symbols (prime modulus)

```
A B C D E F G H I J K L M N O P Q R S T U V W X Y Z 0 1 2 3 4 5 6 7 8 9 $
```

Index 0–25 → A–Z, 26–35 → 0–9, 36 → $

### Character values used in hash

| Character | Value |
|-----------|-------|
| `0`–`9` | 0–9 |
| `A`–`Z` | 10–35 |
| `-` (dash) | 36 |

### Algorithm

**Check char 1 (multiplier = 31):**
1. Start with accumulator = 0
2. For each character in path: `accumulator = (accumulator × 31 + char_value) mod 37`
3. Map result → alphabet (0→A … 25→Z, 26→0 … 35→9, 36→$)

**Check char 2 (multiplier = 17):**
1. Same as above with multiplier 17

Both 31 and 17 are prime and coprime to 37 — preserves full mathematical error-detection guarantees.

### Error detection

| Error type | Detected |
|------------|----------|
| Any single character wrong | **100%** |
| Any two adjacent chars swapped | **100%** |
| Any two characters wrong | **100%** |
| Three or more characters wrong | 99.93% |

### Why mod 37?

- Prime modulus guarantees 100% detection of all single-char errors and adjacent transpositions
- Larger alphabet than v3's mod 29 — stronger detection per character
- No `%` or `*` in output alphabet — avoids URL percent-encoding issues
- Digits 0–9 usable as check chars thanks to the `:` separator (unlike v3 where they'd be ambiguous)

---

## 4. QR Encoding

| Property | Value |
|----------|-------|
| Content | Full URL, uppercase |
| Encoding mode | Alphanumeric (supports A–Z, 0–9, `-`, `:`, `.`, `/`) |
| QR Version | 4 (33×33 modules) |
| Error correction | Q (25%) |
| Max capacity | 67 alphanumeric characters |
| Our usage | 61–65 characters (depends on LP length) |
| Sticker size | 20×20mm |
| Module size | 20 ÷ 41 ≈ **0.49mm** (41 = 33 modules + 4+4 quiet zone) |

**Why uppercase URL?** QR alphanumeric mode only supports uppercase. Uppercase URLs work — browsers treat domain names as case-insensitive, and the server can be configured to accept uppercase paths. Keeps the code in compact alphanumeric mode instead of byte mode (which would require a larger QR version).

**Why V4-Q?**
- V3-Q (47 chars) doesn't fit codes with a URL prefix
- V4-Q (67 chars) fits all valid LP lengths with headroom
- Q (25%) vs M (15%) error correction — meaningfully better scanning reliability, important given documented scanning issues with v1 stickers
- 0.49mm modules at 20mm sticker vs 0.45mm at 15mm — physically larger, more robust

**Escape hatch:** If the format ever needs to grow, dropping to V4-M raises capacity to 90 chars — plenty of room without changing sticker size.

---

## 5. How Each System Reads the Code

### Hemuli (internal repair scanning)

1. Scanner reads QR → receives full URL string
2. Strip URL prefix `HTTPS://SP.SWAPPIE.COM/`
3. Split on `:` → path data + check chars (2 chars)
4. Validate both check characters against recomputed values
5. Parse fields: `{po_country}-{po_number}-{lp}-{shp#}-{boxes}-{box#}-{qty}`
6. Match LP SKU to phone model being repaired (existing logic)
7. Record traceability event

**Backward compatibility required:** Hemuli must also accept v1 format codes (`^\d{4}-[A-Z]…`) during transition period. v1 codes have no check digit to validate.

### Consumer / end user / recycler (EU compliance path)

1. Scan QR with any phone camera → browser opens URL automatically
2. Browser navigates to `https://sp.swappie.com/POFI-00088285-SMAPLP1003-01-05-03-0010:BK`
3. Landing page looks up SKU in battery data database
4. Displays EU-required battery information for that SKU

### Supplier (code generation)

1. Open ASN Generator v4.0 tool
2. Enter PO number, SKUs, quantities
3. Tool generates full URL codes as QR images
4. Download as Excel or CSV
5. Print on 20×20mm stickers and apply to batteries

---

## 6. EU Battery Regulation 2023/1542 — Requirements

Swappie's spare part batteries are classified as **portable batteries** (under 5 kg). No full Digital Battery Passport required, but physical labels and QR code are mandatory.

Swappie is the **EU importer** — Swappie bears responsibility for compliance even if batteries are manufactured and labelled by a third party.

### Deadlines

| Deadline | What is required |
|----------|------------------|
| **18 Aug 2026** | Physical labels: capacity, minimum lifespan, chemistry, separate collection symbol (crossed-out bin), hazardous substance markings (Cd, Pb, Hg if applicable) |
| **18 Feb 2027** | QR code on all batteries. Must link to: chemical composition, carbon footprint, expected lifespan, recycled content share, recycling/end-of-life instructions, traceability data, EU declaration of conformity, waste management info |

### Data retention

The landing page data must remain live for the **battery's expected lifespan** (Regulation requirement). This is a long-term hosting commitment — the page cannot simply be taken down after sale.

### What the landing page must show (Feb 2027)

- Battery chemistry and composition
- Capacity (Wh and/or mAh)
- Expected lifespan / rated cycle count
- Carbon footprint declaration
- Recycled content (cobalt, lithium, nickel %)
- Hazardous substance markings
- Recycling and collection instructions
- Traceability: manufacturer, country of manufacture, batch/lot
- EU Declaration of Conformity link
- Waste prevention and management information

---

## 7. Modified Option 3 — Two QR Codes

Decision: 2026-05-12 stakeholder meeting.

Two complementary QR codes coexist, both pointing to the same landing page (`sp.swappie.com`):

### QR Code 1 — Battery sticker (supplier-printed)

- Printed by: supplier, on the battery label
- Contains: full LP traceability data encoded in the URL
- Format: `HTTPS://SP.SWAPPIE.COM/{PO}-{SKU}-{shp}-{boxes}-{box}-{qty}:{CC}`
- Used by: Hemuli at repair (internal), anyone who scans it (EU compliance)
- Note: if the battery is inaccessible after installation (inside device), the EU regulation may exempt this QR from consumer-facing requirements — to be confirmed

### QR Code 2 — Packaging / leaflet (Swappie-printed at Grading)

- Printed by: Swappie, on device box or leaflet
- Contains: UUID that links to battery spec data
- Supplier first registers battery data in the supplier portal → receives UUID
- UUID stored: links battery lot → phone serial → customer order
- On battery replacement in repair: new UUID issued, old UUID invalidated
- Used by: customer, recycler (EU compliance path)

---

## 8. Open Questions (as of 2026-05-19)

| Question | Status |
|----------|--------|
| Who is responsible economic operator — Swappie alone or shared with supplier? | Open |
| Do existing suppliers already have EU-compliant battery data ready? | Open |
| Does regulation apply to batteries in refurbished phones, or only standalone spare parts? | Open |
| Can on-battery QR be exempted if inaccessible after installation? (Lotta: possibly yes) | Likely yes — confirm |
| Who builds and hosts the supplier portal? | Open |
| Who builds and hosts sp.swappie.com landing page? | Open |
| Timeline for landing page before 18 Aug 2026 labelling deadline? | Open |
| DNS setup: `SP` subdomain on SWAPPIE.COM | Not started |
| SSL certificate for HTTPS on sp.swappie.com | Not started |

---

## 9. Changes from v3

| Aspect | v3 | v4.0 |
|--------|----|---------|
| Format | `0010SMAPLP1003POFI-87654321-01-05-03K` | `HTTPS://SP.SWAPPIE.COM/POFI-00088285-SMAPLP1003-01-05-03-0010:BK` |
| Field order | qty, SKU, PO | PO country, PO number, SKU, shp#, boxes, box#, qty |
| Separators | 3 dashes total | dashes between all fields, `:` before check chars |
| Check characters | 1 (mod 29, A–Z + $%*) | 2 (mod 37, A–Z + 0–9 + $) |
| Error detection | single-char: 100% | single + double: 100%, any two chars: 99.93% |
| QR content | Plain text | URL — opens browser on scan |
| QR Version | 2 (25×25 modules) | 4 (33×33 modules) |
| Error correction | M (15%) | Q (25%) |
| Sticker size | 15×15mm | 20×20mm |
| EU compliant | No | Yes |
| Web landing page | No | Yes (sp.swappie.com) |
| Deployed | Never | Rolling out |
