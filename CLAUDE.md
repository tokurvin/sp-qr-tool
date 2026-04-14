# SpareParts Project — Claude Code Context

## User

Toni Kurvinen — Senior Product Manager at Swappie. Was a coder in the 90s but hasn't coded since — prefers simple, jargon-free explanations. GitHub: tokurvin.

## Rules

- **NEVER** read, display, or reference contents of files containing tokens, secrets, or credentials. If debugging a token issue, ask Toni to verify it himself — troubleshoot from error messages only.

## Project: SP QR Code & ASN Generator

Swappie spare part QR codes used for LP (lot/batch) traceability in repair operations. Scanned in Hemuli during phone repair, synced to NetSuite.

**Old code format:** `1234-LTAPLP12345POFI-87654321-01-98-12` (no validation)
**New code format:** `1234LTAPLP12345POFI-87654321-01-98-12K` (check letter A-Z or $%* appended, first hyphen removed)

Check letter uses rolling hash: accumulator = (acc * 31 + char_value) mod 29, mapped to A-Z plus $%*. Prime modulus 29 guarantees detection of all single-char errors and adjacent transpositions.

Regex: `^\d{4}[A-Z][A-Z0-9]{8,10}PO[A-Z]{2}-\d{8}-\d{2}-\d{2}-\d{2}[A-Z$%*]$`

### Tools (this repo, hosted on tokurvin.github.io/sp-qr-tool)

- `index.html` — landing page
- `qr-checker.html` — validate/generate individual QR codes with check letter
- `asn-generator.html` — bulk ASN generation (PO + SKU + quantities -> Excel/CSV with QR codes)
- `asn-generator-v3.0.html` — shadow copy of v3.0 for download-bug comparison

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
