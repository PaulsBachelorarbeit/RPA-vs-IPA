# RPA_InvoiceProcessing & IPA_InvoiceProcessing — README

UiPath-Workflows zur experimentellen Verarbeitung eines PDF-Korpus im Rahmen einer Bachelorarbeit.  

- **RPA-Workflow**: regelbasierte Baseline  
- **IPA-Workflow**: Document Understanding (OCR + ML-Extraktion + Validation Station)  
- Beide Workflows erfassen identische Metriken.  

---

## 1. Ziel und Geltungsbereich

**Ziel**  
- Sequenzielle Verarbeitung eines PDF-Korpus  
- Extraktion: `invoiceNumber`, `amountGross`, `invoiceDate`  
- Messung: `processingTime`, `reworkTime`  
- Bestimmung: Straight-Through-Processing (STP)  
- Protokollierung: Log-Tabelle  

**Geltungsbereich**  
- **RPA (Baseline)**: nur textbasierte PDFs ohne OCR. Scans ohne eingebetteten Text → `PDF-Read-Error`, Logging als `Erfolgreich = False`.  
- **IPA (DU)**: textbasierte **und** gescannte PDFs, OCR (`deu+eng`), Confidence-Gating, Validierungsstation.  

**Experimentelle Rolle**  
Vergleich beider Verfahren über Input-Stufen mit **0 %, 15 %, 30 %, 45 %, 60 % unstrukturierten Eingaben**.  

---

## 2. Systemvoraussetzungen

- **UiPath Studio** (Community oder Enterprise), getestet in Versionen 2024/2025  
- **Pakete**  
  - RPA: `UiPath.System.Activities`, `UiPath.UiAutomation.Activities`, `UiPath.PDF.Activities`, `UiPath.GSuite.Activities`  
  - IPA zusätzlich: `UiPath.IntelligentOCR.Activities`, `UiPath.DocumentProcessing.Contracts`, `UiPath.DocumentUnderstanding.ML`, `UiPathDocumentOCR`  
- **Framework-Assemblies**: diverse `System.*` (gemäß XAML ReferencesForImplementation)  
- **Google Workspace**: Zugriff auf Ziel-Spreadsheet (bei Sheets-Logging)  
- **DU-Backend (IPA)**: UiPath AI Center-Skill oder Cloud DU Endpoint  

---

## 3. Konfiguration & Argumente

> Empfehlung: feste Assigns vermeiden und stattdessen **In-Argumente** nutzen.

| Argument         | Typ     | Beschreibung |
|------------------|---------|--------------|
| `in_RunLabel`    | String  | Lauf-Label (z. B. `Stufe_60_Unstructured`) |
| `in_LogPath`     | String  | Ziel für das Log (Pfad oder Blattname) |
| `in_InputFolder` | String  | Eingangs-Ordner mit PDFs |

---

## 4. Datenmodell (Log-Tabelle)

Identisches Schema für RPA und IPA. Unterscheidung über Spalte `Verfahren`.

| Spalte              | Typ        | Bedeutung |
|---------------------|------------|-----------|
| Dateiname           | string     | Dateiname der Rechnung |
| Startzeit           | dateTime   | Beginn der Verarbeitung |
| Endzeit             | dateTime   | Ende der Verarbeitung |
| Bearbeitungszeit    | double     | (Endzeit − Startzeit) in Sekunden |
| Nachbearbeitungszeit| double     | Validierungszeit in Sekunden |
| Rechnungsnummer     | string     | extrahierter bzw. validierter Wert |
| Betrag              | string     | formatiert (de-DE) |
| Datum               | string/DT  | Rechnungsdatum |
| Erfolgreich         | boolean    | `False` bei technischem Fehler |
| STP                 | boolean    | `True` wenn keine Nachbearbeitung |
| Fehlerhaft          | boolean    | Abweichung extrahiert vs. validiert |
| Exception           | string     | Fehlermeldung |
| Verfahren           | string     | `RPA` oder `IPA` |

---

## 5. Workflow-Übersicht

### 5.1 RPA_InvoiceProcessing (Baseline)

