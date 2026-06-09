# SP QR Tool — Spare Parts Lot Traceability & EU Battery Compliance

**Live site:** https://tokurvin.github.io/sp-qr-tool/  
**Owner:** Toni Kurvinen (tokurvin) — Senior PM, Swappie  
**Last updated:** 2026-05-19

---

## What this is

Tools for generating and validating spare parts LOT (batch/lot) QR codes used in Swappie's battery supply chain. Codes are scanned in Hemuli (repair app) during phone repair, and linked to NetSuite/WMS for traceability.

The v4 format doubles as an EU Battery Regulation 2023/1542 compliance code — scanning the QR with any phone camera opens a Swappie-hosted landing page showing battery compliance information.

---

## Tools in this repo

| File | Status | Description |
|------|--------|-------------|
| `index.html` | Current | Landing page — links to all tools |
| `qr-checker-v4.html` | Current | Validate or generate individual v4 QR codes |
| `asn-generator-v4.0.html` | Current | Bulk-generate v4 QR codes for a full shipment (Excel/CSV) |
| `battery.html` | Current (demo) | Consumer-facing EU Battery Regulation landing page demo |
| `battery-upload.html` | Current (demo) | Supplier portal for submitting battery spec data (prototype) |
| `qr-checker.html` | Deprecated | QR checker for v3 format — never deployed to production |
| `asn-generator.html` | Deprecated | ASN generator for v3 format — never deployed to production |
| `asn-generator-v4.1.html` | Deprecated | v4.1 intermediate prototype — superseded by v4.0 |

---

## Code formats

### v4.0 — Current standard (EU Battery Regulation compliant)

```
HTTPS://SP.SWAPPIE.COM/POFI-00088285-SMAPLP1003-01-05-03-0010:BK
```

| Field | Example | Notes |
|-------|---------|-------|
| URL prefix | `HTTPS://SP.SWAPPIE.COM/` | Fixed; stripped by Hemuli/WMS before parsing |
| PO country | `POFI` | "PO" + 2-letter country code |
| PO number | `00088285` | 8-digit purchase order number |
| SKU (LP code) | `SMAPLP1003` | SM/TB/LT + APLP + 1–5 digits |
| Shipment # | `01` | 2-digit |
| Total boxes | `05` | 2-digit |
| Box # | `03` | 2-digit |
| Qty | `0010` | 4-digit, zero-padded |
| Check chars | `BK` | Two chars, colon-separated (A–Z + 0–9 + $) |

**Check character algorithm:** Rolling hash, MOD 37.  
C1 = hash(path, multiplier=31) mod 37  
C2 = hash(path, multiplier=17) mod 37  
Char values: 0–9 → 0–9, A–Z → 10–35, `-` → 36. URL prefix excluded.

**QR spec:** Version 4-Q · 33×33 modules · 25% error correction · 67 alphanumeric capacity · 20×20mm sticker

**Domain decision (2026-05-12):** `SP.SWAPPIE.COM` confirmed. Rationale: `.com` = customer-facing per Swappie convention (Antti); specific subdomain needed since `/path` prefix not feasible within QR capacity (Matthieu).

---

### v1 — In production (no tooling, phasing out)

```
1234-LTAPLP12345POFI-87654321-01-98-12
```

Plain text. No check digit. Placed on batteries by suppliers since original rollout. Hemuli and NetSuite/WMS must continue to accept this format throughout the v4 transition.

**Validation regex:** `^\d{4}-[A-Z][A-Z0-9]{8,10}[A-Z]{2}\d{8}-\d{2}-\d{2}-\d{2}$`

---

### v3 — Deprecated (never deployed)

```
1234LTAPLP12345POFI-87654321-01-98-12K
```

Single check letter (MOD 29, A–Z + $%*). Tools were built (qr-checker.html, asn-generator.html) and supplier validation was done, but rollout was paused when EU Battery Regulation requirements made clear a URL-based format was needed. Superseded by v4.

---

## Rollout status (as of 2026-05-19)

| Item | Status | Owner |
|------|--------|-------|
| Decision: v4.0 on battery + UUID QR on sales box | ✅ Done (2026-05-12) | All stakeholders |
| 3× consecutive read in Hemuli scanner | ✅ Done (2026-03-27) | Markus / DTF |
| "Installs battery" check on Support Line 1.0 | ✅ Done | Kevin |
| Supplier labeling guideline updated | ✅ Done | Iaroslav |
| Main batteries vendor validated tool | ✅ Done | Iaroslav |
| WeChat/in-app browser download bug fixed | ✅ Done (v3.1) | — |
| NetSuite/WMS regex updated | 🔄 In progress | Arto + Greenstep |
| Hemuli: v4 check validation + LP SKU↔model check | ❌ Not started | Markus / DTF |
| Supplier portal (battery data upload) | ❌ Not started | Tech team |
| Landing page (sp.swappie.com) | ❌ Not started | Tech team |
| Official hosting (move off Toni's GitHub Pages) | ❌ Not started | Tech team |
| v1 → v4 supplier transition | ⏸ Paused (EU reg clarity needed) | All |

**EU regulation deadlines:**
- **18 Aug 2026** — Physical battery labels required (capacity, lifespan, chemistry, hazard symbols)
- **18 Feb 2027** — QR code mandatory on all batteries placed on EU market

---

## Decision (2026-05-12) — What was agreed

Two QR codes, both pointing to the same Swappie-hosted landing page (`sp.swappie.com`):

1. **v4.0 QR on the battery sticker** — printed by supplier. Encodes LP traceability data in the URL. Scanned by Hemuli at repair. Also opens the EU compliance landing page for anyone who scans it.

2. **UUID QR on the sales box** — printed by Swappie at Grading, as a sticker directly on the box or on a leaflet with a sticker. Encodes a UUID that links to battery spec data registered by the supplier. New UUID issued when battery is replaced; old UUID invalidated.

---

## Key stakeholders

| Person | Role |
|--------|------|
| Toni Kurvinen | Senior PM, project owner |
| Iaroslav Ivanov (Slava) | Supply chain / procurement, supplier interface |
| Arto Leinonen | Finance / NetSuite / WMS |
| Markus Tammeoja | Hemuli / scanner tech (DTF) |
| Marko Rasa | Tech lead |
| Vineet Kulkarni | Warehousing / ops |
| Kevin Lillend | Repair line config |

---

## Version history

See [CHANGELOG.md](CHANGELOG.md) for full change log with dates and rationale.
