RPA_InvoiceProcessing & IPA_InvoiceProcessing — README

Kontext. Dieses Repository umfasst zwei UiPath‑Workflows für das Experiment der Bachelorarbeit: (1) RPA_InvoiceProcessing.xaml als regelbasierte Baseline und (2) IPA_InvoiceProcessing1.xaml (Document Understanding) mit OCR, ML‑Extraktion, Confidence‑Gating und Validierungsstation. Beide Workflows verarbeiten denselben PDF‑Korpus und protokollieren identische Metriken für den Vergleich.

⸻

1. Ziel und Geltungsbereich
	•	Ziel: Sequenzielle Verarbeitung eines PDF‑Korpus, Extraktion von Rechnungsnummer, Bruttosumme, Rechnungsdatum, Messung von Bearbeitungszeit und Nachbearbeitungszeit, Bestimmung von Straight‑Through‑Processing (STP) sowie Protokollierung in einer Log‑Tabelle.
	•	Geltungsbereich: Nur textbasierte PDFs (kein OCR). Scans ohne eingebetteten Text führen zum PDF‑Read‑Error und werden als `Erfolgreich = False` geloggt.
	•	Experimentelle Rolle: RPA‑Baseline ohne KI‑Komponenten. Dient dem Vergleich über die Input‑Stufen (0 %, 15 %, 30 %, 45 %, 60 % unstrukturierte Eingaben).

⸻

2. Systemvoraussetzungen
	•	UiPath Studio (Community/Enterprise), .NET‑basierte Workflows (getestet in 2024/2025‑Versionen)
	•	Pakete (Auszug, laut XAML‑References):
	•	UiPath.System.Activities, UiPath.UiAutomation.Activities, UiPath.PDF.Activities
	•	UiPath.GSuite.Activities (Sheets/Drive via Connection Service)
	•	System.* Framework‑Assemblies (siehe XAML‑Block ReferencesForImplementation)
	•	Google Workspace Zugriff mit berechtigtem Konto auf das Ziel‑Spreadsheet

⸻

3. Eingaben, Variablen und Datenmodell

3.1 In‑Argumente (derzeit nicht aktiv genutzt)

In der XAML sind In‑Argumente definiert, werden jedoch in der Startsequenz überschrieben. Für reproduzierbare Läufe wird empfohlen, die Zuweisungen zu entfernen und die In‑Argumente zu nutzen (siehe Abschnitt 9).

	•	`in_RunLabel : String` — Freies Lauf‑Label (z. B. `“Stufe_60_Unstructured”`).
	•	`in_LogPath : String` — Pfad/Bezeichner für das Logging (z. B. lokale Datei oder Blattname); im aktuellen Stand ungenutzt.
	•	`in_InputFolder : String` — Ordner mit Eingangs‑PDFs; wird aktuell durch eine Assign‑Aktivität überschrieben.

3.2 Interne Variablen (Auswahl)
	•	`inputFolder : String` — hart verdrahtet auf
C:\Users\Paul\Documents\UiPath\RPA_IPA_Experiment\Data\Input\60%_Unstructured
	•	`files : String[]` — Liste aller *.pdf im `inputFolder`
	•	Zeitmessung: `startTime, endTime : DateTime`, `durationSeconds : Double`
	•	Nacharbeit: `valStart, valEnd : DateTime`, `reworkSeconds : Double`
	•	Steuerung: `needsManualValidation : Bool`, `isSTP : Bool` (s. 5.3), `isSuccessful : Bool` (Default `True`)
	•	Extraktion: `pdfText : String`, `invoiceNumber, amount, invoiceDate : String`
	•	Validierungs‑Inputs: `tmpInvoiceNumber, tmpAmount, tmpInvoiceDate : String`
	•	Fehlerflags: `exceptionMessage : String`, `hasBussinesError : Bool` (Tippfehler im Variablennamen ist beibehalten)

3.3 Log‑Tabelle (`dt_Log`)

Schema lt. Build Data Table:

Spalte	Typ	Bedeutung
Dateiname	string	Dateiname der Rechnung
Startzeit	dateTime	Beginn der Verarbeitung
Endzeit	dateTime	Ende der Verarbeitung
Bearbeitungszeit	double	`(Endzeit − Startzeit)` in Sekunden
Nachbearbeitungszeit	double	Summe manuell gemessener Validierungszeiten in Sekunden
Rechnungsnummer	string	Extrahierter Wert
Betrag	string	Extrahierter Wert im DE‑Format (z. B. 1.234,56)
Datum	string	Extrahiertes Rechnungsdatum (YYYY‑MM‑DD oder DD.MM.YYYY)
Erfolgreich	boolean	`False` nur bei technischem Fehler (z. B. PDF‑Read‑Error)
STP	boolean	True, wenn keine Nachbearbeitung stattfand (Definition s. 5.3)
Fehlerhaft	string	Siehe 5.4 (aktuelles Verhalten und Empfehlung)
Exception	string	Fehlertext bei Exception
Verfahren	string	Statischer Wert `“RPA”`


⸻

4. Prozessüberblick
	1.	Initialisierung
	•	Setzen von `inputFolder` und `logPath` per Assign (hart verdrahtet)
	•	Dateien ermitteln: `files = Directory.GetFiles(inputFolder, “*.pdf”)`
	•	`dt_Log` erzeugen, ggf. Zeilen leeren
	2.	Schleife über PDFs (`For Each File In Folder`)
	1.	Zeitstart: `startTime = Now`; `reworkSeconds = 0`; `needsManualValidation = False`; `isSTP = True`
	2.	PDF lesen: `Read PDF Text` → `pdfText`
	•	Bei Fehler: `isSTP=False`, `isSuccessful=False`, Felder leeren, `exceptionMessage = “PDF-Read-Error: …”`
	3.	Regex‑Extraktion (siehe 5.1)
	4.	Regelbasierte Prüfungen (siehe 5.2) → ggf. manuelle Nachbearbeitung
	5.	5‑% Stichprobe: Zufallsvalidierung mit Nutzereingaben (siehe 5.2 b)
	6.	STP‑Bestimmung und Zeitende: `endTime = Now`; `durationSeconds = (endTime - startTime).TotalSeconds`
	7.	Logging: Zeile in `dt_Log`, anschließend Write Range in Google‑Sheets (Überschreiben des Bereichs)

