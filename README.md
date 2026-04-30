# Intune BIOS Compliance Checker — Complete Documentation

**Graph variant v2.3 · CSV variant v1.8**

Documentation for both variants of the Intune BIOS Compliance Checker, including a comparison, shared features, and variant-specific details.

---

## Table of Contents

- [Background](#background)
- [Variant comparison](#variant-comparison)
- [Shared features](#shared-features)
  - [BIOS compliance check](#bios-compliance-check)
  - [Secure Boot CSV import](#secure-boot-csv-import)
  - [Summary tiles](#summary-tiles)
  - [Filters & sorting](#filters--sorting)
  - [CSV export](#csv-export)
  - [BIOS database](#bios-database)
  - [Status codes](#status-codes)
- [Graph variant](#graph-variant)
  - [Requirements](#requirements-graph)
  - [Azure AD app registration](#azure-ad-app-registration)
  - [First sign-in](#first-sign-in)
  - [Data retrieval via Graph API](#data-retrieval-via-graph-api)
  - [Troubleshooting Graph](#troubleshooting-graph)
  - [Changelog Graph](#changelog-graph)
- [CSV variant](#csv-variant)
  - [Requirements](#requirements-csv)
  - [Export the device CSV](#export-the-device-csv)
  - [Import CSV & run analysis](#import-csv--run-analysis)
  - [CSV parsing logic](#csv-parsing-logic)
  - [Troubleshooting CSV](#troubleshooting-csv)
  - [Changelog CSV](#changelog-csv)

---

## Background

The Microsoft Secure Boot certificates issued in 2011 begin expiring in June 2026. For Windows devices to continue receiving Secure Boot-related security updates, OEM vendors must ship BIOS updates containing the new 2023 certificates — and those updates must be installed on devices.

Both variants of the tool compare the installed BIOS version against the minimum versions published by Dell and HP. They differ only in how device data is retrieved.

| Supported vendors | Dell models | HP models | Lenovo models | Microsoft models |
|---|---|---|---|
| 4 | 457 | 305 | 630 | 35 |

> **Privacy:** Both tools run entirely in the browser. No data is transmitted to external servers.

---

## Variant comparison

| Feature | Graph variant | CSV variant |
|---|---|---|
| **File** | `intune-device-overview.html` | `intune-bios-compliance-csv.html` |
| **Version** | v2.3 | v1.8 |
| Sign-in required | ✅ Azure AD | ❌ Not required |
| App registration | ✅ Required | ❌ Not required |
| Admin consent | ✅ Required | ❌ Not required |
| Data source | Live via Microsoft Graph API | CSV export from Intune Admin Center |
| Data freshness | Real-time | As of the CSV export date |
| Loading time | 1–3 min (API request per device) | Instant (local parsing) |
| MDE device filter | Server-side via OData allowlist | Client-side via `Managed by = MDE` |
| Automatable | Yes (API) | Manual only |
| Works offline | No (Graph API required) | Yes (after CSV import) |
| BIOS compliance check | ✅ Identical | ✅ Identical |
| Model matching | ✅ Identical | ✅ Identical |
| Secure Boot CSV import | ✅ Yes | ✅ Yes |
| Summary tiles | ✅ Identical | ✅ Identical |
| CSV export | ✅ Identical | ✅ Identical |

**When to use which variant?**

- **Graph variant** → when real-time data is needed, regular checks are planned, or no CSV exports can be created
- **CSV variant** → when no app registration is possible, Azure AD access is unavailable, or an ad-hoc analysis without IT permissions is required

---

## Shared features

The following features are implemented identically in both variants.

### BIOS compliance check

#### Version comparison

The installed BIOS version is compared against the minimum version using numeric segment-by-segment comparison — not alphabetically:

```js
compareVersions("01.10.00", "01.09.00")  →  1  (current >= minimum ✓)
compareVersions("1.8.0",    "1.10.0")    → -1  (outdated ✗)
compareVersions("1.10.0",   "1.10.0")   →  0  (exact match ✓)
```

#### Model matching (normalisation + fuzzy)

Intune often returns verbose model names. The matching runs in three stages:

| Stage | Method | Example |
|---|---|---|
| 1 — Normalisation | Strip noise words (`Notebook PC`, `Inch`, `Laptop PC`, …) | `HP EliteBook 860 16 Inch G10 Notebook PC` → `hp elitebook 860 16 g10` |
| 2 — Substring | Normalised DB key is contained in Intune name (or vice versa) | `hp elitebook 840 g8` ⊂ `hp elitebook 840 g8 special` |
| 3 — Fuzzy (Jaccard) | Word-set similarity ≥ 60 % | `hp elitebook 840-g8` ↔ `hp elitebook 840 g8` |

A tooltip on the minimum version cell shows which DB entry was used for the match.

#### BIOS version extraction

HP returns BIOS versions with the BIOS family as a prefix:

```
"V70 ver. 01.10.00"  →  "01.10.00"
"T76 Ver. 01.22.00"  →  "01.22.00"
"01.10.00"           →  "01.10.00"   (unchanged)
```

The raw value is shown as a tooltip and exported as a separate CSV column.

---

### Secure Boot CSV import

Secure Boot status and certificate status are not directly accessible via the Graph API — they are only available in Intune Reporting. Both variants support the same optional CSV import.

**Export:** Intune Admin Center → **Reports → Windows Autopatch → Windows quality updates → Reports → Secure Boot status → Export**

**Import:** Drag & drop or click to browse. The tool automatically joins data by device name (case-insensitive).

| Column in tool | Column in CSV | Possible values |
|---|---|---|
| Secure Boot | Secure Boot enabled | Yes / No / Unknown |
| Certificate | Certificate status | Up to date / Not up to date / Not applicable |

The tool is fully usable without a CSV — columns show `—` until a CSV is imported.

---

### Summary tiles

Both variants display up to five tiles above the table:

| Tile | Content | Visible |
|---|---|---|
| Total devices | Number of Windows devices loaded | Always |
| BIOS up to date | Devices with BIOS version ≥ minimum version | Always |
| Update required | Devices with BIOS version < minimum version | Always |
| Not checked | Devices without a DB entry or without a BIOS version | Always |
| **UEFI Cert up to date** | Devices with `Certificate status = Up to date` from Secure Boot CSV | After CSV import only |

---

### Filters & sorting

| Filter | Description |
|---|---|
| Free text search | Searches device name, model, BIOS version and OS |
| Manufacturer | Dropdown of all present manufacturer values |
| Model | Dropdown of all present model values |
| BIOS status | Up to date / Update required / Not checked |
| Certificate status | Up to date / Not up to date / Not applicable / Unknown |

All columns are sortable by clicking the column header. The default sort shows **Update required** devices first.

---

### CSV export

The **Export CSV** button saves the currently filtered view:

| Column | Content |
|---|---|
| Device name | Device name |
| Manufacturer | Manufacturer name |
| Model | Model name |
| Current BIOS | Extracted numeric BIOS version |
| Current BIOS (raw) | Raw value (e.g. `V70 Ver. 01.10.00`) |
| Minimum BIOS | Minimum version from vendor database |
| DB model (matched) | DB entry actually used for the match |
| BIOS status | Up to date / Update required / — |
| Secure Boot enabled | Value from Secure Boot CSV (empty if not imported) |
| Certificate status | Value from Secure Boot CSV (empty if not imported) |
| Operating system | Operating system |
| OS version | OS version number |

File is exported as UTF-8 with BOM — opens correctly in Excel.

---

### BIOS database

The minimum versions are embedded directly in the HTML file — identical in both variants:

| Vendor | Source | As of |
|---|---|---|
| Dell | [KB000347876](https://www.dell.com/support/kbdoc/en-us/000347876) | April 2026 |
| HP | [HP Support](https://support.hp.com/us-en/document/ish_13070353-13070429-16) | April 2026 |
| Lenovo | [HT518129](https://support.lenovo.com/us/en/solutions/ht518129) | April 2026 |
| Microsoft | [Surface Support](https://support.microsoft.com/en-us/surface/drivers-firmware/surface-secure-boot-certificates) | April 2026 |

> Vendors update their lists continuously. For critical compliance decisions, consult the source articles directly.

---

### Status codes

#### BIOS status

| Status | Meaning |
|---|---|
| **Up to date** ✅ | BIOS version ≥ minimum version. Prerequisite for Secure Boot 2023 update met. |
| **Update required** ❌ | BIOS version < minimum version. BIOS update required. |
| **—** | No DB entry or no BIOS version. Manual review recommended. |

#### Secure Boot status (from CSV)

| Status | Meaning |
|---|---|
| **Yes** ✅ | Secure Boot is enabled. |
| **No** ❌ | Secure Boot is disabled. |
| **Unknown / —** | No data available or CSV not imported. |

#### Certificate status (from CSV)

| Status | Meaning |
|---|---|
| **Up to date** ✅ | Secure Boot 2023 certificates installed. |
| **Not up to date** ❌ | 2011 certificates still active. BIOS update + Windows Update required. |
| **Not applicable / —** | Not applicable or CSV not imported. |

---

## Graph variant

File: `intune-device-overview.html` · Version: **v2.3**

### Requirements (Graph)

- Modern browser (Chrome, Edge, Firefox, Safari)
- Microsoft Entra ID (Azure AD) tenant access
- App registration with delegated permission `DeviceManagementManagedDevices.Read.All`
- Intune administrator account for admin consent

### Azure AD app registration

1. [portal.azure.com](https://portal.azure.com) → **Azure Active Directory** → **App registrations** → **New registration**
2. Name e.g. `Intune BIOS Compliance Checker`, account type: **This directory only**
3. Platform: **Single-page application (SPA)** · Redirect URI: full URL of the tool (displayed in the tool)
4. **API permissions** → **Add** → **Microsoft Graph** → **Delegated permissions** → `DeviceManagementManagedDevices.Read.All`
5. **Grant admin consent**
6. Copy the **Application (Client) ID** and **Directory (Tenant) ID**

> The permission must be added as a *delegated* permission — not as an application permission.

**Redirect URI examples:**
```
file:///C:/Users/Username/Downloads/intune-device-overview.html
https://tools.yourcompany.com/intune-bios/
```

### First sign-in

1. Enter **Tenant ID** and **Client ID**
2. **Sign in with Microsoft** → popup opens
3. Sign in with an Intune read-access account, confirm consent if prompted
4. The tool starts loading data automatically

If no cached account is present, the login popup opens immediately.

### Data retrieval via Graph API

**Step 1 — Device list** (Windows + MDM-only, MDE excluded):

```
GET /beta/deviceManagement/managedDevices
    ?$select=deviceName,manufacturer,model,operatingSystem,osVersion,id
    &$filter=operatingSystem eq 'Windows'
      and (managementAgent eq 'mdm' or managementAgent eq 'easMdm'
           or managementAgent eq 'intuneClient' ...)
    &$top=999
```

For > 999 devices, all pages are retrieved via `@odata.nextLink`.

**Step 2 — BIOS data per device** (individual request required, list endpoint returns `hardwareInformation` empty):

```
GET /beta/deviceManagement/managedDevices/{id}
    ?$select=hardwareInformation
```

Requests run in batches of 10 parallel. For 500 devices: ~50 rounds, loading time 1–3 minutes.

**Why the beta endpoint?** `hardwareInformation.systemManagementBIOSVersion` is only available in the `/beta` endpoint.

**Why OData allowlist instead of `ne`?** The OData `ne` operator is not supported for enum fields like `managementAgent`:

```
$filter=... and (managementAgent eq 'mdm' or managementAgent eq 'easMdm'
  or managementAgent eq 'intuneClient' or managementAgent eq 'easIntuneClient'
  or managementAgent eq 'configurationManagerClientMdm'
  or managementAgent eq 'configurationManagerClientMdmEas'
  or managementAgent eq 'microsoft365ManagedMdm'
  or managementAgent eq 'intuneAosp')
```

### Troubleshooting Graph

| Problem | Cause | Solution |
|---|---|---|
| Popup does not open / `popup_window_error` | Browser blocks popups | Allow popups for this page |
| `AADSTS50011`: Redirect URI mismatch | URI differs from file path | Copy URI from tool, register in Azure as SPA |
| `Insufficient privileges` | Admin consent missing | Add as *delegated*, grant admin consent |
| MDE devices still visible | Stale cache | Sign out and sign in again |
| BIOS version empty everywhere | Intune not synced | Sync device, retry after 15 minutes |
| Status shows `—` everywhere | Model name not recognised | Check CSV export, analyse *DB model (matched)* column |
| Very long loading time | Many devices | Expected — allow 3–5 min for 1000+ devices |

### Changelog Graph

#### v2.3 — April 2026
- BIOS database: **Microsoft/Surface** added (35 models, `any` and `End of Life` logic)
- BIOS database: **Lenovo** added (630 entries — ThinkPad, ThinkCentre, ThinkStation, ThinkBook)
- BIOS database: **Dell** expanded to 457 entries
- BIOS database: **HP** expanded to 305 entries
- New status **End of Life**: badge, filter option and CSV column
- New status **Factory default**: Surface devices shipped with the 2023 CA pre-installed

#### v2.0 — April 2026
- Fifth summary tile **UEFI Cert up to date** (appears after Secure Boot CSV import)
- Secure Boot CSV import with drag & drop after sign-in
- New columns: *Secure Boot enabled*, *Certificate status*; new filter by certificate status
- Fix: login popup was not opening when no cached account was present

#### v1.9 — April 2026
- MDE devices excluded server-side via OData allowlist

#### v1.8 — April 2026
- Windows-only filter in Graph API request

#### v1.7 — April 2026
- Fluid layout, `table-layout: auto`, `auto-fit` grids

#### v1.6 — April 2026
- `extractBiosVersion()`: HP format `V70 ver. 01.10.00` → `01.10.00`; raw value tooltip

#### v1.5 — April 2026
- Three-stage model matching: normalisation → substring → fuzzy (Jaccard ≥ 60 %)

#### v1.4 — April 2026
- HP database integrated (296 models)

#### v1.3 — April 2026
- Dell database integrated (~320 models), compliance columns, numeric version comparison

#### v1.0–1.2 — April 2026
- Initial release: MSAL login, Graph fetch, filters, sorting, CSV export, dark mode
- Fix: `/beta` endpoint; BIOS data via individual request

---

## CSV variant

File: `intune-bios-compliance-csv.html` · Version: **v1.8**

### Requirements (CSV)

- Modern browser (Chrome, Edge, Firefox, Safari)
- Read access to the Intune Admin Center (for CSV export)
- No app registration, no admin consent, no token required

### Export the device CSV

**Intune Admin Center → Devices → All devices → Export**

The export is provided as a ZIP file. Extract the CSV and load it directly into the tool.

The tool reads the following columns:

| Column | Usage | Required |
|---|---|---|
| `Device name` | Device name (join key for Secure Boot CSV) | ✅ Yes |
| `Manufacturer` | Vendor for BIOS DB lookup | ✅ Yes |
| `Model` | Model name for BIOS DB lookup | ✅ Yes |
| `OS` | Filter: Windows devices only | ⚠️ Recommended |
| `OS version` | Displayed in table and export | No |
| `SystemManagementBIOSVersion` | Installed BIOS version | No* |
| `Managed by` | Filter: MDE devices (`MDE`) are excluded | No |

*Without a BIOS version, no compliance check is possible.

> **Hardware inventory must be enabled:** The `SystemManagementBIOSVersion` column is only populated if hardware inventory is enabled for devices in Intune and the devices have already reported the data.

### Import CSV & run analysis

1. Load the **device CSV** into the left drop zone (drag & drop or click)
2. Optionally load the **Secure Boot CSV** into the right drop zone
3. Click **Run analysis** → table and tiles appear
4. Use **Reset** to clear all data and start over

Both CSVs can be imported in any order. If the Secure Boot CSV is loaded after the analysis, the table updates automatically.

### CSV parsing logic

The tool uses a fully RFC 4180-compliant parser. The following are automatically filtered during parsing:

- **Non-Windows devices** — column `OS` ≠ `Windows` (case-insensitive)
- **MDE-only devices** — column `Managed by` = `MDE`

The `SystemManagementBIOSVersion` column is identified by searching for the keyword `bios` in the header — regardless of exact capitalisation.

### Troubleshooting CSV

| Problem | Cause | Solution |
|---|---|---|
| "Run analysis" stays inactive | Device CSV not yet imported | Load the device CSV first |
| "Required columns not found" | Wrong CSV file | Use CSV from *Devices → All devices → Export* |
| 0 devices after import | All filtered out | Check `OS` and `Managed by` columns in the CSV |
| BIOS version empty everywhere | Hardware inventory not synced | Sync devices in Intune, create a new export |
| Status shows `—` everywhere | Model name not recognised | Analyse *DB model (matched)* column in the export |
| Secure Boot columns show `—` | CSV not imported or name mismatch | Import Secure Boot CSV; compare device names |

### Changelog CSV

#### v1.8 — April 2026
- MDE-only devices excluded during parsing based on `Managed by = MDE`
- Drop zone layout revised: texts and drop zones in separate rows
- Various syntax fixes

#### v1.0 — April 2026
- Initial release: fully CSV-based, no sign-in required
- Device CSV import (Intune All Devices export)
- Secure Boot CSV import (optional)
- Complete BIOS compliance engine, table, filters, sorting, summary tiles, CSV export
- Automatic filtering of non-Windows devices
