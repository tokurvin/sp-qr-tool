# SP QR Code v4 Specification — EU Battery Regulation Compliant

**Status:** DRAFT
**Date:** 2026-04-14
**Author:** Toni Kurvinen

## 1. Goal

Design a single QR code that serves two purposes:
1. **Internal LP traceability** — scanned by Hemuli during repair (same as v3)
2. **EU Battery Regulation 2023/1542 compliance** — scanned by anyone, opens a web page with required battery information

This replaces the v3 rollout. By embedding the LP code in a URL, we get EU compliance from day one and avoid a second format change later.

## 2. Code Format

### Example

```
HTTPS://QR.SWAPPIE.CLOUD/POFI-00088285-SMAPLP1003-01-05-03-0010:XY
```

### Structure

Fields in logical descending order — broadest to most specific:

```
HTTPS://QR.SWAPPIE.CLOUD/{po_country}-{po_number}-{lp}-{shp#}-{boxes}-{box#}-{qty}:{check1}{check2}
```

| Field      | Example               | Description                             |
|------------|-----------------------|-----------------------------------------|
| URL prefix | HTTPS://QR.SWAPPIE.CLOUD/ | Landing page base URL              |
| po_country | POFI                  | "PO" + 2-letter country code            |
| po_number  | 00088285              | 8-digit purchase order number           |
| lp         | SMAPLP1003            | LP/lot identifier (7–11 alphanumeric)   |
| shp#       | 01                    | 2-digit shipment number                 |
| boxes      | 05                    | 2-digit total boxes in shipment         |
| box#       | 03                    | 2-digit box number                      |
| qty        | 0010                  | 4-digit batch quantity (zero-padded)    |
| check1     | X                     | First check character (A–Z, 0–9, or $) |
| check2     | Y                     | Second check character (A–Z, 0–9, or $)|

### Separators

- Dashes `-` between all data fields
- Colon `:` before check characters — visually distinct, unambiguous

### Space budget (V4-Q, 67 alphanumeric capacity)

URL prefix `HTTPS://QR.SWAPPIE.CLOUD/` = 25 chars (fixed)

| LP length | Path chars | Total URL chars | Headroom |
|-----------|------------|-----------------|----------|
| 7 chars (min, e.g. SMAPLP1) | 38 | 63 | 4 spare |
| 11 chars (max, e.g. SMAPLP12345) | 42 | **67** | **0 spare** |

Exactly at capacity for longest codes. If more space is ever needed, dropping from Q to M error correction raises capacity to 90 chars — plenty of room.

### Regex (path part only, after URL prefix)

```
^PO[A-Z]{2}-\d{8}-(?:SM|TB|LT)APLP\d{1,5}-\d{2}-\d{2}-\d{2}-\d{4}:[A-Z0-9$]{2}$
```

### Full URL regex

```
^HTTPS://QR\.SWAPPIE\.CLOUD/PO[A-Z]{2}-\d{8}-(?:SM|TB|LT)APLP\d{1,5}-\d{2}-\d{2}-\d{2}-\d{4}:[A-Z0-9$]{2}$
```

## 3. Check Characters

Two independent check characters, computed over the **path part** of the URL (everything between the last `/` and the `:` separator — i.e. all data fields including dashes, excluding the colon and check chars themselves).

```
Input example: POFI-00088285-SMAPLP1003-01-05-03-0010
```

### Alphabet: 37 symbols (prime)

```
A B C D E F G H I J K L M N O P Q R S T U V W X Y Z 0 1 2 3 4 5 6 7 8 9 $
```

Index 0–25 → A–Z, 26–35 → 0–9, 36 → $

### Character values used in hash

| Character | Value |
|-----------|-------|
| 0–9 | 0–9 |
| A–Z | 10–35 |
| - (dash) | 36 |

### Algorithm

**Check char 1 (multiplier = 31):**
1. Start with accumulator = 0
2. For each character in path: `accumulator = (accumulator × 31 + char_value) mod 37`
3. Map result → alphabet (0→A … 25→Z, 26→0 … 35→9, 36→$)

**Check char 2 (multiplier = 17):**
1. Same as above with multiplier 17

Both 31 and 17 are prime and coprime to 37, preserving all mathematical guarantees.

### Error detection strength

| Error type                     | Detected |
|--------------------------------|----------|
| Any single character wrong     | **100%** |
| Any two adjacent chars swapped | **100%** |
| Any two characters wrong       | **100%** |
| Three or more characters wrong | 99.93%   |

### Why mod 37?

- Prime modulus = guaranteed detection of all single-char errors and transpositions
- Larger alphabet than v3's mod 29 = stronger detection per character
- No `%` or `*` — avoids URL percent-encoding issues
- Digits 0–9 usable as check chars thanks to the `:` separator

## 4. QR Encoding