⸻

5. Extraktion, Prüfregeln und Metriken

5.1 Reguläre Ausdrücke (Extraktion)
	•	Rechnungsnummer:

Rechnungsnummer\s*\(BT-1\):\s*(\S+)


	•	Bruttosumme (Betrag):

Bruttosumme\s*\(BT-115\):\s*([\d,.]+)\s*€


	•	Rechnungsdatum:

Rechnungsdatum\s*\(BT-2\):\s*(\d{4}-\d{2}-\d{2})



Hinweis: Die Formate sind eng an den synthetischen Vorlagen ausgerichtet. Abweichende Formate werden unter 5.2 als Validierungsgrund behandelt.

5.2 Validierungen und manuelle Schritte

a) Validierung nach Extraktion

`needsManualValidation = True`, wenn eine der Bedingungen zutrifft:
	•	Mindestens ein Feld leer: `invoiceNumber`, `amount` oder `invoiceDate` ist leer
	•	Mehrdeutigkeit bei Rechnungsnummer:

Rechnungsnummer\s*\(BT-1\):\s*(\S+)|Rechnung\s*Nr.\s*(\S+)

→ Wenn mehr als ein Match gefunden wird

	•	Betragsformat nicht DE‑konform:

^\d{1,3}(\.\d{3})*(,\d{2})$|^\d+([.,]\d{2})$


	•	Datumsformat nicht in `YYYY-MM-DD` oder `DD.MM.YYYY`:

^\d{4}-\d{2}-\d{2}$|^\d{2}\.\d{2}\.\d{4}$



Wenn `needsManualValidation = True`:
	•	PDF wird geöffnet
	•	Für fehlende Felder werden Input Dialogs angezeigt
	•	Zeitfenster wird mit `valStart/valEnd` gemessen → `reworkSeconds`
	•	`isSTP = False`

b) 5‑% Zufalls‑Stichprobe
	•	`rand = (new Random(Guid.NewGuid().GetHashCode())).NextDouble()`
	•	Falls `rand < 0.05`: manuelle Plausibilitätsprüfung über drei Input Dialogs
	•	Zeitmessung analog a) → `reworkSeconds += …`; `isSTP = False`

5.3 Definition von STP
	•	`STP = True` genau dann, wenn keine manuelle Nachbearbeitung erfolgte:
`Not needsManualValidation AndAlso reworkSeconds = 0`

5.4 Spalte „Fehlerhaft“ — aktuelles Verhalten und Empfehlung
	•	Aktuelles Verhalten (Code‑Stand):
	•	`hasBussinesError` (Bool) wird nur gesetzt, wenn mindestens eine der temporären Eingaben `tmpInvoiceNumber/tmpAmount/tmpInvoiceDate` ungleich `“0”` ist:
If (tmpInvoiceNumber <> "0" OrElse tmpAmount <> "0" OrElse tmpInvoiceDate <> "0") Then hasBussinesError = needsManualValidation
	•	In der „Fehlerhaft“‑Spalte (vom Typ string) wird dieser Bool‑Wert geschrieben.
	•	Konsequenz: Fälle mit fehlenden Feldern (Zweig 5.2 a) setzen `needsManualValidation = True`, ohne die `tmp*`‑Variablen zu nutzen. Dadurch bleibt `hasBussinesError = False`, obwohl eine Nachbearbeitung erfolgte.
	•	Empfehlung (fachlich konsistent):
	•	Setze immer: `hasBussinesError = needsManualValidation`
	•	Typ der „Fehlerhaft“‑Spalte auf boolean ändern
	•	Optional: „Fehlerhaft“ verwenden, um inhaltliche Abweichungen zu kennzeichnen und technische Fehler weiterhin separat in `Erfolgreich` bzw. `Exception` abzubilden

⸻

6. Fehlerbehandlung
	•	PDF‑Read‑Error (z. B. gescanntes PDF ohne Text):
`isSuccessful = False`, `isSTP = False`, Felder leer, Meldung in `Exception`
	•	Sonstige Ausnahmen innerhalb der Extraktion/Validierung sind im aktuellen Stand nicht gesondert abgefangen.

⸻

7. Ausgabeziel (Google Sheets)
	•	Aktivität: Write Range (Connections) aus `UiPath.GSuite.Activities`
	•	Schreibmodus: `Overwrite` des Bereichs `Tabellenblatt1`
	•	Verbindung: über UiPath Connection Service (im XAML durch `ConnectionId` und Browser‑Auswahl referenziert)
	•	Hinweis: Bei großen Korpora führt das wiederholte Überschreiben nach jeder Datei zu unnötigen Schreibzugriffen. Alternative: Batch‑Flush am Ende oder Append‑Strategie.

⸻

8. Reproduzierbarkeit und Parametrisierung

Empfohlene Anpassungen für Experiment‑Runs:
	1.	In‑Argumente nutzen
	•	Entferne die Assigns für `inputFolder` und `logPath`
	•	Binde `in_InputFolder` und `in_LogPath` in den Aktivitäten ein
	2.	Deterministische Stichprobe
	•	Ersetze den Zufall mit festem Seed pro Lauf oder verwende eine deterministische Auswahlregel (z. B. jeder 20. Beleg)
	3.	Log‑Schreibstrategie
	•	Schreibe einmal am Ende des Durchlaufs oder nutze Append
	4.	„Fehlerhaft“‑Semantik korrigieren
	•	Siehe 5.4
	5.	Versionskennung
	•	Schreibe `in_RunLabel` zusätzlich als Spalte in `dt_Log`

