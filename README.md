# Intune BIOS Compliance Checker — Documentation
**CSV Mode · v1.11**

A login-free, fully CSV-based variant of the Intune BIOS Compliance Checker. Instead of fetching data live via the Microsoft Graph API, two CSV exports from the Intune Admin Center are imported. No Azure AD, no app registration, no token — the browser processes everything locally.

---

## Table of Contents
- [What is this tool?](#what-is-this-tool)
- [Comparison: CSV vs. Graph](#comparison-csv-vs-graph)
- [Requirements](#requirements)
- [Export the device CSV](#export-the-device-csv)
- [Export the Secure Boot CSV](#export-the-secure-boot-csv)
- [CSV import](#csv-import)
- [Run analysis](#run-analysis)
- [Summary tiles](#summary-tiles)
- [Filters & sorting](#filters--sorting)
- [New columns (from v1.9)](#new-columns-from-v19)
- [Unverifiable devices](#unverifiable-devices)
- [Export functions](#export-functions)
- [CSV parsing logic](#csv-parsing-logic)
- [BIOS compliance check](#bios-compliance-check)
- [BIOS database](#bios-database)
- [Status codes](#status-codes)
- [Troubleshooting](#troubleshooting)
- [Changelog](#changelog)

---

## What is this tool?

The tool checks Windows devices from Intune for BIOS compliance with respect to the **Windows Secure Boot 2023 certificate update** (expiring June 2026). It compares the installed BIOS version against the minimum versions published by Dell, HP and Lenovo.

| Vendor | DB entries |
|---|---|
| Dell | ~320 |
| HP | ~300 |
| Lenovo | ~650 |

> All data is processed exclusively locally in the browser. No data is transmitted.

---

## Comparison: CSV mode vs. Graph mode

| Feature | CSV mode | Graph mode |
|---|---|---|
| Sign-in required | ✅ No | ⚠️ Yes (Azure AD) |
| App registration | ✅ Not needed | ⚠️ Required |
| Data retrieval | CSV import (manual) | Live via Graph API |
| Data freshness | As of CSV export date | Real-time |
| Load time | Instant | 1–3 min |
| Use without IT permissions | ✅ Possible | Restricted |
| BIOS compliance check | ✅ Identical | ✅ Identical |
| Secure Boot CSV import | ✅ Yes | ✅ Yes |

---

## Requirements

- Modern browser (Chrome, Edge, Firefox, Safari)
- Read access to the Intune Admin Center (for the CSV export)
- No app registration, no admin consent, no token required

---

## Export the device CSV

1. Navigate to [intune.microsoft.com](https://intune.microsoft.com) → **Devices** → **All devices**
2. Click **Export** (top right of the device list)
3. Download the ZIP export and load the contained CSV file into the tool

> ⚠️ **Hardware inventory must be enabled.** The `SystemManagementBIOSVersion` column is only populated if hardware inventory is enabled in Intune and devices have already reported their data.

### Relevant columns

| Column | Usage | Required |
|---|---|---|
| `Device name` | Device name (join key for Secure Boot CSV) | ✅ Yes |
| `Manufacturer` | Vendor for BIOS DB lookup | ✅ Yes |
| `Model` | Model name for BIOS DB lookup | ✅ Yes |
| `OS` | Filter: Windows devices only | ⚠️ Recommended |
| `OS version` | Display in table and CSV export | No |
| `SystemManagementBIOSVersion` | Installed BIOS version | No* |
| `Managed by` | Filter: MDE devices are excluded | No |

*Without a BIOS version, no compliance check is possible — devices are shown as *Check manually*.

---

## Export the Secure Boot CSV

**Intune Admin Center** → **Reports** → **Windows Autopatch** → **Windows quality updates** → **Reports** → **Secure Boot status** → **Export**

The tool automatically links the data via device name (case-insensitive). Columns show `—` until a CSV is imported.

> ⚠️ **As of May 2026:** The Intune export only includes the base columns (`Device name`, `Secure Boot enabled`, `Firmware version`). The new columns (`Certificate status`, `Confidence level`, `Secure Boot trust configuration`, `Date last reported`) are visible in the report UI but not yet included in the export. The tool is prepared and ready.

---

## CSV import

The tool provides two drop zones:
- **Device CSV** (required) — via drag & drop or click
- **Secure Boot Status CSV** (optional) — via drag & drop or click

The following devices are automatically excluded during parsing:
- **Non-Windows devices** — column `OS` does not contain `Windows`
- **MDE-only devices** — column `Managed by` = `MDE`
- **Personal devices** — column `Ownership` = `Personal`
- **Windows Home SKU** — column `SkuFamily` contains `home`
- **VMware VMs** — manufacturer or model contains `vmware` (case-insensitive)
- **Microsoft Hyper-V/Azure VMs** — manufacturer = `Microsoft Corporation` AND model contains `Virtual Machine` (Surface devices are retained)

---

## Run analysis

After importing the device CSV, the **Run analysis** button becomes active. Clicking it:
1. Populates the filter dropdowns with available vendors and models
2. Runs the BIOS compliance check for all devices
3. Shows the results table and summary tiles

**Reset** clears all imported data and returns the tool to its initial state.

---

## Summary tiles

All tiles (except Total devices) also show the percentage share of the total:

| Tile | Content | Visible |
|---|---|---|
| Total devices | Number of loaded Windows devices | Always |
| BIOS up to date | Count + % with BIOS ≥ minimum | Always |
| Update required | Count + % with BIOS < minimum | Always |
| Not checked | Count + % without DB entry or BIOS version | Always |
| **UEFI Cert up to date** | Count + % with `Certificate status = Up to date` | Only after Secure Boot CSV import |
| **UEFI Cert update required** | Count + % with `Certificate status = Not up to date` | Only after Secure Boot CSV import |

---

## Filters & sorting

| Filter | Description |
|---|---|
| Free text search | Searches device name, model, BIOS version and OS |
| Manufacturer | Dropdown with all available vendor values |
| Model | Dropdown with all available model values |
| BIOS status | Up to date / Update required / Not checked / End of Life / **Check manually** |
| Certificate status | Up to date / Not up to date / Not applicable / Unknown |
| **Confidence Level** | High Confidence / Under Observation / No Data Observed / Temporarily Paused / Not Supported |

All columns are sortable. Default sort: *Update required* first.

---

## New columns (from v1.9)

| Column | Source | Description |
|---|---|---|
| **Confidence Level** | `Confidence level` | Microsoft's assessment for automatic/manual rollout. Values: High Confidence ✅ / Under Observation ⚠️ / No Data Observed ⚠️ / Temporarily Paused ❌ / Not Supported ❌ |
| **Secure Boot Trust** | `Secure Boot trust configuration` | Indicates whether Secure Boot Trust covers Microsoft-only or also third-party certificates |
| **Date last reported** | `Date last reported` | Date of the device's last data report |

---

## Unverifiable devices

For devices where automatic BIOS compliance checking is not possible, the tool shows `—` in both BIOS installed and BIOS minimum columns, and a **Check manually** badge in the status column:

| Device class | Detection | Badge | Action |
|---|---|---|---|
| **IdeaPad / Legion / Yoga / WinBook** | Prefix `82xx` or `83xx`, not in `LENOVO_MT` | Check manually ↗ (clickable) | Opens [HT518129 (IdeaPad section)](https://support.lenovo.com/us/en/solutions/ht518129#ideapadandothers) |
| **ThinkStation / ThinkCentre / IdeaCentre** | Prefix `30xx`, `10–13xx`, `90/91xx`, `F0xx` | Check manually ⓘ | Tooltip: desktop device, no BIOS version info |
| **All other vendors** | Model not in DB, no BIOS version | Check manually ⓘ | Tooltip: model not in database |

> **ThinkBook devices** (`21xx` and some `83xx` codes) are explicitly mapped in `LENOVO_MT` and are treated like ThinkPad devices — provided the Intune export contains the version in parentheses (e.g. `MNCN37WW (1.37)`).

---

## Export functions

An **Export ▾** dropdown provides three options:

### 📄 CSV export
Exports the currently filtered view (UTF-8 with BOM, opens correctly in Excel):

| Column | Content |
|---|---|
| Device name | From the imported device CSV |
| Manufacturer | From the imported device CSV |
| Model | From the imported device CSV |
| BIOS installed | Extracted numeric BIOS version |
| BIOS installed (raw) | Raw value (e.g. `V70 Ver. 01.10.00`) |
| BIOS minimum | Minimum version from vendor database |
| DB model (matched) | Actual DB entry used for the match |
| BIOS status | Up to date / Update required / Check manually |
| Secure Boot enabled | Value from Secure Boot CSV |
| Certificate status | Value from Secure Boot CSV |
| Confidence Level | Raw value from Secure Boot CSV |
| Secure Boot Trust | Raw value from Secure Boot CSV |
| Date last reported | Date from Secure Boot CSV |
| Operating system | From the imported device CSV |
| OS version | From the imported device CSV |

### 🌐 HTML report
Fully self-contained HTML file (`intune_bios_compliance_report.html`):
- Summary tiles as snapshot at time of export
- Full filtered device table with colour-coded badges
- Interactive sorting by clicking column headers
- Secure Boot columns only if CSV was imported — always clean without empty columns

### 🖨 PDF / Print
Browser print dialog with optimised print layout:
- Import area and footer hidden
- Summary tiles in a single row
- Compact 10px font with cell borders
- Print header with title and timestamp
- Save as PDF via browser's built-in PDF printer

---

## CSV parsing logic

### Device CSV
RFC 4180-compatible parser (quoted fields, embedded commas, line breaks). Automatic filters: Non-Windows, MDE, Personal, Home SKU, VMware VMs, Hyper-V/Azure VMs (see [CSV import](#csv-import)).

The `SystemManagementBIOSVersion` column is detected by searching for `bios` in the header — case-insensitive.

### Secure Boot CSV
RFC 4180-compatible. Join key: `Device name` (case-insensitive).

**Required:** `Device name`, `Secure Boot enabled`

**Optional (auto-detected):** `Certificate status`, `Confidence level`, `Secure Boot trust configuration`, `Date last reported`

---

## BIOS compliance check

### Version comparison

Segment-by-segment numeric comparison (not alphabetic):

```
compareVersions("01.10.00", "01.09.00")  →  1  (current ≥ minimum ✓)
compareVersions("1.8.0",    "1.10.0")    → -1  (outdated ✗)
compareVersions("1.10.0",   "1.10.0")   →  0  (exactly equal ✓)
```

### Model matching (normalisation + fuzzy)

| Stage | Method | Example |
|---|---|---|
| 1. Normalisation | Remove noise words | `HP EliteBook 860 16 inch G10 Notebook PC` → `hp elitebook 860 16 g10` |
| 2. Substring | DB key contained in Intune name | `hp elitebook 840 g8` ⊂ `hp elitebook 840 g8 special` |
| 3. Fuzzy (Jaccard) | Word-set similarity ≥ 60% | `hp elitebook 840-g8` ↔ `hp elitebook 840 g8` |

### BIOS version extraction

```
// HP: family prefix + "ver." + version
"V70 ver. 01.10.00"       →  "01.10.00"

// Lenovo ThinkPad/ThinkBook (commercial): version in parentheses
"N2XET42W (1.32 )"        →  "1.32"
"R1EET56W(1.56 )"         →  "1.56"
"F8CN59WW(V2.22)"         →  "2.22"
"MNCN37WW (1.37)"         →  "1.37"   (ThinkBook)

// Lenovo IdeaPad/Legion/Yoga (consumer): no version format
"GLCN46WW"                →  not extractable → Check manually

// Dell / generic: already numeric
"01.10.00"                →  "01.10.00"  (unchanged)
```

### Lenovo machine type detection

Intune reports **machine type codes** for Lenovo devices (e.g. `20S1SFT70Y`), not the product name. The tool uses a `LENOVO_MT` lookup table with 463 entries from [HT518129](https://support.lenovo.com/us/en/solutions/ht518129):

| Step | Action | Example |
|---|---|---|
| 1. Fuzzy match | Normal DB lookup | Fails for machine type codes |
| 2. BIOS family disambiguation | Composite key `MT_BIOSPREFIX` in `LENOVO_MT_BIOS` | `20VX_N3WET` → min 1.13 instead of 1.61 |
| 3. Machine type lookup | First 4 characters in `LENOVO_MT` | `20S1` → `t14 gen 1` → min 1.31 |
| 4. Desktop detection | Prefix `10–13xx`, `30xx`, `90/91xx`, `F0xx` | `30CE` → ThinkStation → Check manually ⓘ |
| 5. Consumer detection | Prefix `82xx` or `83xx` without MT entry | `83LK` → IdeaPad → Check manually ↗ |

**BIOS family disambiguation (P14s/P15s Gen 2):** Some models ship with two different BIOS families with different minimum versions. The tool identifies the family from the first 5 characters of the raw value: `20VX` with `N34ET` BIOS → min 1.61, with `N3WET` BIOS → min 1.13.

---

## BIOS database

Embedded directly in the HTML file:

| Vendor | Source | As of |
|---|---|---|
| Dell | [KB000347876](https://www.dell.com/support/kbdoc/en-us/000347876) | February 2026 |
| HP | [HP Support](https://support.hp.com/us-en/document/ish_13070353-13070429-16) | 2025/2026 |
| Lenovo | [HT518129](https://support.lenovo.com/us/en/solutions/ht518129) | June 2026 |

> ⚠️ Vendors update their lists continuously. For critical compliance decisions, consult the source articles directly.

---

## Status codes

### BIOS status

| Status | Meaning |
|---|---|
| ✅ **Up to date** | BIOS version ≥ minimum. Device meets Secure Boot 2023 prerequisite. |
| ❌ **Update required** | BIOS version < minimum. A BIOS update is required. |
| ⚠️ **End of Life** | No BIOS update available — compliance requirement cannot be met. |
| ⚠️ **Check manually ↗** | IdeaPad/Legion/Yoga (82xx/83xx): BIOS version not available as a numeric value. Badge is clickable → opens HT518129. BIOS fields show `—`. |
| ⚠️ **Check manually ⓘ** | Desktop devices: Intune provides no BIOS version info. Or: model not in DB / BIOS version missing. BIOS fields show `—`. |

### Certificate status (from Secure Boot CSV)

| Status | Meaning |
|---|---|
| ✅ **Up to date** | Secure Boot 2023 certificates installed. |
| ❌ **Not up to date** | 2011 certificates still active. BIOS update + Windows Update required. |
| **N/A / —** | Not applicable or Secure Boot CSV not imported. |

### Confidence Level (from Secure Boot CSV)

| Status | Meaning |
|---|---|
| ✅ **High Confidence** | Microsoft recommends automatic rollout. |
| ⚠️ **Under Observation** | Device is being monitored, rollout proceeding in controlled manner. |
| ⚠️ **No Data Observed** | No data yet — manual rollout recommended. |
| ❌ **Temporarily Paused** | Rollout paused, possible compatibility issues. |
| ❌ **Not Supported** | Device does not support the update. |
| **—** | Secure Boot CSV not imported or column not present. |

---

## Troubleshooting

| Problem | Cause | Solution |
|---|---|---|
| "Run analysis" button stays inactive | Device CSV not yet imported | Load the device CSV first |
| Error: "Required columns not found" | Wrong CSV file | Use CSV from *Devices → All devices → Export* |
| 0 devices after import | All devices were filtered out | Check `OS` and `Managed by` columns |
| BIOS version empty everywhere | Hardware inventory not synced | Sync devices in Intune, create new export |
| Status shows Check manually everywhere | Model not recognised or consumer/desktop | For Lenovo: clickable badge opens HT518129. For others: add DB entry if needed. |
| Secure Boot columns show `—` | CSV not imported or name mismatch | Import Secure Boot CSV; compare device names |
| Confidence Level/Trust/Date show `—` | Export columns not yet available | Microsoft issue (as of May 2026), tool is prepared |
| Lenovo devices all show Check manually | Machine type code not in `LENOVO_MT` | Consumer (82/83xx): clickable → HT518129. Desktop: correct. Newer models: not yet in table. |
| VMware/Azure VMs appear in table | Older CSV format | Filtered automatically when manufacturer/model contains known VM strings |

---

## Changelog

### v1.11 — June 2026
- Complete Lenovo machine type table (`LENOVO_MT`): 136 → 463 entries from Lenovo HT518129
- BIOS family disambiguation (`LENOVO_MT_BIOS`): P14s/P15s Gen 2 dual-BIOS variants correctly distinguished
- Desktop detection: ThinkStation/ThinkCentre/IdeaCentre → **Check manually ⓘ** with tooltip
- Consumer detection: IdeaPad/Legion/Yoga (82xx/83xx) → clickable **Check manually ↗** → HT518129
- For all unverifiable devices: BIOS installed + BIOS minimum show `—` instead of raw BIOS code
- VM filter: VMware VMs and Microsoft Hyper-V/Azure VMs automatically excluded during import
- 17 new BIOS DB entries (disambiguation variants, missing E-/L-/X-series)
- Lenovo BIOS extraction: bracket format `N2XET42W (1.32 )` and `F8CN59WW(V2.22)` supported

### v1.10 — May 2026
- Export dropdown replaces the single CSV button
- New export: **HTML report** — standalone, sortable HTML file with tile snapshot
- New export: **PDF / Print** — optimised `@media print` layout with print header
- HTML report adapts columns dynamically to available Secure Boot data

### v1.9 — May 2026
- Three new table columns from Secure Boot CSV: **Confidence Level**, **Secure Boot Trust**, **Date last reported**
- New filter: **Confidence Level**
- Percentage share on all summary tiles
- New tile: **UEFI Cert update required**
- Colour-coded badges for Confidence Level
- Fix: Certificate status showed "Unknown" when `Certificate status` column missing from Secure Boot CSV
- Fix: `sortFiltered()` crash on non-string return values
- Fix: Several duplicate function declarations cleaned up

### v1.1 — April 2026
- MDE-only devices excluded during parsing based on `Managed by = MDE` column
- Drop zone layout revised
- Various syntax fixes

### v1.0 — April 2026
- Initial release: fully CSV-based, no sign-in required
- Device CSV import, Secure Boot CSV import (optional)
- Complete BIOS compliance engine (database, normalisation, fuzzy match, HP extraction)
- Automatic filtering of non-Windows devices