1. Initialisierung: Input/Log setzen, Dateiliste ermitteln, Log-Tabelle vorbereiten  
2. Zeitstart pro Datei (`startTime`, `reworkSeconds = 0`, `isSTP = True`)  
3. PDF lesen → `Read PDF Text`  
   - Fehler → `isSTP = False`, `isSuccessful = False`, Exception loggen  
4. Regex-Extraktion (Rechnungsnummer, Bruttosumme, Rechnungsdatum)  
5. Regelbasierte Validierungen → ggf. manuelle Eingabe (Input Dialog)  
6. 5 % Zufallsstichprobe zur zusätzlichen Plausibilisierung  
7. Zeitende & STP bestimmen  
8. Logging in Excel/Sheets  

**STP-Definition**  
```vb
STP = Not needsManualValidation AndAlso reworkSeconds = 0
```

---

### 5.2 IPA_InvoiceProcessing (Document Understanding)

1. **Digitize Document** (OCR `deu+eng`)  
2. **Load Taxonomy**: Felder `invoiceNumber`, `amount`, `invoiceDate`  
3. **Data Extraction Scope**: Machine Learning Extractor (`InvoiceML`)  
4. **Confidence/Gating**: Schwellenwertprüfung, Zufallsstichprobe  
5. **Validation Station** (bei Bedarf) → Rework-Zeit messen  
6. **STP-Definition** wie oben  
7. **Logging** analog RPA, zusätzlich `Verfahren = IPA`  

---

## 6. Metriken

- **Bearbeitungszeit [s]** = Endzeit − Startzeit  
- **Nachbearbeitungszeit [s]** = Dauer Validierungen  
- **STP [bool]** = keine Nachbearbeitung  
- **Erfolgreich [bool]** = keine technischen Fehler  
- **Fehlerhaft [bool]** = Abweichung validiert vs. extrahiert  

---

## 7. Fehlerbehandlung

- **RPA**: `PDF-Read-Error` → `isSuccessful = False`, `STP = False`  
- **IPA**: OCR-/Extractor-/Endpoint-Fehler → analoge Behandlung  

---

## 8. Reproduzierbarkeit & Parametrisierung

- Argumente aktivieren (`in_InputFolder`, `in_LogPath`)  
- Stichprobe deterministisch steuern (Seed oder Regel, z. B. jeder 20. Beleg)  
- Schwellenwerte als Argumente setzen (`in_ConfThresh_*`)  
- Schreibstrategie wählen: Batch-Flush oder Append  
- Versionskennung über `in_RunLabel` im Log speichern  
- Secrets: DU-Endpoint/API-Key via Orchestrator Assets  

---

## 9. Einrichtung (Step by Step)

1. Projekt in UiPath Studio öffnen  
2. Pakete prüfen und auflösen (`Manage Packages`)  
3. Google-Verbindung konfigurieren (bei Sheets-Logging)  
4. DU-Skill/Endpoint einrichten (AI Center oder Cloud DU)  
5. Input-Ordner setzen  
6. Testlauf mit Stichprobe durchführen  
7. Hauptläufe für alle Stufen (0–60 %)  

---

## 10. Beispiel-Logs

### RPA
```json
{
  "Dateiname": "INV_000123.pdf",
  "Bearbeitungszeit": 0.89,
  "Nachbearbeitungszeit": 0,
  "Rechnungsnummer": "RN-2025-000123",
  "Betrag": "1.234,56",
  "Datum": "2025-08-12",
  "Erfolgreich": true,
  "STP": true,
  "Fehlerhaft": false,
  "Verfahren": "RPA"
}
```

### IPA
```json
{
  "Dateiname": "INV_004201.pdf",
  "Bearbeitungszeit": 3.78,
  "Nachbearbeitungszeit": 12.41,
  "Rechnungsnummer": "RN-2025-004201",
  "Betrag": "982,40",
  "Datum": "2025-08-28",
  "Erfolgreich": true,
  "STP": false,
  "Fehlerhaft": true,
  "Verfahren": "IPA"
}
```

---

## 11. Lizenz & Kontakt

- Interner Forschungs-Workflow im Rahmen der Bachelorarbeit von **Paul Schmidt (TH Brandenburg)**  
- Drittanbieter-Pakete gemäß deren Lizenzbedingungen  
- Rückfragen: bitte an den Autor der Arbeit  

---