⸻

9. Einrichtung Schritt für Schritt
	1.	Projekt öffnen in UiPath Studio
	2.	Pakete prüfen (Manage Packages) und auflösen
	3.	Google‑Verbindung herstellen
	•	In Studio: Data Service / Connections → G‑Sheets Verbindung anlegen
	•	Zugriff auf das Zieldokument gewähren
	•	In der Aktivität Write Range (Connections) das Ziel‑Spreadsheet auswählen
	4.	Input‑Ordner festlegen
	•	Entweder Assign anpassen oder (empfohlen) `in_InputFolder` aktivieren
	5.	Testlauf mit kleiner Stichprobe
	6.	Hauptlauf
	•	Für die Stufen 0 %, 15 %, 30 %, 45 %, 60 % jeweils separat starten


⸻



10. Beispiel: Auszug einer Log‑Zeile

Dateiname:          INV_000123.pdf
Startzeit:          2025-09-12T10:44:12
Endzeit:            2025-09-12T10:44:13
Bearbeitungszeit:   0.89
Nachbearbeitungszeit: 0
Rechnungsnummer:    RN-2025-000123
Betrag:             1.234,56
Datum:              2025-08-12
Erfolgreich:        True
STP:                True
Fehlerhaft:         False
Exception:          
Verfahren:          RPA

⸻

11. Lizenz & Autorenschaft
	•	Interner Forschungs‑Workflow im Rahmen der Bachelorarbeit von Paul Schmidt (TH Brandenburg)
	•	Drittanbieter‑Pakete gemäß deren Lizenzen

⸻


IPA_InvoiceProcessing — Document Understanding

Kurzüberblick. Der IPA‑Workflow implementiert UiPath Document Understanding (DU): Digitize Document (UiPath Document OCR, deu+eng), Load Taxonomy, Data Extraction Scope mit MachineLearningExtractor (SelectedMLSkill = "InvoiceMl") und Present Validation Station. Auf Basis von Konfidenzwerten werden Fälle automatisch freigegeben oder zur manuellen Validierung vorgelegt. Ergebnisse werden geloggt und optional formatiert.

A. Ziel und Geltungsbereich (IPA)
	•	Ziel: KI‑gestützte Extraktion von Rechnungsnummer, Bruttosumme, Rechnungsdatum aus gescannten und textbasierten PDFs, inkl. Zeitmessung, STP‑Bestimmung und Fehlerflag.
	•	Geltungsbereich: Auch gescannte PDFs (OCR). Sprache OCR: deu+eng.
	•	Experimentelle Rolle: IPA‑Variante zur Gegenüberstellung mit der RPA‑Baseline über die Stufen 0 % / 15 % / 30 % / 45 % / 60 % unstrukturierte Eingaben.

B. Systemvoraussetzungen (zusätzlich zur RPA‑Variante)
	•	Pakete/Namespaces (Auszug): UiPath.IntelligentOCR.Activities (DU), UiPath.DocumentProcessing.Contracts, UiPath.DocumentUnderstanding.ML (MachineLearningExtractor), UiPathDocumentOCR.
	•	ML‑Skill: SelectedMLSkill = "InvoiceMl" (Konfiguration über UiPath AI Center bzw. DU Endpoint).
	•	API‑Key/Endpoint: Im XAML ist ein API‑Key referenziert. Empfehlung: Schlüssel/Endpoint in Orchestrator Assets oder Connection Service ablegen und zur Laufzeit binden (keine Hardcodierung im Workflow/Repo).

C. Eingaben, Variablen und Datenmodell (IPA)
	•	Pfadsteuerung: inputFolder = Path.Combine(CurrentDirectory, "Data", "Input", "60%_Unstructured") (hart vergeben; für reproduzierbare Läufe parametrisierbar).
	•	Log‑Datei: logPath = "Logs\\ipa_log.xlsx" (Excel‑Ausgabe; siehe Abschnitt G).
	•	Kernvariablen:
	•	Zeiten: startTime, endTime, durationSeconds; Validierung: valStart, valEnd, reworkSeconds.
	•	Extraktion (final): invoiceNumber : String, amount : Double, invoiceDate : DateTime.
	•	Validiert (tmp): tmpInvoiceNumber : String, tmpAmount : Double, tmpInvoiceDate : DateTime.
	•	Steuerung: confInvoiceNumber : Double, confTotal : Double (Betrag), needsValidation : Bool, forceValidation : Bool (5‑% Probe), isSTP : Bool, hasBusinessError : Bool, isSuccessful : Bool, exceptionMessage : String.
	•	Hilfen: rawAmount : String, amountFormatted : String, ocrText : String, documentObject, taxonomy.
	•	Taxonomie: Finance.IncomingInvoice.Invoice mit invoiceNumber, amount, invoiceDate.
	•	Log‑Schema (dt_Log):
	•	Dateiname (string), Startzeit (dateTime), Endzeit (dateTime), Bearbeitungszeit (double), Nachbearbeitungszeit (double),
	•	Rechnungsnummer (string), Betrag (string, formatiert), Datum (dateTime),
	•	Erfolgreich (bool), Fehlerhaft (bool), STP (bool), Exception (string), Verfahren (string="IPA").