| Property         | Value                                              |
|------------------|----------------------------------------------------|
| Content          | Full URL, uppercase                                |
| Encoding mode    | Alphanumeric (A–Z, 0–9, -, :, ., /)               |
| QR Version       | 4 (33×33 modules)                                  |
| Error correction | Q (25%)                                            |
| Max capacity     | 67 alphanumeric characters                         |
| Our usage        | 63–67 characters (depends on LP length)            |
| Sticker size     | 20×20mm                                            |
| Module size      | 20 ÷ 41 = **0.49mm** (41 = 33 modules + 4+4 quiet zone) |

**Why uppercase URL?** QR alphanumeric mode only supports uppercase. Uppercase URLs work fine — browsers and servers treat domain names as case-insensitive, and `qr.swappie.cloud` can be configured to accept uppercase paths. Keeps the code in a compact QR version instead of requiring byte mode.

**Why V4-Q over V3-M?**
- V3-Q (47 chars) doesn't fit our codes
- V4-Q (67 chars) fits exactly
- Q error correction (25%) vs M (15%) — meaningfully better scanning reliability, important given known scanning issues with v1 stickers
- 0.49mm modules at 20mm sticker vs 0.45mm at 15mm — bigger and more robust

**Escape hatch:** if the format ever needs to grow, dropping to V4-M raises capacity to 90 chars — plenty of room without changing sticker size.

## 5. Domain

`qr.swappie.cloud` — subdomain on Swappie's existing `.cloud` infrastructure (same as Hemuli). Kept separate from commercial `.fi` sites.

- [ ] DNS: add `QR` subdomain (tech team / DTF)
- [ ] SSL certificate for HTTPS

## 6. How Each System Reads the Code

### Hemuli (internal repair scanning)

1. Scanner reads QR → gets full URL string
2. Strip URL prefix `HTTPS://QR.SWAPPIE.CLOUD/`
3. Split on `:` → path data + check chars
4. Validate both check characters
5. Parse fields from path: `{po_country}-{po_number}-{lp}-{shp#}-{boxes}-{box#}-{qty}`
6. Match LP SKU to phone model (existing logic)

**Backward compatibility:** Hemuli must also accept old v3-format codes (without URL prefix) during transition period.

### Consumer / inspector (EU compliance)

1. Scan QR with any phone camera → opens browser
2. Browser navigates to `https://qr.swappie.cloud/POFI-00088285-SMAPLP1003-01-05-03-0010:XY`
3. Landing page displays EU-required battery information for that SKU

### Supplier (code generation)

1. Open ASN generator tool (updated to v4)
2. Enter PO number, SKU, quantities
3. Tool generates full URLs as QR codes
4. Supplier prints 20×20mm QR stickers and applies to batteries

## 7. Landing Page (qr.swappie.cloud)

The URL resolves to a web page showing EU Battery Regulation required information.

### Required information (EU 2023/1542, portable batteries)

**By Aug 2026 (labelling deadline):**
- Battery capacity (Ah or Wh)
- Minimum average duration / expected lifetime
- Chemical composition (e.g. Li-ion)
- Separate collection symbol (crossed-out wheelie bin)
- Hazardous substance markings (Cd, Pb if applicable)

**By Feb 2027 (QR code deadline):**
- All of the above, plus:
- Carbon footprint declaration
- Recycled content share
- Recycling/end-of-life instructions
- Traceability data (manufacturer, batch, origin)
- EU declaration of conformity
- Waste prevention and management information

### Open questions

- [ ] Where does battery data come from? (Suppliers? Swappie internal?)
- [ ] Who builds and maintains the landing page? (Tech team / DTF?)
- [ ] Hosting setup on qr.swappie.cloud?
- [ ] Per-SKU data sufficient, or per-batch?

## 8. Migration

| Phase | What |
|-------|------|
| **Now** | Spec review and approval |
| **Next** | Update ASN generator tool to v4 format |
| **Next** | Set up qr.swappie.cloud with placeholder landing page |
| **Next** | Update Hemuli: strip URL prefix, parse new field order, accept both old and new formats |
| **Aug 2026** | Landing page shows labelling-required data |
| **Feb 2027** | Landing page shows full QR-required data |

## 9. Changes from v3

| Aspect | v3 | v4 |
|--------|----|----|
| Format | `0010SMAPLP1003POFI-00088285-01-05-03K` | `HTTPS://QR.SWAPPIE.CLOUD/POFI-00088285-SMAPLP1003-01-05-03-0010:XY` |
| Field order | qty, SKU, PO | PO country, PO number, SKU, shp#, boxes, box#, qty |
| Separators | 3 dashes | dashes between fields, `:` before check chars |
| Check characters | 1 (mod 29, A–Z + $%*) | 2 (mod 37, A–Z + 0–9 + $) |
| Error detection | 97% (single-char: 100%) | 99.93% (single + double: 100%) |
| QR content | Plain text | URL |
| QR Version | 2 (25×25) | 4 (33×33) |
| Error correction | M (15%) | Q (25%) |
| QR capacity used | ~38/38 chars | 63–67/67 chars |
| Module size | 0.45mm @ 15×15mm | 0.49mm @ 20×20mm |
| Sticker size | 15×15mm | 20×20mm |
| EU compliant | No | Yes |
| Web landing page | No | Yes (qr.swappie.cloud) |
