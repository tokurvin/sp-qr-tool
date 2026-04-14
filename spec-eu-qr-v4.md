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
HTTPS://Q.SWAPPIE.FI/0010-SMAPLP1003-POFI-00088285-01-05-03-XY
```

### Structure

```
HTTPS://Q.SWAPPIE.FI/{qty}-{lp}-{po_country}-{po_number}-{line}-{box}-{seq}-{check1}{check2}
```

| Field       | Example              | Description                              |
|-------------|----------------------|------------------------------------------|
| URL prefix  | HTTPS://Q.SWAPPIE.FI/ | Landing page base URL                  |
| qty         | 0010                 | 4-digit batch quantity (zero-padded)     |
| lp          | SMAPLP1003           | LP/lot identifier (5–11 alphanumeric)    |
| po_country  | POFI                 | "PO" + 2-letter country code             |
| po_number   | 00088285             | 8-digit purchase order number            |
| line        | 01                   | 2-digit shipment number                  |
| box         | 05                   | 2-digit total boxes                      |
| seq         | 03                   | 2-digit box number                       |
| check1      | X                    | First check character (A–Z, 0–9, or $)   |
| check2      | Y                    | Second check character (A–Z, 0–9, or $)  |

### Dashes

Six dashes — one between every logical block:
- After qty: `0010-SMAPLP1003`
- After LP: `SMAPLP1003-POFI`
- After PO country: `POFI-00088285`
- Between line, boxes, box#: `-01-05-03`
- Before check characters: `-XY`

Dashes are **included** in the check character computation (they're part of the code string).

### Space budget (V4-Q, 67 alphanumeric capacity)

| LP length | Path chars | Total URL chars | Headroom |
|-----------|-----------|-----------------|----------|
| 5 chars (min) | 38 | 59 | 8 spare |
| 11 chars (max) | 44 | 65 | 2 spare |

Wait — let me recalculate with 6 dashes:
- Fixed: `0010-` + `-POFI-00088285-01-05-03-XY` = 5 + 27 = 32
- Variable: LP (5–11 chars) + 1 dash before POFI
- Min: 32 + 5 + 1 = 38 path → 59 URL (8 spare)
- Max: 32 + 11 + 1 = 44 path → 65 URL (2 spare)

### Regex (path part only, after URL prefix)

```
^\d{4}-[A-Z][A-Z0-9]{4,10}-PO[A-Z]{2}-\d{8}-\d{2}-\d{2}-\d{2}-[A-Z0-9$]{2}$
```

### Full URL regex

```
^HTTPS://Q\.SWAPPIE\.FI/\d{4}-[A-Z][A-Z0-9]{4,10}-PO[A-Z]{2}-\d{8}-\d{2}-\d{2}-\d{2}-[A-Z0-9$]{2}$
```

## 3. Check Characters

Two independent check characters, computed over the **path part** of the URL (everything between the last `/` and the check characters themselves, including all dashes).

```
Input: 0010-SMAPLP1003-POFI-00088285-01-05-03
```

### Alphabet: 37 symbols (prime)

```
A B C D E F G H I J K L M N O P Q R S T U V W X Y Z 0 1 2 3 4 5 6 7 8 9 $
```

Index 0–25 → A–Z, 26–35 → 0–9, 36 → $

### Algorithm

Both use a rolling hash with the same modulus (37) but different multipliers:

**Check char 1 (multiplier = 31):**
1. Start with accumulator = 0
2. For each character: accumulator = (accumulator × 31 + char_code) mod 37
3. Map result to alphabet (0 → A, ..., 25 → Z, 26 → 0, ..., 35 → 9, 36 → $)

**Check char 2 (multiplier = 17):**
1. Start with accumulator = 0
2. For each character: accumulator = (accumulator × 17 + char_code) mod 37
3. Map result to same alphabet

Both 31 and 17 are prime and coprime to 37, preserving the mathematical guarantees.

### Error detection strength

| Error type                    | Detected |
|-------------------------------|----------|
| Any single character wrong    | **100%** |
| Any two adjacent chars swapped | **100%** |
| Any two characters wrong      | **100%** |
| Three or more characters wrong | 99.93%  |

The two independent hashes would both have to "miss" the same error simultaneously — probability 1 in 1,369.

### Why mod 37 instead of v3's mod 29?

- Larger alphabet (37 vs 29) = stronger error detection per character
- 37 is prime = guarantees all single-char and transposition detection
- No `%` or `*` in the alphabet — avoids URL-encoding issues (`%` is the escape character in URLs)
- Digits 0–9 now usable thanks to the dash separator before check characters

## 4. QR Encoding

| Property         | Value                                    |
|------------------|------------------------------------------|
| Content          | Full URL, uppercase                      |
| Encoding mode    | Alphanumeric (all chars are A–Z, 0–9, or .:/-$) |
| QR Version       | 4 (33×33 modules)                        |
| Error correction | Q (25%)                                  |
| Max capacity     | 67 alphanumeric characters               |
| Our usage        | 59–65 characters (depends on LP length)  |
| Sticker size     | 20×20mm (recommended minimum)            |
| Module size      | 20 ÷ 41 = 0.49mm (41 = 33 + 4+4 quiet zone) |

**Why upgrade from V3-M to V4-Q?**
- V3-Q only holds 47 chars — our codes need up to 65 → doesn't fit
- V4-Q holds 67 chars → fits with 2–8 chars to spare
- Q error correction (25%) vs M (15%) — meaningfully better scanning reliability, important given known issues with v1 scanning
- Module size at 20mm (0.49mm) is larger than v3 was at 15mm (0.45mm) — bigger and more robust

**Why uppercase URL?** QR alphanumeric mode only supports uppercase. Uppercase URLs work fine — browsers and servers treat domains as case-insensitive, and the server can be configured to handle uppercase paths. This keeps the QR code in a small version instead of requiring byte mode.

**Space budget breakdown (V4-Q, 67 capacity):**
- URL prefix `HTTPS://Q.SWAPPIE.FI/` = 21 characters (fixed)
- Path with min LP (5 chars) = 38 characters → total 59 (8 spare)
- Path with max LP (11 chars) = 44 characters → total 65 (2 spare)