D. Prozessüberblick (IPA)
	1.	Initialisierung: Pfade setzen, dt_Log aufbauen, Dateiliste (*.pdf) ermitteln.
	2.	Digitize Document (UiPath Document OCR, deu+eng): OCR für gescannte PDFs (ApplyOCROnPdf = Auto, ForceApplyOCR = False).
	3.	Load Taxonomy → Data Extraction Scope mit MachineLearningExtractor:
	•	SelectedMLSkill = "InvoiceMl", Timeout = 100000, UseServerSideOCR = False.
	•	Extraktionsresultate (extractionResults) stehen als ResultsDocument mit Feldern/Confidence‑Werten bereit.
	4.	Konfidenzen & Normalisierung:
	•	confInvoiceNumber = Confidence(Invoice.invoiceNumber); confTotal = Confidence(Invoice.amount).
	•	Betrag wird aus rawAmount nach de‑DE bzw. InvariantCulture geparst → amount : Double; zusätzlich amountFormatted = amount.ToString("N2", de-DE) für das Log.
	5.	Gating & Stichprobe:
	•	forceValidation = (New Random()).NextDouble() <= 0.05 (5‑% Zufallsprobe).
	•	needsValidation = (confInvoiceNumber < 0.85) OR (confTotal < 0.85 AND String.IsNullOrWhiteSpace(amountProbe)) OR forceValidation.
	6.	Validierung (falls nötig): Present Validation Station mit Taxonomie.
	•	Zeitfenster messen (valStart/valEnd) → reworkSeconds += ....
	•	Validierte Werte in tmpInvoiceNumber/tmpAmount/tmpInvoiceDate übernehmen.
	•	Business‑Fehlerermittlung:

hasBusinessError = Not (
   String.Equals(tmpInvoiceNumber, invoiceNumber) _
   AndAlso (Math.Abs(Math.Round(tmpAmount,2) - Math.Round(amount,2)) <= 0.01) _
   AndAlso (tmpInvoiceDate.Date = invoiceDate.Date)
)


	7.	STP‑Definition (IPA): isSTP = Not needsValidation AndAlso reworkSeconds = 0.
	8.	Logging: AddDataRow → WriteRange (Excel) mit Schema oben, Verfahren = "IPA".
	9.	Fehlerbehandlung: In Exceptions → isSuccessful = False, exceptionMessage setzen; Werte ansonsten leer/Default.

E. Metriken (IPA)
	•	Bearbeitungszeit [s] = (Endzeit − Startzeit).TotalSeconds
	•	Nachbearbeitungszeit [s] = Summe der Validierungsphasen
	•	STP [bool] = siehe D.7
	•	Erfolgreich [bool] = False bei technischem Fehler
	•	Fehlerhaft [bool] = True, wenn validierte Werte von initialen Extraktionen abweichen (Toleranz: ±0,01 beim Betrag; Datum auf Tagesebene)


F. Einrichtung Schritt für Schritt (IPA)
	1.	Pakete auflösen (IntelligentOCR, DU, UiPathDocumentOCR).
	2.	AI‑Skill/Endpoint bereitstellen; ApiKey/Endpoint als Asset ablegen und im Workflow referenzieren.
	3.	Taxonomie prüfen (Invoice: invoiceNumber, amount, invoiceDate).
	4.	inputFolder anpassen oder als In‑Argument/Config auslagern.
	5.	Testlauf mit wenigen PDFs; danach Stufen 0/15/30/45/60 % separat laufen lassen.
	6.	Log‑Ziel Logs/ipa_log.xlsx verifizieren (Schreibrechte, Blatt Log).

RPA_InvoiceProcessing & IPA_InvoiceProcessing — README

Kontext. Dieses Repository umfasst zwei UiPath‑Workflows für das Experiment der Bachelorarbeit: (1) RPA_InvoiceProcessing.xaml als regelbasierte Baseline und (2) IPA_InvoiceProcessing1.xaml (Document Understanding) mit OCR, ML‑Extraktion, Confidence‑Gating und Validierungsstation. Beide Workflows verarbeiten denselben PDF‑Korpus und protokollieren identische Metriken für den Vergleich.

⸻

1. Ziel und Geltungsbereich
	•	Ziel: Sequenzielle Verarbeitung eines PDF‑Korpus, Extraktion von Rechnungsnummer, Bruttosumme, Rechnungsdatum, Messung von Bearbeitungszeit und Nachbearbeitungszeit, Bestimmung von Straight‑Through‑Processing (STP) sowie Protokollierung in einer Log‑Tabelle.
	•	Geltungsbereich: Nur textbasierte PDFs (kein OCR). Scans ohne eingebetteten Text führen zum PDF‑Read‑Error und werden als `Erfolgreich = False` geloggt.
	•	Experimentelle Rolle: RPA‑Baseline ohne KI‑Komponenten. Dient dem Vergleich über die Input‑Stufen (0 %, 15 %, 30 %, 45 %, 60 % unstrukturierte Eingaben).

⸻

2. Systemvoraussetzungen
	•	UiPath Studio (Community/Enterprise), .NET‑basierte Workflows (getestet in 2024/2025‑Versionen)
	•	Pakete (Auszug, laut XAML‑References):
	•	UiPath.System.Activities, UiPath.UiAutomation.Activities, UiPath.PDF.Activities
	•	UiPath.GSuite.Activities (Sheets/Drive via Connection Service)
	•	System.* Framework‑Assemblies (siehe XAML‑Block ReferencesForImplementation)
	•	Google Workspace Zugriff mit berechtigtem Konto auf das Ziel‑Spreadsheet

⸻

3. Eingaben, Variablen und Datenmodell

3.1 In‑Argumente (derzeit nicht aktiv genutzt)

