# EU Battery Regulation — Data Flow Decisions

**Status:** Design draft (prototype implemented at https://tokurvin.github.io/sp-qr-tool/)
**Last updated:** 2026-05-12
**Owner:** Toni Kurvinen

This document captures the design decisions made for getting supplier battery data into Swappie's system and out to end users via QR codes, in compliance with **EU Battery Regulation 2023/1542** (QR deadline 18 Feb 2027).

---

## TL;DR — The end-to-end flow

```
   ┌──────────────┐                                 ┌──────────────┐
   │  SUPPLIER    │                                 │   SWAPPIE    │
   └──────┬───────┘                                 └──────┬───────┘
          │                                                │
   1. Register battery data                                │
      at battery-upload.html                               │
      → Supplier ID + SKU + all EU fields                  │
      → System assigns a NEW UUID per submission           │
      → Supplier prints UUID URL on box / leaflet          │
          │                                                │
          │           ──── data submission ────►           │
          │                                                │
          │                                         2. Swappie issues PO
          │                                            for SKU + qty
          │           ◄──── PO + ASN sheet ────         (ASN generator
          │                                             writes PO + SKU
          │                                             → UUID mapping
          │                                             to DB)
          │                                                │
   3. Supplier prints QR labels                            │
      using asn-generator-v4.0.html                        │
      (one QR per battery, encodes PO+SKU+batch)           │
          │                                                │
          │           ──── ships batteries ────►           │
          │                                                │
                                                   4. End user / repair tech
                                                      scans QR or visits
                                                      UUID URL → lands on
                                                      battery.html
                                                      → sees EU-compliant
                                                        data sheet
```

Two ways to reach the data sheet:

- **Path A — Direct UUID** (`https://qr.swappie.com/{uuid}`)
  Printed on retail box or supplier leaflet. Resolves directly to the data record.

- **Path B — Battery sticker QR** (`https://qr.swappie.com/{PO}-{SKU}-{shp#}-{boxes}-{box#}-{qty}:{C1}{C2}`)
  Scanned from the battery itself. Server looks up PO+SKU → UUID → same data sheet, plus batch context (shipment, box, position).

Both paths render from the same underlying record. Path B additionally shows traceability data (PO, batch, supplier-derived-from-PO).

---

## Key design decisions

### 1. UUID is the version, not a permanent product ID

**Decision:** Each time a supplier submits/updates battery data for a (Supplier × SKU) combination, the system issues a **new UUID**. Old UUIDs and the QR codes referencing them remain valid and continue to resolve to the data version they were issued with.

**Why:** If UUID were permanent and the QR code didn't carry a version number, every old QR in the field would silently start serving newest data — destroying historical traceability. Regenerating the UUID per version means the QR code itself doesn't need to know about versions; the UUID it resolves to does.

**Implication:** Swappie's PO→UUID mapping is locked in at the moment the ASN is generated. After that the binding is immutable.

### 2. (Supplier × SKU) is the logical product key — not SKU alone

**Decision:** The same Swappie SKU can be sourced from multiple suppliers, and each supplier's variant may have different chemistry, capacity, or carbon footprint. Battery records are therefore keyed on **(Supplier ID, SKU)** logically; UUID is the physical identifier issued against that pair.

**Why:** Suppliers cannot overwrite each other. The data sheet shown for a given battery must reflect *that specific supplier's* product, not a generic SKU-level average.

### 3. PO anchors the supplier — supplier not encoded in QR

**Decision:** The v4 QR code carries PO, SKU, shipment, box, position, quantity — but not supplier or UUID. Swappie's internal DB resolves `PO → supplier`, and `(supplier, SKU) → UUID for that version`.

**Why:** Keeps the QR payload within QR Version 2 budget. Supplier identity is already implicit in the PO (Swappie's procurement system knows which supplier each PO went to).

### 4. Versioning: Swappie auto-increments; supplier adds notes

**Decision:** When a supplier loads a previous version as a template and submits, the system auto-assigns Version N+1. Supplier provides a free-text "Version notes" field describing what changed (e.g. *"Changed cathode from LiCoO₂ to NMC; new production facility from March 2026"*).

**Why:** Internal version numbers are reliable and don't depend on supplier discipline; the free-text note captures the *why* for auditability.

### 5. Two paths, one data sheet

**Decision:** The `battery.html` landing page detects whether the URL fragment is a UUID or a v4 QR code and renders accordingly:

- **UUID fragment** → "Direct Access" view. Shows SKU, Supplier ID, Data version, and all EU-required spec sections. No batch context (because the UUID can be on a box that contains many units).
- **v4 QR fragment** → "Traceability" view. Shows the same EU data plus PO, shipment number, box number, position-in-box, quantity.

**Why:** Same underlying record serves both consumer-facing (box/leaflet) and operational (battery sticker) use cases without duplicating data.

### 6. UUID gate in ASN generator

**Decision:** The ASN generator blocks Excel/CSV download unless every SKU row in the ASN has a valid UUID entered. Suppliers must therefore have submitted battery data *before* they can ship.

**Why:** Prevents shipments where the QR codes would resolve to "Record not found." Forces the data registration step to happen first.

---

## Current prototype state

| File | Purpose |
|---|---|
| `battery-upload.html` | Supplier portal — register EU battery data, get UUID |
| `battery.html` | Landing page for QR scans + direct UUID URLs |
| `asn-generator-v4.0.html` | Bulk QR/label generation, UUID-gated |
| `spec-eu-qr-v4.md` | v4 QR code format spec |

Hosted on GitHub Pages: https://tokurvin.github.io/sp-qr-tool/

Demo UUID for testing: `f47ac10b-58cc-4372-a567-0e02b2c3d479`

---

## Open questions

- **Responsible economic operator under EU 2023/1542?** Swappie as importer is likely on the hook, but needs legal confirmation.
- **Scope:** Does the QR/labelling obligation apply to batteries inside refurbished phones, or only to standalone spare-part batteries?
- **Supplier compliance:** Do current suppliers already produce EU-compliant data, or will Swappie need to push them?
- **Landing page hosting:** Currently on `tokurvin.github.io`. Production needs `qr.swappie.com` on Swappie infra with a real backend and DB.
- **Authentication:** Supplier portal currently has none. Production needs supplier login so Supplier A cannot overwrite Supplier B's record for the same SKU.
- **Supplier ID format:** What does Swappie use in NetSuite? The portal currently accepts free text (`SW-SUP-0042` as placeholder).
- **UUID printing on retail box:** Plain text URL, or also as a small QR? Probably both for UX.
- **Coexistence with LP QR:** The LP traceability QR (v3 / v4) and the EU Battery QR could share the same label or be separate stickers. Decision pending.

---

## Related decisions from the LP QR rollout (paused 2026-04-14)

The LP QR work was paused specifically to figure out how it interacts with EU Battery Reg before continuing. The v4 design above is the merged answer: **one QR carrying LP traceability data, resolving to a landing page that also satisfies the EU consumer-data obligation.**
