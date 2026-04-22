# Intune BIOS Compliance Checker — Dokumentation

**Version 1.8**

Ein browserbasiertes Tool zur Übersicht aller Windows-Geräte in Microsoft Intune mit Abgleich der aktuellen BIOS-Version gegen die von Dell und HP veröffentlichten Mindestversionen für das **Windows Secure Boot 2023 Zertifikats-Update** (Ablauf Juni 2026).

---

## Inhaltsverzeichnis

- [Was ist das Tool?](#was-ist-das-tool)
- [Voraussetzungen](#voraussetzungen)
- [Azure AD App-Registrierung](#azure-ad-app-registrierung)
- [Erste Anmeldung](#erste-anmeldung)
- [Datenabruf](#datenabruf)
- [BIOS-Compliance-Prüfung](#bios-compliance-prüfung)
- [Filter & Sortierung](#filter--sortierung)
- [CSV-Export](#csv-export)
- [BIOS-Datenbank](#bios-datenbank)
- [Status-Codes](#status-codes)
- [Graph API Referenz](#graph-api-referenz)
- [Fehlerbehebung](#fehlerbehebung)
- [Changelog](#changelog)

---

## Was ist das Tool?

Die Microsoft Secure Boot Zertifikate aus dem Jahr 2011 laufen ab Juni 2026 aus. Damit Windows-Geräte weiterhin Secure-Boot-bezogene Sicherheitsupdates erhalten können, müssen OEM-Hersteller BIOS-Updates mit den neuen 2023-Zertifikaten bereitstellen — und diese müssen auf den Geräten installiert sein.

Dieses Tool liest alle Windows-Geräte aus Intune via Microsoft Graph API aus, holt die BIOS-Version per Einzelabruf ab und vergleicht sie automatisch mit den veröffentlichten Mindestversionen der Hersteller. Das Ergebnis ist eine filterbare, exportierbare Compliance-Übersicht.

| Unterstützte Hersteller | Dell-Modelle | HP-Modelle | Lenovo-Modelle |
|---|---|---|---|
| 3 | ~320 | 296 | ausstehend |

> **Datenschutz:** Das Tool läuft vollständig im Browser. Es werden keine Daten an externe Server gesendet. Tenant-ID und Client-ID werden nur in der `sessionStorage` des Browsers gespeichert und beim Schliessen des Tabs verworfen.

---

## Voraussetzungen

- Moderner Browser (Chrome, Edge, Firefox, Safari)
- Zugriff auf einen Microsoft Entra ID (Azure AD) Tenant
- Berechtigung zum Erstellen einer App-Registrierung (oder eine bereits vorhandene)
- Die App-Registrierung benötigt die delegierte Berechtigung `DeviceManagementManagedDevices.Read.All`
- Ein Intune-Administrator-Account für die erste Anmeldung (Admin Consent)

---

## Azure AD App-Registrierung

### Neue App erstellen

1. Navigiere zu [portal.azure.com](https://portal.azure.com) → **Azure Active Directory** → **App-Registrierungen** → **Neue Registrierung**
2. Wähle einen aussagekräftigen Namen, z.B. `Intune BIOS Compliance Checker`. Den Kontotyp auf **Nur dieses Verzeichnis** lassen.
3. Plattform: **Single-Page Application (SPA)**. URI: die vollständige URL, unter der das Tool geöffnet wird. Das Tool zeigt die korrekte URI beim Öffnen automatisch an.
4. Nach der Erstellung die **Anwendungs-ID (Client ID)** und die **Verzeichnis-ID (Tenant ID)** von der Übersichtsseite kopieren.

### Berechtigungen konfigurieren

In der App-Registrierung unter **API-Berechtigungen**:

1. **Berechtigung hinzufügen** → **Microsoft Graph**
2. Typ: **Delegierte Berechtigungen**
3. `DeviceManagementManagedDevices.Read.All` suchen und aktivieren
4. **Administratorzustimmung erteilen** (erfordert Intune-Admin-Rolle)

> **Wichtig:** Die Berechtigung muss als *delegierte* Berechtigung eingetragen sein — nicht als Anwendungsberechtigung.

### Redirect URI

Die Redirect URI muss exakt mit dem Pfad übereinstimmen, unter dem das HTML-File im Browser geöffnet wird:

```
# Lokale Datei
file:///C:/Users/Benutzername/Downloads/intune-geraete-uebersicht.html

# Webserver
https://tools.meinefirma.ch/intune-bios/
```

---

## Erste Anmeldung

1. **Tenant ID** und **Client ID** in die Felder eingeben
2. Klick auf **Mit Microsoft anmelden** — es öffnet sich ein Popup-Fenster
3. Mit einem Intune-Leseberechtigten Account anmelden
4. Beim ersten Mal ggf. den Zustimmungsdialog bestätigen
5. Das Tool beginnt automatisch mit dem Datenabruf

Tenant ID und Client ID werden in der `sessionStorage` gespeichert und stehen beim nächsten Öffnen im selben Tab wieder vor.

---

## Datenabruf

### Schritt 1 — Geräteliste

Das Tool ruft alle Windows-Geräte aus Intune ab:

```
GET /beta/deviceManagement/managedDevices
    ?$select=deviceName,manufacturer,model,operatingSystem,osVersion,id
    &$filter=operatingSystem eq 'Windows'
    &$top=999
```

Bei mehr als 999 Geräten werden automatisch alle Seiten via `@odata.nextLink` abgerufen.

### Schritt 2 — BIOS-Daten pro Gerät

`hardwareInformation` ist im `/beta`-Endpunkt nur per Einzelabruf verfügbar:

```
GET /beta/deviceManagement/managedDevices/{id}
    ?$select=hardwareInformation
```

Dieser Abruf erfolgt für alle Geräte parallel in Batches von je 10 Anfragen, um Graph API Throttling zu vermeiden.

> **Ladezeit:** Bei 500 Geräten sind das ~50 Runden à 10 parallele Anfragen. Die Ladezeit beträgt je nach Tenant-Grösse und Netzwerk 1–3 Minuten.

---

## BIOS-Compliance-Prüfung

### Versionsvergleich

Die aktuelle BIOS-Version wird segmentweise numerisch mit der Mindestversion verglichen — nicht alphabetisch:

```js
compareVersions("01.10.00", "01.09.00")  →  1  (aktuell ≥ minimum ✓)
compareVersions("1.8.0",    "1.10.0")    → -1  (veraltet ✗)
compareVersions("1.10.0",   "1.10.0")   →  0  (exakt gleich ✓)
```

### Modellabgleich (Normalisierung + Fuzzy)

Intune liefert Modellnamen oft in ausführlicher Form:

```
HP EliteBook 860 16 Inch G10 Notebook PC   ← Intune
HP EliteBook 860 16 G10                    ← HP-Datenbank
```

Der Abgleich läuft in drei Stufen:

| Stufe | Methode | Beispiel |
|---|---|---|
| 1 — Normalisierung | Störwörter entfernen (`Notebook PC`, `Inch`, `Laptop PC`, …) | `HP EliteBook 860 16 Inch G10 Notebook PC` → `hp elitebook 860 16 g10` |
| 2 — Substring | Normalisierter DB-Schlüssel ist in Intune-Namen enthalten (oder umgekehrt) | `hp elitebook 840 g8` ⊂ `hp elitebook 840 g8 sonderconfig` |
| 3 — Fuzzy (Jaccard) | Wortmengen-Ähnlichkeit ≥ 60 % | `hp elitebook 840-g8` ↔ `hp elitebook 840 g8` |

Wenn ein Fuzzy-Match verwendet wird, zeigt ein Tooltip auf der Mindestversions-Zelle an, welcher DB-Eintrag abgeglichen wurde.

### BIOS-Versionsextraktion

HP gibt BIOS-Versionen in Intune mit der BIOS Family als Präfix zurück:

```
V70 ver. 01.10.00   ← Intune (HP)
01.10.00            ← HP-Datenbank
```

Das Tool extrahiert die numerische Version automatisch:

```js
// Regex: /ver\.?\s*(\d+[\d.]+)/i
"V70 ver. 01.10.00"  →  "01.10.00"
"T76 Ver. 01.22.00"  →  "01.22.00"
"01.10.00"           →  "01.10.00"   // unverändert
"1.32.1"             →  "1.32.1"     // unverändert
```

Der ursprüngliche Rohwert aus Intune ist als Tooltip auf der BIOS-Zelle sichtbar und wird im CSV-Export als separate Spalte ausgegeben.

---

## Filter & Sortierung

| Filter | Beschreibung |
|---|---|
| Freitextsuche | Sucht in Gerätename, Modell, BIOS-Version und Betriebssystem |
| Hersteller | Dropdown mit allen vorhandenen Hersteller-Werten |
| Modell | Dropdown mit allen vorhandenen Modell-Werten |
| Status | Aktuell / Update erforderlich / Nicht geprüft |

Alle Spalten sind durch Klick auf den Spaltentitel sortierbar. Versionsfelder werden numerisch-semantisch sortiert. Die Standardsortierung zeigt Geräte mit **Update erforderlich** zuerst.

---

## CSV-Export

Der Export über **CSV exportieren** umfasst immer die aktuell gefilterte Ansicht:

| Spalte | Inhalt |
|---|---|
| Gerätename | Intune-Gerätename |
| Hersteller | Herstellername aus Intune |
| Modell | Modellname aus Intune |
| BIOS aktuell | Extrahierte numerische BIOS-Version |
| BIOS aktuell (raw) | Rohwert wie von Intune geliefert |
| BIOS Minimum | Mindestversion laut Hersteller-Datenbank |
| DB-Modell (abgeglichen) | Tatsächlich verwendeter DB-Eintrag für den Match |
| Status | Aktuell / Update nötig / — |
| Betriebssystem | Betriebssystem aus Intune |
| OS-Version | OS-Versionsnummer |

Die Datei wird als UTF-8 mit BOM exportiert und öffnet sich in Excel korrekt mit Umlauten.

---

## BIOS-Datenbank

Die Mindestversionen sind direkt im JavaScript des Tools als Lookup-Objekt hinterlegt:

| Hersteller | Quelle | Stand |
|---|---|---|
| Dell | [KB000347876](https://www.dell.com/support/kbdoc/en-us/000347876) | Februar 2026 |
| HP | [HP Support](https://support.hp.com/us-en/document/ish_13070353-13070429-16) | 2025/2026 |
| Lenovo | [HT518129](https://support.lenovo.com/us/en/solutions/ht518129) | ausstehend |

> **Hinweis:** Die Hersteller aktualisieren ihre Listen laufend. Für kritische Compliance-Entscheidungen sollten die Quellartikel direkt konsultiert werden.

---

## Status-Codes

| Status | Bedeutung |
|---|---|
| **Aktuell** ✅ | Installierte BIOS-Version ≥ Mindestversion. Das Gerät erfüllt die Voraussetzung für das Secure Boot 2023 Zertifikats-Update. |
| **Update erforderlich** ❌ | Installierte BIOS-Version < Mindestversion. Ein BIOS-Update ist notwendig, bevor das Secure Boot Zertifikat über Windows Update eingespielt werden kann. |
| **—** (nicht geprüft) | Kein Eintrag in der Datenbank für diesen Hersteller/Modell, oder `hardwareInformation` enthält keine BIOS-Version. Manuelle Prüfung empfohlen. |

---

## Graph API Referenz

### Verwendete Endpunkte

| Endpunkt | Zweck |
|---|---|
| `GET /beta/deviceManagement/managedDevices` | Geräteliste (Windows-gefiltert, paginiert) |
| `GET /beta/deviceManagement/managedDevices/{id}` | BIOS-Daten eines einzelnen Geräts |
| `POST /oauth2/v2.0/token` | Token-Anfrage (via MSAL) |

### Warum Beta-Endpunkt?

Das Feld `hardwareInformation.systemManagementBIOSVersion` ist nur im `/beta`-Endpunkt verfügbar. Im stabilen `/v1.0`-Endpunkt existiert `hardwareInformation` nicht als selektierbares Feld.

### Warum Einzelabruf statt Liste?

`hardwareInformation` wird beim Listen-Endpunkt grundsätzlich leer zurückgegeben — auch wenn es explizit in `$select` aufgeführt wird. Nur der Einzelabruf eines Geräts via `/managedDevices/{id}` liefert das befüllte Objekt.

---

## Fehlerbehebung

| Problem | Ursache | Lösung |
|---|---|---|
| Popup öffnet sich nicht | Browser blockiert Popups | Popups für die Seite im Browser erlauben |
| `AADSTS50011`: Redirect URI stimmt nicht überein | URI in App-Registrierung weicht vom Dateipfad ab | URI aus dem Tool kopieren und in Azure exakt so eintragen (Plattform: SPA) |
| `Insufficient privileges` | Admin Consent fehlt oder falsche Berechtigungsart | Berechtigung als *delegiert* eintragen und Admin Consent erteilen |
| BIOS-Version überall leer | Intune hat `hardwareInformation` noch nicht synchronisiert | Gerät in Intune → Synchronisieren → nach 15 Min. erneut versuchen |
| Status zeigt überall `—` | Modellname aus Intune wird nicht erkannt | CSV exportieren, Spalte *DB-Modell (abgeglichen)* prüfen; ggf. Eintrag in `BIOS_DB` ergänzen |
| Sehr lange Ladezeit | Viele Geräte, jedes braucht einen eigenen API-Request | Normal. Bei 1000+ Geräten 3–5 Minuten einplanen. |

---

## Changelog

### v1.8 — April 2026
- Windows-Filter direkt im Graph API Request (`$filter=operatingSystem eq 'Windows'`) — iOS/Android werden nicht mehr abgerufen

### v1.7 — April 2026
- Vollständig fluides Layout ohne feste `max-width`
- `table-layout: auto` — Spalten passen sich dem Inhalt an
- Gerätename und Modell können umbrechen statt abgeschnitten zu werden
- Filter- und Stats-Grid mit `auto-fit` für alle Bildschirmbreiten

### v1.6 — April 2026
- `extractBiosVersion()`: HP-Format `V70 ver. 01.10.00` → `01.10.00`
- Rohwert-Tooltip auf BIOS-Zelle und zusätzliche CSV-Spalte *BIOS aktuell (raw)*

### v1.5 — April 2026
- Dreistufiger Modellabgleich: Normalisierung → Substring → Fuzzy (Jaccard ≥ 60 %)
- Tooltip auf Mindestversions-Zelle zeigt abgeglichenen DB-Eintrag
- CSV-Spalte *DB-Modell (abgeglichen)*

### v1.4 — April 2026
- HP-Datenbank integriert (296 Modelle: Business Notebooks, Desktops, ZBooks, Thin Clients, POS)

### v1.3 — April 2026
- Dell-Datenbank integriert (~320 Modelle: Latitude, OptiPlex, Precision, XPS, Vostro, Alienware, …)
- Compliance-Spalten: BIOS aktuell, BIOS Minimum, Status
- Versionsvergleich semantisch-numerisch
- 4 Summary-Kacheln, Status-Filter
- CSV-Export mit Compliance-Daten

### v1.2 — April 2026
- Fix: `hardwareInformation` per Einzelabruf statt Listenendpunkt
- Zweistufiger Ladefortschritt (Geräteliste + BIOS-Daten)

### v1.1 — April 2026
- Fix: `/beta`-Endpunkt statt `/v1.0` für `hardwareInformation`

### v1.0 — April 2026
- Initiale Version: MSAL-Login, Graph-Abruf, Filter, Sortierung, CSV-Export, Dark Mode, Abmelden