In der XAML sind In‑Argumente definiert, werden jedoch in der Startsequenz überschrieben. Für reproduzierbare Läufe wird empfohlen, die Zuweisungen zu entfernen und die In‑Argumente zu nutzen (siehe Abschnitt 9).

	•	`in_RunLabel : String` — Freies Lauf‑Label (z. B. `“Stufe_60_Unstructured”`).
	•	`in_LogPath : String` — Pfad/Bezeichner für das Logging (z. B. lokale Datei oder Blattname); im aktuellen Stand ungenutzt.
	•	`in_InputFolder : String` — Ordner mit Eingangs‑PDFs; wird aktuell durch eine Assign‑Aktivität überschrieben.

3.2 Interne Variablen (Auswahl)
	•	`inputFolder : String` — hart verdrahtet auf
C:\Users\Paul\Documents\UiPath\RPA_IPA_Experiment\Data\Input\60%_Unstructured
	•	`files : String[]` — Liste aller *.pdf im `inputFolder`
	•	Zeitmessung: `startTime, endTime : DateTime`, `durationSeconds : Double`
	•	Nacharbeit: `valStart, valEnd : DateTime`, `reworkSeconds : Double`
	•	Steuerung: `needsManualValidation : Bool`, `isSTP : Bool` (s. 5.3), `isSuccessful : Bool` (Default `True`)
	•	Extraktion: `pdfText : String`, `invoiceNumber, amount, invoiceDate : String`
	•	Validierungs‑Inputs: `tmpInvoiceNumber, tmpAmount, tmpInvoiceDate : String`
	•	Fehlerflags: `exceptionMessage : String`, `hasBussinesError : Bool` (Tippfehler im Variablennamen ist beibehalten)

3.3 Log‑Tabelle (`dt_Log`)

Schema lt. Build Data Table:

Spalte	Typ	Bedeutung
Dateiname	string	Dateiname der Rechnung
Startzeit	dateTime	Beginn der Verarbeitung
Endzeit	dateTime	Ende der Verarbeitung
Bearbeitungszeit	double	`(Endzeit − Startzeit)` in Sekunden
Nachbearbeitungszeit	double	Summe manuell gemessener Validierungszeiten in Sekunden
Rechnungsnummer	string	Extrahierter Wert
Betrag	string	Extrahierter Wert im DE‑Format (z. B. 1.234,56)
Datum	string	Extrahiertes Rechnungsdatum (YYYY‑MM‑DD oder DD.MM.YYYY)
Erfolgreich	boolean	`False` nur bei technischem Fehler (z. B. PDF‑Read‑Error)
STP	boolean	True, wenn keine Nachbearbeitung stattfand (Definition s. 5.3)
Fehlerhaft	string	Siehe 5.4 (aktuelles Verhalten und Empfehlung)
Exception	string	Fehlertext bei Exception
Verfahren	string	Statischer Wert `“RPA”`


⸻

4. Prozessüberblick
	1.	Initialisierung
	•	Setzen von `inputFolder` und `logPath` per Assign (hart verdrahtet)
	•	Dateien ermitteln: `files = Directory.GetFiles(inputFolder, “*.pdf”)`
	•	`dt_Log` erzeugen, ggf. Zeilen leeren
	2.	Schleife über PDFs (`For Each File In Folder`)
	1.	Zeitstart: `startTime = Now`; `reworkSeconds = 0`; `needsManualValidation = False`; `isSTP = True`
	2.	PDF lesen: `Read PDF Text` → `pdfText`
	•	Bei Fehler: `isSTP=False`, `isSuccessful=False`, Felder leeren, `exceptionMessage = “PDF-Read-Error: …”`
	3.	Regex‑Extraktion (siehe 5.1)
	4.	Regelbasierte Prüfungen (siehe 5.2) → ggf. manuelle Nachbearbeitung
	5.	5‑% Stichprobe: Zufallsvalidierung mit Nutzereingaben (siehe 5.2 b)
	6.	STP‑Bestimmung und Zeitende: `endTime = Now`; `durationSeconds = (endTime - startTime).TotalSeconds`
	7.	Logging: Zeile in `dt_Log`, anschließend Write Range in Google‑Sheets (Überschreiben des Bereichs)

⸻

5. Extraktion, Prüfregeln und Metriken

5.1 Reguläre Ausdrücke (Extraktion)
	•	Rechnungsnummer:

Rechnungsnummer\s*\(BT-1\):\s*(\S+)


	•	Bruttosumme (Betrag):

Bruttosumme\s*\(BT-115\):\s*([\d,.]+)\s*€


	•	Rechnungsdatum:

Rechnungsdatum\s*\(BT-2\):\s*(\d{4}-\d{2}-\d{2})



Hinweis: Die Formate sind eng an den synthetischen Vorlagen ausgerichtet. Abweichende Formate werden unter 5.2 als Validierungsgrund behandelt.

5.2 Validierungen und manuelle Schritte

a) Validierung nach Extraktion

`needsManualValidation = True`, wenn eine der Bedingungen zutrifft:
	•	Mindestens ein Feld leer: `invoiceNumber`, `amount` oder `invoiceDate` ist leer
	•	Mehrdeutigkeit bei Rechnungsnummer:

Rechnungsnummer\s*\(BT-1\):\s*(\S+)|Rechnung\s*Nr.\s*(\S+)

→ Wenn mehr als ein Match gefunden wird

	•	Betragsformat nicht DE‑konform:

^\d{1,3}(\.\d{3})*(,\d{2})$|^\d+([.,]\d{2})$


	•	Datumsformat nicht in `YYYY-MM-DD` oder `DD.MM.YYYY`:

^\d{4}-\d{2}-\d{2}$|^\d{2}\.\d{2}\.\d{4}$



Wenn `needsManualValidation = True`:
	•	PDF wird geöffnet
	•	Für fehlende Felder werden Input Dialogs angezeigt
	•	Zeitfenster wird mit `valStart/valEnd` gemessen → `reworkSeconds`
	•	`isSTP = False`

