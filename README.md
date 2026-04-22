# Intune BIOS Compliance Checker — Documentation

**Version 2.0**

A browser-based tool for reviewing all Windows devices in Microsoft Intune, comparing installed BIOS versions against the minimum versions published by Dell and HP for the **Windows Secure Boot 2023 certificate update** (expiring June 2026). Optionally extended with Secure Boot and certificate status via CSV import from the Intune portal.

---

## Table of Contents

- [What is this tool?](#what-is-this-tool)
- [Requirements](#requirements)
- [Azure AD App Registration](#azure-ad-app-registration)
- [First sign-in](#first-sign-in)
- [Data retrieval](#data-retrieval)
- [BIOS compliance check](#bios-compliance-check)
- [Secure Boot CSV import](#secure-boot-csv-import)
- [Filters & sorting](#filters--sorting)
- [CSV export](#csv-export)
- [BIOS database](#bios-database)
- [Status codes](#status-codes)
- [Graph API reference](#graph-api-reference)
- [Troubleshooting](#troubleshooting)
- [Changelog](#changelog)

---

## What is this tool?

The Microsoft Secure Boot certificates issued in 2011 begin expiring in June 2026. For Windows devices to continue receiving Secure Boot-related security updates, OEM vendors must ship BIOS updates containing the new 2023 certificates — and those updates must be installed on devices.

This tool reads all Windows devices from Intune via the Microsoft Graph API, fetches the BIOS version for each device individually, and automatically compares it against the published minimum versions. The result is a filterable, exportable compliance overview.

| Supported vendors | Dell models | HP models | Lenovo models |
|---|---|---|---|
| 3 | ~320 | 296 | pending |

> **Privacy:** The tool runs entirely in the browser. No data is sent to external servers. Tenant ID and Client ID are stored only in the browser's `sessionStorage` and discarded when the tab is closed.

---

## Requirements

- Modern browser (Chrome, Edge, Firefox, Safari)
- Access to a Microsoft Entra ID (Azure AD) tenant
- Permission to create an app registration (or an existing one)
- The app registration requires the delegated permission `DeviceManagementManagedDevices.Read.All`
- An Intune administrator account for the first sign-in (admin consent)

---

## Azure AD App Registration

### Create an app

1. Navigate to [portal.azure.com](https://portal.azure.com) → **Azure Active Directory** → **App registrations** → **New registration**
2. Choose a descriptive name, e.g. `Intune BIOS Compliance Checker`. Leave the account type set to **This directory only**.
3. Platform: **Single-page application (SPA)**. URI: the full URL where the tool will be opened. The tool displays the detected URI in the configuration field automatically.
4. Copy the **Application (Client) ID** and **Directory (Tenant) ID** from the app overview page.

### Configure permissions

1. **Add a permission** → **Microsoft Graph**
2. Type: **Delegated permissions**
3. Search for `DeviceManagementManagedDevices.Read.All` and check the box
4. **Grant admin consent** (requires Intune admin role)

> **Important:** The permission must be added as a *delegated* permission — not as an application permission.

### Redirect URI

```
# Local file
file:///C:/Users/Username/Downloads/intune-device-overview.html

# Web server
https://tools.yourcompany.com/intune-bios/
```

---

## First sign-in

1. Enter **Tenant ID** and **Client ID**
2. Click **Sign in with Microsoft** — a popup window will open
3. Sign in with an Intune read-access account
4. Confirm the consent dialog if prompted on the first use
5. The tool will start loading device data automatically

If no cached account is present, the login popup opens immediately — no silent token attempt that would fail in a `file://` context.

---

## Data retrieval

### Step 1 — Device list

The tool fetches all MDM-managed Windows devices. MDE-only devices (`managementAgent = msSense`) are excluded server-side via an allowlist filter:

```
GET /beta/deviceManagement/managedDevices
    ?$select=deviceName,manufacturer,model,operatingSystem,osVersion,id
    &$filter=operatingSystem eq 'Windows'
      and (managementAgent eq 'mdm' or managementAgent eq 'easMdm' ...)
    &$top=999
```

All pages are automatically retrieved via `@odata.nextLink` for tenants with more than 999 devices.

### Step 2 — BIOS data per device

Since `hardwareInformation` is always returned empty by the list endpoint, an individual request is made for each device in batches of 10 parallel requests:

```
GET /beta/deviceManagement/managedDevices/{id}
    ?$select=hardwareInformation
```

> **Loading time:** For 500 devices this means ~50 rounds of 10 parallel requests. Loading time is typically 1–3 minutes depending on tenant size and network speed.

---

## BIOS compliance check

### Version comparison

The installed BIOS version is compared against the minimum version using numeric segment-by-segment comparison — not alphabetically:

```js
compareVersions("01.10.00", "01.09.00")  →  1  (current >= minimum ✓)
compareVersions("1.8.0",    "1.10.0")    → -1  (outdated ✗)
compareVersions("1.10.0",   "1.10.0")   →  0  (exact match ✓)
```

### Model matching (normalisation + fuzzy)

| Stage | Method | Example |
|---|---|---|
| 1 — Normalisation | Strip noise words (`Notebook PC`, `Inch`, `Laptop PC`, …) | `HP EliteBook 860 16 Inch G10 Notebook PC` → `hp elitebook 860 16 g10` |
| 2 — Substring | Normalised DB key is contained in Intune name (or vice versa) | `hp elitebook 840 g8` ⊂ `hp elitebook 840 g8 special` |
| 3 — Fuzzy (Jaccard) | Word-set similarity ≥ 60 % | `hp elitebook 840-g8` ↔ `hp elitebook 840 g8` |

### BIOS version extraction

HP returns BIOS versions in Intune with the BIOS family as a prefix:

```
"V70 ver. 01.10.00"  →  "01.10.00"
"T76 Ver. 01.22.00"  →  "01.22.00"
"01.10.00"           →  "01.10.00"   (unchanged)
```

---

## Secure Boot CSV import

Secure Boot status and certificate status are not directly accessible via the Graph API.

### Export the CSV

In the Intune Admin Center: **Reports → Windows Autopatch → Windows quality updates → Reports → Secure Boot status → Export**

### Import the CSV

After signing in, a CSV import card appears. Drop the CSV file on the drop zone or click to browse. The tool automatically joins the data by device name (case-insensitive).

The tool is fully usable without a CSV — Secure Boot and certificate columns display `—` until a CSV is imported.

### Imported columns

| Column in tool | Column in CSV | Possible values |
|---|---|---|
| Secure Boot | Secure Boot enabled | Yes / No / Unknown |
| Certificate | Certificate status | Up to date / Not up to date / Not applicable |

---

## Filters & sorting

| Filter | Description |
|---|---|
| Free text search | Searches device name, model, BIOS version and OS |
| Manufacturer | Dropdown of all present manufacturer values |
| Model | Dropdown of all present model values |
| BIOS status | Up to date / Update required / Unknown |
| Certificate status | Up to date / Not up to date / Not applicable / Unknown |

All columns are sortable by clicking the column header. Version fields are sorted numerically. The default sort shows **Update required** devices first.

---

## CSV export

| Column | Content |
|---|---|
| Device name | Intune device name |
| Manufacturer | Manufacturer name from Intune |
| Model | Model name from Intune |
| BIOS current | Extracted numeric BIOS version |
| BIOS current (raw) | Raw value as returned by Intune |
| BIOS minimum | Minimum version from vendor database |
| DB model (matched) | DB entry actually used for the match |
| BIOS status | Up to date / Update required / — |
| Secure Boot enabled | Value from imported CSV (empty if no import) |
| Certificate status | Value from imported CSV (empty if no import) |
| Operating system | OS from Intune |
| OS version | OS version number |

The file is exported as UTF-8 with BOM and opens correctly in Excel.

---

## BIOS database

| Vendor | Source | As of |
|---|---|---|
| Dell | [KB000347876](https://www.dell.com/support/kbdoc/en-us/000347876) | February 2026 |
| HP | [HP Support](https://support.hp.com/us-en/document/ish_13070353-13070429-16) | 2025/2026 |
| Lenovo | [HT518129](https://support.lenovo.com/us/en/solutions/ht518129) | pending |

> **Note:** Vendors update their lists continuously. For critical compliance decisions, consult the source articles directly.

---

## Status codes

### BIOS status

| Status | Meaning |
|---|---|
| **Up to date** ✅ | BIOS version ≥ minimum version. Device meets the prerequisite for the Secure Boot 2023 certificate update. |
| **Update required** ❌ | BIOS version < minimum version. A BIOS update is required. |
| **—** | No DB entry or no BIOS version returned by Intune. Manual review recommended. |

### Secure Boot status (from CSV)

| Status | Meaning |
|---|---|
| **Yes** ✅ | Secure Boot is enabled. |
| **No** ❌ | Secure Boot is disabled. |
| **Unknown** | No diagnostic data available from the device. |

### Certificate status (from CSV)

| Status | Meaning |
|---|---|
| **Up to date** ✅ | Secure Boot 2023 certificates are installed. |
| **Not up to date** ❌ | 2011 certificates still active. BIOS update + Windows Update required. |
| **Not applicable** | Not applicable, e.g. Secure Boot is disabled. |

---

## Graph API reference

### Endpoints used

| Endpoint | Purpose |
|---|---|
| `GET /beta/deviceManagement/managedDevices` | Device list (filtered, paginated) |
| `GET /beta/deviceManagement/managedDevices/{id}` | BIOS data for a single device |
| `POST /oauth2/v2.0/token` | Token request (via MSAL) |

### Device filtering

Since the OData `ne` operator is not supported for enum fields, the tool uses an allowlist:

```
$filter=operatingSystem eq 'Windows'
  and (managementAgent eq 'mdm'
       or managementAgent eq 'easMdm'
       or managementAgent eq 'intuneClient'
       or managementAgent eq 'easIntuneClient'
       or managementAgent eq 'configurationManagerClientMdm'
       or managementAgent eq 'configurationManagerClientMdmEas'
       or managementAgent eq 'microsoft365ManagedMdm'
       or managementAgent eq 'intuneAosp')
```

### Why the beta endpoint?

`hardwareInformation.systemManagementBIOSVersion` is only available in the `/beta` endpoint.

### Why individual requests instead of the list?

`hardwareInformation` is always returned empty by the list endpoint. Only an individual request via `/managedDevices/{id}` returns the populated object.

---

## Troubleshooting

| Problem | Cause | Solution |
|---|---|---|
| Popup does not open / `popup_window_error` | Browser blocks popups | Allow popups for this page in browser settings |
| `AADSTS50011`: Redirect URI mismatch | URI differs from actual file path | Copy URI from the tool, register in Azure as SPA |
| `Insufficient privileges` | Admin consent missing | Add as *delegated* permission, grant admin consent |
| MDE-only devices still visible | Stale cache | Sign out and sign in again |
| BIOS version empty everywhere | Intune not synced | Sync device in Intune, retry after 15 minutes |
| Status shows `—` everywhere | Model name not recognised | Check CSV export, column *DB model (matched)* |
| CSV import shows error | Wrong CSV file | Use only the Secure Boot Status CSV export from Intune |
| Very long loading time | Many devices | Expected. Allow 3–5 minutes for 1000+ devices. |

---

## Changelog

### v2.0 — April 2026
- Secure Boot CSV import: drag & drop or file picker after sign-in
- New table columns: *Secure Boot enabled* and *Certificate status*
- New filter by certificate status
- CSV export includes Secure Boot and certificate columns
- Fix: login popup was not opening when no cached account was present

### v1.9 — April 2026
- MDE-only devices (`msSense`/`microsoftSense`) excluded server-side via allowlist filter

### v1.8 — April 2026
- Windows-only filter in Graph API request

### v1.7 — April 2026
- Fluid layout, `table-layout: auto`, `auto-fit` grids

### v1.6 — April 2026
- `extractBiosVersion()`: HP format `V70 ver. 01.10.00` → `01.10.00`
- Raw value tooltip and CSV column *BIOS current (raw)*

### v1.5 — April 2026
- Three-stage model matching: normalisation → substring → fuzzy (Jaccard ≥ 60 %)

### v1.4 — April 2026
- HP database integrated (296 models)

### v1.3 — April 2026
- Dell database integrated (~320 models), compliance columns, numeric version comparison

### v1.0–1.2 — April 2026
- Initial release: MSAL login, Graph fetch, filters, sorting, CSV export, dark mode
- Fix: `/beta` endpoint for `hardwareInformation`
- Fix: BIOS data via individual request instead of list endpoint
