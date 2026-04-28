# Intune BIOS Compliance Checker — Gesamtdokumentation

**Graph-Variante v2.0 · CSV-Variante v1.1**

Dokumentation beider Varianten des Intune BIOS Compliance Checkers inklusive Vergleich, gemeinsamer Funktionen und variantenspezifischer Besonderheiten.

---

## Inhaltsverzeichnis

- [Hintergrund](#hintergrund)
- [Variantenvergleich](#variantenvergleich)
- [Gemeinsame Funktionen](#gemeinsame-funktionen)
  - [BIOS-Compliance-Prüfung](#bios-compliance-prüfung)
  - [Secure Boot CSV-Import](#secure-boot-csv-import)
  - [Summary-Kacheln](#summary-kacheln)
  - [Filter & Sortierung](#filter--sortierung)
  - [CSV-Export](#csv-export)
  - [BIOS-Datenbank](#bios-datenbank)
  - [Status-Codes](#status-codes)
- [Graph-Variante](#graph-variante)
  - [Voraussetzungen](#voraussetzungen-graph)
  - [Azure AD App-Registrierung](#azure-ad-app-registrierung)
  - [Erste Anmeldung](#erste-anmeldung)
  - [Datenabruf via Graph API](#datenabruf-via-graph-api)
  - [Fehlerbehebung Graph](#fehlerbehebung-graph)
  - [Changelog Graph](#changelog-graph)
- [CSV-Variante](#csv-variante)
  - [Voraussetzungen](#voraussetzungen-csv)
  - [Geräte-CSV exportieren](#geräte-csv-exportieren)
  - [CSV importieren & Analyse starten](#csv-importieren--analyse-starten)
  - [CSV-Parsing-Logik](#csv-parsing-logik)
  - [Fehlerbehebung CSV](#fehlerbehebung-csv)
  - [Changelog CSV](#changelog-csv)

---

## Hintergrund

Die Microsoft Secure Boot Zertifikate aus dem Jahr 2011 laufen ab Juni 2026 aus. Damit Windows-Geräte weiterhin Secure-Boot-bezogene Sicherheitsupdates erhalten können, müssen OEM-Hersteller BIOS-Updates mit den neuen 2023-Zertifikaten bereitstellen — und diese müssen auf den Geräten installiert sein.

Beide Varianten des Tools vergleichen die installierte BIOS-Version mit den von Dell und HP veröffentlichten Mindestversionen. Sie unterscheiden sich ausschliesslich in der Art, wie die Gerätedaten bezogen werden.

| Unterstützte Hersteller | Dell-Modelle | HP-Modelle | Lenovo-Modelle |
|---|---|---|---|
| 3 | ~320 | 296 | ausstehend |

> **Datenschutz:** Beide Tools laufen vollständig im Browser. Es werden keine Daten an externe Server gesendet.

---

## Variantenvergleich

| Merkmal | Graph-Variante | CSV-Variante |
|---|---|---|
| **Datei** | `intune-geraete-uebersicht.html` | `intune-bios-compliance-csv.html` |
| **Version** | v2.0 | v1.1 |
| Login erforderlich | ✅ Azure AD | ❌ Nicht nötig |
| App-Registrierung | ✅ Erforderlich | ❌ Nicht nötig |
| Admin Consent | ✅ Erforderlich | ❌ Nicht nötig |
| Datenbezug | Live via Microsoft Graph API | CSV-Export aus Intune Admin Center |
| Aktualität der Daten | Echtzeit | Stand des CSV-Exports |
| Ladezeit | 1–3 Min. (API-Requests pro Gerät) | Sofort (lokales Parsen) |
| MDE-Geräte-Filter | Serverseitig via OData Allowlist | Clientseitig via `Managed by = MDE` |
| Automatisierbar | Ja (API) | Nur manuell |
| Offline nutzbar | Nein (Graph API erforderlich) | Ja (nach CSV-Import) |
| BIOS-Compliance-Prüfung | ✅ Identisch | ✅ Identisch |
| Modellabgleich | ✅ Identisch | ✅ Identisch |
| Secure Boot CSV-Import | ✅ Ja | ✅ Ja |
| Summary-Kacheln | ✅ Identisch | ✅ Identisch |
| CSV-Export | ✅ Identisch | ✅ Identisch |

**Wann welche Variante?**

- **Graph-Variante** → wenn Echtzeit-Daten benötigt werden, regelmässige Prüfungen geplant sind oder keine CSV-Exporte erstellt werden können
- **CSV-Variante** → wenn keine App-Registrierung möglich ist, kein Azure AD Zugriff besteht, oder die Analyse ad-hoc ohne IT-Berechtigungen durchgeführt werden soll

---

## Gemeinsame Funktionen

Folgende Funktionen sind in beiden Varianten identisch implementiert.

### BIOS-Compliance-Prüfung

#### Versionsvergleich

Die aktuelle BIOS-Version wird segmentweise numerisch verglichen — nicht alphabetisch:

```js
compareVersions("01.10.00", "01.09.00")  →  1  (aktuell >= minimum ✓)
compareVersions("1.8.0",    "1.10.0")    → -1  (veraltet ✗)
compareVersions("1.10.0",   "1.10.0")   →  0  (exakt gleich ✓)
```

#### Modellabgleich (Normalisierung + Fuzzy)

Intune liefert Modellnamen oft in ausführlicher Form. Der Abgleich läuft in drei Stufen:

| Stufe | Methode | Beispiel |
|---|---|---|
| 1 — Normalisierung | Störwörter entfernen (`Notebook PC`, `Inch`, `Laptop PC`, …) | `HP EliteBook 860 16 Inch G10 Notebook PC` → `hp elitebook 860 16 g10` |
| 2 — Substring | Normalisierter DB-Schlüssel ist in Intune-Namen enthalten | `hp elitebook 840 g8` ⊂ `hp elitebook 840 g8 sonderconfig` |
| 3 — Fuzzy (Jaccard) | Wortmengen-Ähnlichkeit ≥ 60 % | `hp elitebook 840-g8` ↔ `hp elitebook 840 g8` |

Ein Tooltip auf der Mindestversions-Zelle zeigt an, welcher DB-Eintrag für den Match verwendet wurde.

#### BIOS-Versionsextraktion

HP gibt BIOS-Versionen mit der BIOS Family als Präfix zurück:

```
"V70 ver. 01.10.00"  →  "01.10.00"
"T76 Ver. 01.22.00"  →  "01.22.00"
"01.10.00"           →  "01.10.00"   (unverändert)
```

Der Rohwert aus Intune wird als Tooltip und als separate CSV-Spalte ausgegeben.

---

### Secure Boot CSV-Import

Secure Boot Status und Zertifikat-Status sind über die Graph API nicht direkt abrufbar — sie stehen nur im Intune Reporting zur Verfügung. Beide Varianten unterstützen den gleichen optionalen CSV-Import.

**Export:** Intune Admin Center → **Reports → Windows Autopatch → Windows quality updates → Reports → Secure Boot status → Exportieren**

**Import:** Per Drag & Drop oder Klick-Dialog. Verknüpfung mit Gerätedaten erfolgt automatisch über den Gerätenamen (case-insensitive).

| Spalte im Tool | Spalte in der CSV | Mögliche Werte |
|---|---|---|
| Secure Boot | Secure Boot enabled | Yes / No / Unknown |
| Zertifikat | Certificate status | Up to date / Not up to date / Not applicable |

Das Tool ist ohne CSV vollständig nutzbar — die Spalten zeigen `—` bis ein CSV importiert wurde.

---

### Summary-Kacheln

Beide Varianten zeigen bis zu fünf Kacheln oberhalb der Tabelle:

| Kachel | Inhalt | Sichtbar |
|---|---|---|
| Geräte gesamt | Anzahl geladener Windows-Geräte | Immer |
| BIOS aktuell | Geräte mit BIOS-Version ≥ Mindestversion | Immer |
| Update erforderlich | Geräte mit BIOS-Version < Mindestversion | Immer |
| Nicht geprüft | Geräte ohne DB-Eintrag oder ohne BIOS-Version | Immer |
| **UEFI Cert aktuell** | Geräte mit `Certificate status = Up to date` aus Secure Boot CSV | Nur nach CSV-Import |

---

### Filter & Sortierung

| Filter | Beschreibung |
|---|---|
| Freitextsuche | Sucht in Gerätename, Modell, BIOS-Version und Betriebssystem |
| Hersteller | Dropdown mit allen vorhandenen Hersteller-Werten |
| Modell | Dropdown mit allen vorhandenen Modell-Werten |
| BIOS-Status | Aktuell / Update erforderlich / Nicht geprüft |
| Zertifikat-Status | Up to date / Not up to date / Not applicable / Unbekannt |

Alle Spalten sind durch Klick auf den Spaltentitel sortierbar. Die Standardsortierung zeigt Geräte mit **Update erforderlich** zuerst.

---

### CSV-Export

Der Export über **CSV exportieren** umfasst immer die aktuell gefilterte Ansicht:

| Spalte | Inhalt |
|---|---|
| Gerätename | Gerätename |
| Hersteller | Herstellername |
| Modell | Modellname |
| BIOS aktuell | Extrahierte numerische BIOS-Version |
| BIOS aktuell (raw) | Rohwert (z.B. `V70 Ver. 01.10.00`) |
| BIOS Minimum | Mindestversion laut Hersteller-Datenbank |
| DB-Modell (abgeglichen) | Tatsächlich verwendeter DB-Eintrag |
| BIOS Status | Aktuell / Update nötig / — |
| Secure Boot aktiviert | Wert aus Secure Boot CSV (leer wenn nicht importiert) |
| Zertifikat Status | Wert aus Secure Boot CSV (leer wenn nicht importiert) |
| Betriebssystem | Betriebssystem |
| OS-Version | OS-Versionsnummer |

Datei wird als UTF-8 mit BOM exportiert — öffnet sich in Excel korrekt mit Umlauten.

---

### BIOS-Datenbank

Die Mindestversionen sind direkt im HTML-File hinterlegt — identisch in beiden Varianten:

| Hersteller | Quelle | Stand |
|---|---|---|
| Dell | [KB000347876](https://www.dell.com/support/kbdoc/en-us/000347876) | Februar 2026 |
| HP | [HP Support](https://support.hp.com/us-en/document/ish_13070353-13070429-16) | 2025/2026 |
| Lenovo | [HT518129](https://support.lenovo.com/us/en/solutions/ht518129) | ausstehend |

> Die Hersteller aktualisieren ihre Listen laufend. Für kritische Compliance-Entscheidungen die Quellartikel direkt konsultieren.

---

### Status-Codes

#### BIOS-Status

| Status | Bedeutung |
|---|---|
| **Aktuell** ✅ | BIOS-Version ≥ Mindestversion. Voraussetzung für Secure Boot 2023 Update erfüllt. |
| **Update erforderlich** ❌ | BIOS-Version < Mindestversion. BIOS-Update notwendig. |
| **—** | Kein DB-Eintrag oder keine BIOS-Version. Manuelle Prüfung empfohlen. |

#### Secure Boot Status (aus CSV)

| Status | Bedeutung |
|---|---|
| **Ja** ✅ | Secure Boot ist aktiviert. |
| **Nein** ❌ | Secure Boot ist deaktiviert. |
| **Unbekannt / —** | Keine Daten oder CSV nicht importiert. |

#### Zertifikat-Status (aus CSV)

| Status | Bedeutung |
|---|---|
| **Up to date** ✅ | Secure Boot 2023 Zertifikate installiert. |
| **Not up to date** ❌ | Noch 2011 Zertifikate aktiv. BIOS-Update + Windows Update erforderlich. |
| **Not applicable / —** | Nicht anwendbar oder CSV nicht importiert. |

---

## Graph-Variante

Datei: `intune-geraete-uebersicht.html` · Version: **v2.0**

### Voraussetzungen (Graph)

- Moderner Browser (Chrome, Edge, Firefox, Safari)
- Microsoft Entra ID (Azure AD) Tenant-Zugriff
- App-Registrierung mit delegierter Berechtigung `DeviceManagementManagedDevices.Read.All`
- Intune-Administrator-Account für Admin Consent

### Azure AD App-Registrierung

1. [portal.azure.com](https://portal.azure.com) → **Azure Active Directory** → **App-Registrierungen** → **Neue Registrierung**
2. Name z.B. `Intune BIOS Compliance Checker`, Kontotyp: **Nur dieses Verzeichnis**
3. Plattform: **Single-Page Application (SPA)** · Redirect URI: vollständige URL des Tools (wird im Tool angezeigt)
4. **API-Berechtigungen** → **Hinzufügen** → **Microsoft Graph** → **Delegierte Berechtigungen** → `DeviceManagementManagedDevices.Read.All`
5. **Administratorzustimmung erteilen**
6. **Anwendungs-ID (Client ID)** und **Verzeichnis-ID (Tenant ID)** kopieren

> Die Berechtigung muss als *delegierte* Berechtigung eingetragen sein — nicht als Anwendungsberechtigung.

**Redirect URI Beispiele:**
```
file:///C:/Users/Benutzername/Downloads/intune-geraete-uebersicht.html
https://tools.meinefirma.ch/intune-bios/
```

### Erste Anmeldung

1. **Tenant ID** und **Client ID** in die Felder eingeben
2. **Mit Microsoft anmelden** → Popup öffnet sich
3. Mit Intune-Lesezugriff-Account anmelden, ggf. Zustimmung bestätigen
4. Das Tool startet automatisch den Datenabruf

Gibt es noch keinen gecachten Account, wird direkt das Login-Popup geöffnet.

### Datenabruf via Graph API

**Schritt 1 — Geräteliste** (Windows + MDM-only, MDE ausgeschlossen):

```
GET /beta/deviceManagement/managedDevices
    ?$select=deviceName,manufacturer,model,operatingSystem,osVersion,id
    &$filter=operatingSystem eq 'Windows'
      and (managementAgent eq 'mdm' or managementAgent eq 'easMdm'
           or managementAgent eq 'intuneClient' ...)
    &$top=999
```

Bei > 999 Geräten werden alle Seiten via `@odata.nextLink` abgerufen.

**Schritt 2 — BIOS-Daten pro Gerät** (Einzelabruf nötig, da List-Endpunkt `hardwareInformation` leer zurückgibt):

```
GET /beta/deviceManagement/managedDevices/{id}
    ?$select=hardwareInformation
```

Requests laufen in Batches von 10 parallel. Bei 500 Geräten: ~50 Runden, Ladezeit 1–3 Minuten.

**Warum Beta-Endpunkt?** `hardwareInformation.systemManagementBIOSVersion` ist nur im `/beta`-Endpunkt verfügbar.

**Warum OData Allowlist statt `ne`?** Der OData `ne`-Operator wird für Enum-Felder wie `managementAgent` von Graph nicht unterstützt:

```
$filter=... and (managementAgent eq 'mdm' or managementAgent eq 'easMdm'
  or managementAgent eq 'intuneClient' or managementAgent eq 'easIntuneClient'
  or managementAgent eq 'configurationManagerClientMdm'
  or managementAgent eq 'configurationManagerClientMdmEas'
  or managementAgent eq 'microsoft365ManagedMdm'
  or managementAgent eq 'intuneAosp')
```

### Fehlerbehebung Graph

| Problem | Ursache | Lösung |
|---|---|---|
| Popup öffnet sich nicht / `popup_window_error` | Browser blockiert Popups | Popups für die Seite erlauben |
| `AADSTS50011`: Redirect URI Mismatch | URI weicht vom Dateipfad ab | URI aus dem Tool kopieren, in Azure als SPA eintragen |
| `Insufficient privileges` | Admin Consent fehlt | Als *delegiert* eintragen, Admin Consent erteilen |
| MDE-Geräte noch sichtbar | Alter Cache | Abmelden und neu anmelden |
| BIOS-Version überall leer | Intune nicht synchronisiert | Gerät synchronisieren, nach 15 Min. erneut versuchen |
| Status zeigt überall `—` | Modellname nicht erkannt | CSV prüfen, Spalte *DB-Modell (abgeglichen)* analysieren |
| Sehr lange Ladezeit | Viele Geräte | Normal — bei 1000+ Geräten 3–5 Min. einplanen |

### Changelog Graph

#### v2.0 — April 2026
- Fünfte Summary-Kachel **UEFI Cert aktuell** (erscheint nach Secure Boot CSV-Import)
- Secure Boot CSV-Import mit Drag & Drop nach dem Login
- Neue Spalten: *Secure Boot aktiviert*, *Zertifikat-Status*; neuer Filter nach Zertifikat-Status
- Fix: Login-Popup wurde nicht geöffnet wenn kein gecachter Account vorhanden war

#### v1.9 — April 2026
- MDE-Geräte per OData Allowlist serverseitig ausgeschlossen

#### v1.8 — April 2026
- Windows-only Filter im Graph API Request

#### v1.7 — April 2026
- Fluides Layout, `table-layout: auto`, `auto-fit` Grids

#### v1.6 — April 2026
- `extractBiosVersion()`: HP-Format `V70 ver. 01.10.00` → `01.10.00`; Rohwert-Tooltip

#### v1.5 — April 2026
- Dreistufiger Modellabgleich: Normalisierung → Substring → Fuzzy (Jaccard ≥ 60 %)

#### v1.4 — April 2026
- HP-Datenbank integriert (296 Modelle)

#### v1.3 — April 2026
- Dell-Datenbank integriert (~320 Modelle), Compliance-Spalten, numerischer Versionsvergleich

#### v1.0–1.2 — April 2026
- Initiale Version: MSAL-Login, Graph-Abruf, Filter, Sortierung, CSV-Export, Dark Mode
- Fix: `/beta`-Endpunkt; BIOS-Daten per Einzelabruf

---

## CSV-Variante

Datei: `intune-bios-compliance-csv.html` · Version: **v1.1**

### Voraussetzungen (CSV)

- Moderner Browser (Chrome, Edge, Firefox, Safari)
- Lesezugriff auf das Intune Admin Center (für CSV-Export)
- Keine App-Registrierung, kein Admin Consent, kein Token erforderlich

### Geräte-CSV exportieren

**Intune Admin Center → Geräte → Alle Geräte → Exportieren**

Der Export wird als ZIP bereitgestellt. Die enthaltene CSV direkt ins Tool laden.

Das Tool liest folgende Spalten:

| Spalte | Verwendung | Pflicht |
|---|---|---|
| `Device name` | Gerätename (Join-Key für Secure Boot CSV) | ✅ Ja |
| `Manufacturer` | Hersteller für BIOS-DB-Lookup | ✅ Ja |
| `Model` | Modellname für BIOS-DB-Lookup | ✅ Ja |
| `OS` | Filter: nur Windows-Geräte | ⚠️ Empfohlen |
| `OS version` | Anzeige in Tabelle und Export | Nein |
| `SystemManagementBIOSVersion` | Installierte BIOS-Version | Nein* |
| `Managed by` | Filter: MDE-Geräte (`MDE`) werden ausgeschlossen | Nein |

*Ohne BIOS-Version ist keine Compliance-Prüfung möglich.

> **Hardwareinventar muss aktiviert sein:** Die Spalte `SystemManagementBIOSVersion` ist nur befüllt wenn das Hardwareinventar für die Geräte in Intune aktiviert ist und die Geräte die Daten bereits gemeldet haben.

### CSV importieren & Analyse starten

1. **Geräte-CSV** in die linke Drop-Zone laden (Drag & Drop oder Klick)
2. **Secure Boot CSV** optional in die rechte Drop-Zone laden
3. **Analyse starten** klicken → Tabelle und Kacheln erscheinen
4. Mit **Zurücksetzen** alle Daten löschen und neu beginnen

Beide CSVs können in beliebiger Reihenfolge importiert werden. Wird die Secure Boot CSV nach der Analyse nachgeladen, aktualisiert sich die Tabelle automatisch.

### CSV-Parsing-Logik

Das Tool verwendet einen vollständigen RFC 4180-kompatiblen Parser. Beim Parsen werden automatisch gefiltert:

- **Nicht-Windows-Geräte** — Spalte `OS` ≠ `Windows` (case-insensitive)
- **MDE-only Geräte** — Spalte `Managed by` = `MDE`

Die Spalte `SystemManagementBIOSVersion` wird über eine Suche nach dem Schlüsselwort `bios` im Header erkannt — unabhängig von der genauen Gross-/Kleinschreibung.

### Fehlerbehebung CSV

| Problem | Ursache | Lösung |
|---|---|---|
| „Analyse starten" bleibt inaktiv | Geräte-CSV noch nicht importiert | Zuerst die Geräte-CSV laden |
| „Erforderliche Spalten nicht gefunden" | Falsche CSV-Datei | CSV aus *Geräte → Alle Geräte → Exportieren* verwenden |
| 0 Geräte nach Import | Alle gefiltert | `OS`- und `Managed by`-Spalten in der CSV prüfen |
| BIOS-Version überall leer | Hardwareinventar nicht synchronisiert | Geräte in Intune synchronisieren, neuen Export erstellen |
| Status zeigt überall `—` | Modellname nicht erkannt | Spalte *DB-Modell (abgeglichen)* im Export analysieren |
| Secure Boot Spalten zeigen `—` | CSV nicht importiert oder Namensmismatch | Secure Boot CSV importieren; Gerätenamen vergleichen |

### Changelog CSV

#### v1.1 — April 2026
- MDE-Geräte werden anhand `Managed by = MDE` beim Parsen herausgefiltert
- Drop-Zonen Layout überarbeitet: Texte und Kästen in separaten Zeilen
- Diverse Syntaxfixes

#### v1.0 — April 2026
- Initiale Version: vollständig CSV-basiert, kein Login erforderlich
- Geräte-CSV Import (Intune Alle Geräte Export)
- Secure Boot CSV Import (optional)
- Komplette BIOS-Compliance-Engine, Tabelle, Filter, Sortierung, Summary-Kacheln, CSV-Export
- Automatische Filterung von Nicht-Windows-Geräten