b) 5‑% Zufalls‑Stichprobe
	•	`rand = (new Random(Guid.NewGuid().GetHashCode())).NextDouble()`
	•	Falls `rand < 0.05`: manuelle Plausibilitätsprüfung über drei Input Dialogs
	•	Zeitmessung analog a) → `reworkSeconds += …`; `isSTP = False`

5.3 Definition von STP
	•	`STP = True` genau dann, wenn keine manuelle Nachbearbeitung erfolgte:
`Not needsManualValidation AndAlso reworkSeconds = 0`

5.4 Spalte „Fehlerhaft“ — aktuelles Verhalten und Empfehlung
	•	Aktuelles Verhalten (Code‑Stand):
	•	`hasBussinesError` (Bool) wird nur gesetzt, wenn mindestens eine der temporären Eingaben `tmpInvoiceNumber/tmpAmount/tmpInvoiceDate` ungleich `“0”` ist:
If (tmpInvoiceNumber <> "0" OrElse tmpAmount <> "0" OrElse tmpInvoiceDate <> "0") Then hasBussinesError = needsManualValidation
	•	In der „Fehlerhaft“‑Spalte (vom Typ string) wird dieser Bool‑Wert geschrieben.
	•	Konsequenz: Fälle mit fehlenden Feldern (Zweig 5.2 a) setzen `needsManualValidation = True`, ohne die `tmp*`‑Variablen zu nutzen. Dadurch bleibt `hasBussinesError = False`, obwohl eine Nachbearbeitung erfolgte.
	•	Empfehlung (fachlich konsistent):
	•	Setze immer: `hasBussinesError = needsManualValidation`
	•	Typ der „Fehlerhaft“‑Spalte auf boolean ändern
	•	Optional: „Fehlerhaft“ verwenden, um inhaltliche Abweichungen zu kennzeichnen und technische Fehler weiterhin separat in `Erfolgreich` bzw. `Exception` abzubilden

⸻

6. Fehlerbehandlung
	•	PDF‑Read‑Error (z. B. gescanntes PDF ohne Text):
`isSuccessful = False`, `isSTP = False`, Felder leer, Meldung in `Exception`
	•	Sonstige Ausnahmen innerhalb der Extraktion/Validierung sind im aktuellen Stand nicht gesondert abgefangen.

⸻

7. Ausgabeziel (Google Sheets)
	•	Aktivität: Write Range (Connections) aus `UiPath.GSuite.Activities`
	•	Schreibmodus: `Overwrite` des Bereichs `Tabellenblatt1`
	•	Verbindung: über UiPath Connection Service (im XAML durch `ConnectionId` und Browser‑Auswahl referenziert)
	•	Hinweis: Bei großen Korpora führt das wiederholte Überschreiben nach jeder Datei zu unnötigen Schreibzugriffen. Alternative: Batch‑Flush am Ende oder Append‑Strategie.

⸻

8. Reproduzierbarkeit und Parametrisierung

Empfohlene Anpassungen für Experiment‑Runs:
	1.	In‑Argumente nutzen
	•	Entferne die Assigns für `inputFolder` und `logPath`
	•	Binde `in_InputFolder` und `in_LogPath` in den Aktivitäten ein
	2.	Deterministische Stichprobe
	•	Ersetze den Zufall mit festem Seed pro Lauf oder verwende eine deterministische Auswahlregel (z. B. jeder 20. Beleg)
	3.	Log‑Schreibstrategie
	•	Schreibe einmal am Ende des Durchlaufs oder nutze Append
	4.	„Fehlerhaft“‑Semantik korrigieren
	•	Siehe 5.4
	5.	Versionskennung
	•	Schreibe `in_RunLabel` zusätzlich als Spalte in `dt_Log`

⸻

9. Einrichtung Schritt für Schritt
	1.	Projekt öffnen in UiPath Studio
	2.	Pakete prüfen (Manage Packages) und auflösen
	3.	Google‑Verbindung herstellen
	•	In Studio: Data Service / Connections → G‑Sheets Verbindung anlegen
	•	Zugriff auf das Zieldokument gewähren
	•	In der Aktivität Write Range (Connections) das Ziel‑Spreadsheet auswählen
	4.	Input‑Ordner festlegen
	•	Entweder Assign anpassen oder (empfohlen) `in_InputFolder` aktivieren
	5.	Testlauf mit kleiner Stichprobe
	6.	Hauptlauf
	•	Für die Stufen 0 %, 15 %, 30 %, 45 %, 60 % jeweils separat starten

⸻

10. Metrikdefinitionen (für Auswertung in Kap. 5/6 der Arbeit)
	•	Bearbeitungszeit [s]: `(Endzeit − Startzeit).TotalSeconds`
	•	Nachbearbeitungszeit [s]: Summe aller Zeitfenster, die durch manuelle Eingaben entstehen (inkl. 5‑% Stichprobe)
	•	STP [bool]: True, wenn keine manuelle Nachbearbeitung; sonst False
	•	Erfolgreich [bool]: False nur bei technischem Fehler (aktueller Stand). Inhaltliche Abweichungen werden über „Fehlerhaft“ empfohlen abzubilden

⸻

11. Bekannte Einschränkungen
	•	Kein OCR → gescannte PDFs werden nicht verarbeitet
	•	Enge Regex‑Vorgaben → geringe Toleranz gegenüber Layout/Label‑Varianten
	•	Nicht deterministische Stichprobe (GUID‑basierter Zufall)
	•	Hart verdrahtete Pfade und Blattnamen
	•	„Fehlerhaft“ inkonsistent (siehe 5.4)
	•	Datentyp „Fehlerhaft“ ist string, obwohl ein Bool sinnvoller wäre

⸻