## 5. How Each System Reads the Code

### Hemuli (internal repair scanning)

1. Scanner reads QR → gets `HTTPS://Q.SWAPPIE.FI/0010SMAPLP1003POFI-00088285-01-05-03-XY`
2. Strip URL prefix `HTTPS://Q.SWAPPIE.FI/`
3. Remaining path = LP code: `0010SMAPLP1003POFI-00088285-01-05-03-XY`
4. Validate both check characters
5. Parse fields, match LP SKU to phone model (existing logic)

**Backward compatibility:** Hemuli must also accept old-format codes (without URL prefix) during transition period.

### Consumer / inspector (EU compliance)

1. Scan QR with any phone camera → opens browser
2. Browser navigates to `https://q.swappie.fi/0010SMAPLP1003POFI-00088285-01-05-03-XY`
3. Landing page displays EU-required battery information for that SKU

### Supplier (code generation)

1. Open ASN generator tool on Swappie website
2. Enter PO number, SKU, quantities (same as today)
3. Tool generates full URLs as QR codes with 2 check characters
4. Supplier prints QR stickers and applies to batteries

## 6. Landing Page (q.swappie.fi)

The URL resolves to a web page showing EU Battery Regulation required information.

### Required information (EU 2023/1542, portable batteries)

**By Aug 2026 (labelling):**
- Battery capacity (Ah or Wh)
- Minimum average duration / expected lifetime
- Chemical composition (e.g., Li-ion)
- Separate collection symbol (crossed-out wheelie bin)
- Hazardous substance markings (Cd, Pb if applicable)

**By Feb 2027 (QR code data):**
- All of the above, plus:
- Carbon footprint declaration
- Recycled content share
- Recycling/end-of-life instructions
- Traceability data (manufacturer, batch, origin)
- EU declaration of conformity
- Waste prevention and management information

### Page behavior

- Parse the LP code from the URL path
- Look up battery data by SKU (the first 4 digits)
- Display the required information in a clean, mobile-friendly page
- Page should work without JavaScript (static HTML per SKU is fine)

### Open questions

- [ ] Where does the battery data come from? (Suppliers? Swappie internal?)
- [ ] Who builds and maintains the landing page? (Tech team? External?)
- [ ] What's the hosting setup? (Static site on q.swappie.fi? CMS?)
- [ ] Do we need per-batch data or is per-SKU sufficient?

## 7. Domain

`q.swappie.fi` — short subdomain under Swappie's existing domain.

- [ ] DNS setup needed (tech team)
- [ ] SSL certificate for HTTPS

## 8. Migration

| Phase | What happens |
|-------|-------------|
| **Now** | Spec review and approval |
| **Next** | Update ASN generator tool to produce v4 URL-format codes |
| **Next** | Set up q.swappie.fi with placeholder/basic landing page |
| **Next** | Update Hemuli to strip URL prefix and accept both old and new formats |
| **Aug 2026** | Landing page shows labelling-required data |
| **Feb 2027** | Landing page shows full QR-required data |

## 9. Changes from v3

| Aspect | v3 | v4 |
|--------|----|----|
| Format | `0010SMAPLP1003POFI-00088285-01-05-03K` | `HTTPS://Q.SWAPPIE.FI/0010-SMAPLP1003-POFI-00088285-01-05-03-XY` |
| Dashes | 3 (between PO fields) | 6 (between every block) |
| Check characters | 1 (mod 29, A–Z + $%*) | 2 (mod 37, A–Z + 0–9 + $) |
| Error detection | 97% (single-char errors: 100%) | 99.93% (single + double errors: 100%) |
| QR content | Plain text | URL |
| QR Version | 2 (25×25) | 4 (33×33) |
| Error correction | M (15%) | Q (25%) |
| QR capacity used | ~38 of 38 chars (V2-M) | 59–65 of 67 chars (V4-Q) |
| Module size | 0.45mm at 15×15mm | 0.49mm at 20×20mm |
| Sticker size | 15×15mm | 20×20mm |
| EU compliant | No | Yes |
| Web landing page | No | Yes |