12. Qualitätssicherung (Empfehlungen)
	•	Pilotläufe je Stufe (10–20 Belege) mit Labeling der Fehlerursachen
	•	Validierungs‑Protokoll: Manuelle Eingaben protokollieren (Feld, alter Wert, neuer Wert)
	•	Konfigurierbare Regeln: Regex und Formate in eine Konfiguration auslagern
	•	Einheitliche Dezimalnotation: Vor der Analyse Betrag in numerischen Typ überführen ("," → Dezimaltrennzeichen)

⸻

13. Beispiel: Auszug einer Log‑Zeile

Dateiname:          INV_000123.pdf
Startzeit:          2025-09-12T10:44:12
Endzeit:            2025-09-12T10:44:13
Bearbeitungszeit:   0.89
Nachbearbeitungszeit: 0
Rechnungsnummer:    RN-2025-000123
Betrag:             1.234,56
Datum:              2025-08-12
Erfolgreich:        True
STP:                True
Fehlerhaft:         False
Exception:          
Verfahren:          RPA


⸻

14. Änderungsvorschläge (konkret)
	1.	Argumente aktivieren
	•	Ersetze in der Startsequenz:
	•	`inputFolder = in_InputFolder`
	•	`logPath    = in_LogPath`
	2.	„Fehlerhaft“ vereinheitlichen
	•	Vor dem Logging: `hasBussinesError = needsManualValidation`
	•	Schema: Spalte „Fehlerhaft“ → boolean
	3.	STP‑Definition im Log sichtbar machen
	•	Zusatzspalte `ValidationReason` (z. B. MissingField, AmbiguousInvoiceNumber, RandomSample)
	4.	Schreibfrequenz reduzieren
	•	Write Range nur am Ende ausführen oder auf Append umstellen
	5.	Regex robuster gestalten
	•	Alternativen je Label (`Rechnung Nr.`, `Re.-Nr.`) und Währungen berücksichtigen

⸻

15. Lizenz & Autorenschaft
	•	Interner Forschungs‑Workflow im Rahmen der Bachelorarbeit von Paul Schmidt (TH Brandenburg)
	•	Drittanbieter‑Pakete gemäß deren Lizenzen

⸻

16. Kontakt
	•	Rückfragen zur Umsetzung, Parametrisierung oder Auswertung bitte an den Autor der Arbeit richten.

⸻

IPA_InvoiceProcessing — Document Understanding

Kontext. Dieser Abschnitt spiegelt Aufbau Struktur Detaillierungsgrad des RPA‑Teils wider, damit beide Workflows methodisch gleich dokumentiert sind.

⸻

1. Ziel und Geltungsbereich
	•	Ziel: KI‑gestützte Extraktion von Rechnungsnummer, Bruttosumme, Rechnungsdatum aus textbasierten sowie gescannten PDFs. Messung von Bearbeitungszeit und Nachbearbeitungszeit, Bestimmung von STP und Protokollierung in eine Log‑Tabelle.
	•	Geltungsbereich: Unterstützung für Scans durch OCR (UiPath Document OCR, Sprachen deu und eng).
	•	Experimentelle Rolle: IPA‑Variante mit UiPath Document Understanding zur Gegenüberstellung mit der RPA‑Baseline über die Stufen 0 %, 15 %, 30 %, 45 %, 60 % unstrukturierte Eingaben.

⸻

2. Systemvoraussetzungen
	•	UiPath Studio mit IntelligentOCR / Document Understanding Paketen
	•	Pakete (Auszug):
	•	UiPath.IntelligentOCR.Activities, UiPath.DocumentProcessing.Contracts
	•	UiPath.DocumentUnderstanding.ML, UiPathDocumentOCR
	•	UiPath.System.Activities, UiPath.GSuite.Activities (falls Sheets genutzt werden)
	•	DU‑Backends: UiPath AI Center Skill oder Cloud DU Endpoint für MachineLearningExtractor
	•	Zugriff: Berechtigung auf Ziel‑Spreadsheet oder Dateiablage

⸻

3. Eingaben, Variablen und Datenmodell

3.1 In‑Argumente

Wie im RPA‑Teil sind In‑Argumente definiert bzw. empfohlen. Für reproduzierbare Läufe sollten hart verdrahtete Assigns entfernt werden.

	•	in_RunLabel : String — Lauf‑Bezeichnung (z. B. Stufe_30_Unstructured).
	•	in_LogPath : String — Ziel für das Log (Tabelle Datei Blattname je nach Ziel).
	•	in_InputFolder : String — Eingangsordner mit PDFs je Stufe.
	•	(optional empfohlen) in_ConfThresh_InvoiceNumber : Double in_ConfThresh_Amount : Double in_SampleRate : Double.

3.2 Interne Variablen (Auswahl)
	•	Pfade/Dateien: inputFolder : String files : String[]
	•	Zeiten: startTime endTime durationSeconds valStart valEnd reworkSeconds
	•	Extraktion (final): invoiceNumber : String amount : Double invoiceDate : DateTime
	•	Rohwerte/Hilfen: ocrText : String rawAmount : String amountFormatted : String
	•	Confidence/Gating: confInvoiceNumber : Double confTotal : Double needsValidation : Bool forceValidation : Bool
	•	Flags: isSTP : Bool hasBusinessError : Bool isSuccessful : Bool exceptionMessage : String
	•	DU‑Objekte: taxonomy documentObject extractionResults

3.3 Log‑Tabelle (dt_Log)

Schema analog zum RPA‑Workflow mit Verfahren = "IPA":

Spalte	              Typ	        Bedeutung
Dateiname	            string	    Dateiname der Rechnung
Startzeit	            dateTime	  Beginn der Verarbeitung
Endzeit	              dateTime	  Ende der Verarbeitung
Bearbeitungszeit	    double	    (Endzeit − Startzeit) in Sekunden
Nachbearbeitungszeit	double	    Summe der Validierungszeiten
Rechnungsnummer	      string	    Extrahierter bzw. validierter Wert
Betrag	              string	    Formatiert in de‑DE mit zwei Dezimalstellen
Datum	                dateTime  	Rechnungsdatum als DateTime
Erfolgreich	          boolean    	False bei technischem Fehler
STP	                  boolean    	True wenn keine Nachbearbeitung 
Fehlerhaft	          boolean	    True bei Abweichung validiert vs. extrahiert
Exception	            string	    Fehlertext bei Exception
Verfahren	            string	    Statischer Wert "IPA"


⸻

4. Prozessüberblick
	1.	Initialisierung
	•	inputFolder setzen files = Directory.GetFiles(inputFolder, "*.pdf")
	•	dt_Log erzeugen Zeilen leeren
	2.	Schleife über PDFs
	1.	Zeitstart: startTime = Now reworkSeconds = 0 isSTP = True
	2.	Digitize Document: UiPath Document OCR (deu+eng) für Scans sowie textbasierte PDFs
	3.	Load Taxonomy → Data Extraction Scope mit MachineLearningExtractor
	4.	Confidence ermitteln Beträge normalisieren amount : Double amountFormatted
	5.	Gating & Stichprobe needsValidation berechnen forceValidation anwenden (5‑% Probe)
	6.	Validation Station bei Bedarf Werte prüfen übernehmen Zeitfenster messen
	7.	STP bestimmen Zeitende berechnen durationSeconds
	8.	Logging AddDataRow Write Range ins Ziel

⸻

5. Extraktion Prüfregeln und Metriken

5.1 OCR und Digitize‑Einstellungen
	•	Engine: UiPath Document OCR
	•	Sprachen: deu eng
	•	Apply OCR on PDF: Auto ForceApplyOCR: False (Text‑PDFs werden direkt gelesen)

5.2 ML‑Extraktion (Data Extraction Scope)
	•	Extractor: MachineLearningExtractor mit Skill InvoiceMl
	•	Feldmapping: invoiceNumber amount invoiceDate in der Taxonomie
	•	Timeout/Hinweise: ausreichend hoch wählen damit große Scans verarbeitet werden

5.3 Gating Konfidenzen Stichprobe
	•	Konfidenzschwellen (Beispielwerte): 0,85 für invoiceNumber und amount
	•	Regel für needsValidation:

needsValidation = (confInvoiceNumber < Schwelle)
               ODER (confTotal < Schwelle)
               ODER forceValidation


	•	5‑% Zufallsprobe: forceValidation = (New Random(Guid.NewGuid().GetHashCode())).NextDouble() < 0.05
	•	Betragsnormalisierung: rawAmount nach de‑DE bzw. invariant parsen Rundung auf 2 Dezimalstellen

5.4 Definition von STP
	•	STP = True genau dann wenn keine Validierung stattfand:
Not needsValidation AndAlso reworkSeconds = 0

5.5 Semantik „Fehlerhaft“
	•	True wenn validierte Werte von der initialen Extraktion abweichen
	•	Toleranzbetrag ±0,01 Datum tagesgenau

⸻

6. Fehlerbehandlung
	•	OCR‑Fehler/Timeout: isSuccessful = False exceptionMessage setzen Felder leeren isSTP = False
	•	Extractor/Endpoint‑Fehler: gleiches Verfahren Logging der Meldung zur Nachvollziehbarkeit

⸻

7. Ausgabeziel
	•	Write Range in Excel (z. B. Logs/ipa_log.xlsx) oder Google Sheets über Connection Service
	•	Empfehlung: Gemeinsames Ziel für RPA und IPA um die Auswertung zu vereinheitlichen

⸻

8. Reproduzierbarkeit und Parametrisierung
	1.	Argumente nutzen statt hart verdrahteter Assigns (in_InputFolder in_LogPath in_RunLabel)
	2.	Deterministische Stichprobe: festen Seed oder regelbasiert (z. B. jeder 20. Beleg)
	3.	Schwellen als Parameter: in_ConfThresh_* pro Feld
	4.	Secrets in Assets: DU Endpoint und API‑Key nicht im XAML

⸻

9. Einrichtung Schritt für Schritt
	1.	Projekt öffnen Pakete auflösen
	2.	AI‑Skill/Endpoint in Orchestrator anlegen Asset für API‑Key konfigurieren
	3.	Taxonomie mit invoiceNumber amount invoiceDate laden/prüfen
	4.	in_InputFolder je Stufe setzen
	5.	Testlauf mit wenigen PDFs danach Hauptläufe pro Stufe
	6.	Log‑Ziel prüfen Schreibrechte verifizieren

⸻

10. Metrikdefinitionen
	•	Bearbeitungszeit [s] (Endzeit − Startzeit).TotalSeconds
	•	Nachbearbeitungszeit [s] Summe der Validierungsfenster
	•	STP [bool] siehe 5.4
	•	Erfolgreich [bool] False bei technischem Fehler
	•	Fehlerhaft [bool] True bei Abweichung nach Validierung

⸻


11. Beispiel: Auszug einer Log‑Zeile (IPA)

Dateiname:          INV_004201.pdf
Startzeit:          2025-09-12T11:03:41
Endzeit:            2025-09-12T11:03:45
Bearbeitungszeit:   3.78
Nachbearbeitungszeit: 12.41
Rechnungsnummer:    RN-2025-004201
Betrag:             982,40
Datum:              2025-08-28
Erfolgreich:        True
STP:                False
Fehlerhaft:         True
Exception:          
Verfahren:          IPA


⸻

12. Lizenz & Autorenschaft
	•	Interner Forschungs‑Workflow im Rahmen der Bachelorarbeit von Paul Schmidt (TH Brandenburg)
	•	Drittanbieter‑Pakete gemäß deren Lizenzen

