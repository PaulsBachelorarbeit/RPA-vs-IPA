RPA_InvoiceProcessing & IPA_InvoiceProcessing — README

UiPath‑Workflows zur experimentellen Verarbeitung eines PDF‑Korpus für eine Bachelorarbeit. Der RPA‑Workflow bildet die regelbasierte Baseline; der IPA‑Workflow nutzt Document Understanding (OCR + ML‑Extraktion + Validation Station). Beide erfassen identische Metriken.

⸻

Inhaltsverzeichnis
	1.	Ziel und Geltungsbereich
	2.	Systemvoraussetzungen
	3.	Konfiguration & Argumente
	4.	Datenmodell (Log‑Tabelle)
	5.	RPA_InvoiceProcessing (Baseline)
	6.	IPA_InvoiceProcessing (Document Understanding)
	7.	Metriken
	8.	Fehlerbehandlung
	9.	Reproduzierbarkeit & Parametrisierung
	10.	Einrichtung Schritt für Schritt
	11.	Beispiele: Log‑Zeilen
	12.	Bekannte Einschränkungen
	13.	Änderungsvorschläge
	14.	Lizenz & Kontakt

⸻

Ziel und Geltungsbereich

Ziel. Sequenzielle Verarbeitung eines PDF‑Korpus, Extraktion von invoiceNumber, amount (gross) und invoiceDate, Messung von processingTime und reworkTime, Bestimmung von Straight‑Through‑Processing (STP) sowie Protokollierung in eine Log‑Tabelle.

Geltungsbereich.
	•	RPA (Baseline): Textbasierte PDFs ohne OCR. Scans ohne eingebetteten Text führen zu PDF‑Read‑Error und werden als Erfolgreich = False geloggt.
	•	IPA (DU): Textbasierte und gescannte PDFs (OCR: deu+eng), Konfidenz‑Gating, Validierungsstation.

Experimentelle Rolle. Vergleich beider Verfahren über Input‑Stufen mit 0 %, 15 %, 30 %, 45 %, 60 % unstrukturierten Eingaben.

⸻

Systemvoraussetzungen
	•	UiPath Studio (Community/Enterprise), .NET‑basierte Workflows (getestet in 2024/2025‑Versionen)
	•	Pakete (Auszug)
	•	RPA: UiPath.System.Activities, UiPath.UiAutomation.Activities, UiPath.PDF.Activities, UiPath.GSuite.Activities
	•	IPA zusätzlich: UiPath.IntelligentOCR.Activities, UiPath.DocumentProcessing.Contracts, UiPath.DocumentUnderstanding.ML, UiPathDocumentOCR
	•	Framework‑Assemblies: diverse System.* (gemäß XAML ReferencesForImplementation)
	•	Google Workspace: Zugriff auf das Ziel‑Spreadsheet (bei Sheets‑Logging)
	•	DU‑Backend (IPA): UiPath AI Center‑Skill oder Cloud DU Endpoint

⸻

Konfiguration & Argumente

Empfehlung: Hart verdrahtete Assigns entfernen und Argumente verwenden.

Gemeinsame In‑Argumente
	•	in_RunLabel : String — Lauf‑Label (z. B. Stufe_60_Unstructured).
	•	in_LogPath : String — Ziel für das Log (Pfad/Blattname je nach Ziel; optional bei Sheets).
	•	in_InputFolder : String — Eingangs‑Ordner mit PDFs.

Zusätzliche IPA‑Argumente (empfohlen)
	•	in_ConfThresh_InvoiceNumber : Double — Konfidenzschwelle (Standard: 0.85).
	•	in_ConfThresh_Amount : Double — Konfidenzschwelle (Standard: 0.85).
	•	in_SampleRate : Double — Stichprobenrate für zusätzliche Validierung (Standard: 0.05).

⸻

Datenmodell (Log‑Tabelle)

Schema (identisch für RPA/IPA; Spalte Verfahren unterscheidet die Variante):

Spalte	Typ	Bedeutung
Dateiname	string	Dateiname der Rechnung
Startzeit	dateTime	Beginn der Verarbeitung
Endzeit	dateTime	Ende der Verarbeitung
Bearbeitungszeit	double	(Endzeit − Startzeit) in Sekunden
Nachbearbeitungszeit	double	Summe der Validierungszeiten in Sekunden
Rechnungsnummer	string	Extrahierter bzw. validierter Wert
Betrag	string	Formatiert (de‑DE, z. B. 1.234,56)
Datum	string/dateTime	Rechnungsdatum (RPA: String; IPA: DateTime)
Erfolgreich	boolean	False bei technischem Fehler
STP	boolean	True, wenn keine Nachbearbeitung stattfand
Fehlerhaft	boolean	Inh. Abweichung zw. extrahiert und validiert (s. u.)
Exception	string	Fehlertext bei Exception
Verfahren	string	RPA oder IPA
(empf.) ValidationReason	string	Grund für Validierung (z. B. MissingField)
(empf.) RunLabel	string	aus in_RunLabel

Hinweis „Fehlerhaft“. Im ursprünglichen RPA‑Code wurde hasBussinesError inkonsistent befüllt und als string gespeichert. Empfehlung: boolesche Spalte, Wert = needsManualValidation (oder, bei IPA, True wenn validierte Werte abweichen; Toleranz ±0,01).

⸻

RPA_InvoiceProcessing (Baseline)

Prozessüberblick
	1.	Initialisierung: inputFolder/logPath setzen, Dateiliste (*.pdf) ermitteln, dt_Log aufbauen/clearen.
	2.	Zeitstart pro Datei: startTime = Now; reworkSeconds = 0; needsManualValidation = False; isSTP = True.
	3.	PDF lesen: Read PDF Text → pdfText.
	•	Bei Fehler: isSTP=False, isSuccessful=False, Felder leer, exceptionMessage = "PDF-Read-Error: …".
	4.	Regex‑Extraktion (s. unten).
	5.	Regelbasierte Prüfungen → ggf. manuelle Nachbearbeitung (Input Dialogs) und Zeitmessung (valStart/valEnd).
	6.	5‑% Zufallsstichprobe (zusätzliche Plausibilisierung, analog zur Validierung).
	7.	Zeitende & STP: endTime = Now; durationSeconds = (endTime - startTime).TotalSeconds.
	8.	Logging: Add Data Row → Write Range (Sheets). Standardmäßig Overwrite.

Regex‑Extraktion

# Rechnungsnummer
Rechnungsnummer\s*(BT-1):\s*(\S+)

# Bruttosumme (Betrag)
Bruttosumme\s*(BT-115):\s*([\d,.]+)\s*€

# Rechnungsdatum
Rechnungsdatum\s*(BT-2):\s*(\d{4}-\d{2}-\d{2})

Eng an synthetischen Vorlagen. Abweichende Labels/Formate → Validierungsgrund.

Validierungen
	•	Missing Field: Mindestens ein Feld leer.
	•	Ambiguous Invoice Number:

Rechnungsnummer\s*(BT-1):\s*(\S+)|Rechnung\sNr.\s(\S+)


	•	Betragsformat (DE): ^\d{1,3}(\.\d{3})*(,\d{2})$|^\d+([.,]\d{2})$
	•	Datumsformat: ^\d{4}-\d{2}-\d{2}$|^\d{2}.\d{2}.\d{4}$
	•	Bei Validierung: PDF öffnen, Dialogeingaben, reworkSeconds messen, isSTP=False.

STP‑Definition

STP = Not needsManualValidation AndAlso reworkSeconds = 0

⸻

IPA_InvoiceProcessing (Document Understanding)

Pipeline
	1.	Digitize Document: UiPath Document OCR (deu+eng), ApplyOCROnPdf = Auto, ForceApplyOCR = False.
	2.	Load Taxonomy: Finance → IncomingInvoice → Felder invoiceNumber, amount, invoiceDate.
	3.	Data Extraction Scope: MachineLearningExtractor (SelectedMLSkill = "InvoiceMl", Timeout großzügig, UseServerSideOCR = False).
	4.	Konfidenzen & Normalisierung: confInvoiceNumber, confTotal; rawAmount → amount : Double → amountFormatted (de-DE).
	5.	Gating & Stichprobe: needsValidation = (conf < Schwelle) OR forceValidation; forceValidation = Random() < in_SampleRate.
	6.	Present Validation Station (falls nötig): Werte prüfen/übernehmen; reworkSeconds messen.
	7.	STP: Not needsValidation AndAlso reworkSeconds = 0.
	8.	Logging: AddDataRow → WriteRange (Excel oder Sheets). Verfahren = "IPA".

IPA‑Variablen (Auswahl)
	•	Zeiten: startTime, endTime, durationSeconds, valStart, valEnd, reworkSeconds.
	•	Extraktion (final): invoiceNumber : String, amount : Double, invoiceDate : DateTime.
	•	Confidence/Gating: confInvoiceNumber : Double, confTotal : Double, needsValidation : Bool, forceValidation : Bool.
	•	Flags: isSTP, hasBusinessError, isSuccessful, exceptionMessage.
	•	DU‑Objekte: documentObject, taxonomy, extractionResults.

Semantik „Fehlerhaft“ (IPA)

True, wenn validierte Werte von der initialen Extraktion abweichen (Toleranz ±0,01 beim Betrag; Datum auf Tagesebene).

⸻

Metriken
	•	Bearbeitungszeit [s]: (Endzeit − Startzeit).TotalSeconds
	•	Nachbearbeitungszeit [s]: Summe aller Validierungsfenster (inkl. Stichprobe)
	•	STP [bool]: s. Definitionen in den Workflow‑Abschnitten
	•	Erfolgreich [bool]: False bei technischem Fehler
	•	Fehlerhaft [bool]: s. Log‑Semantik oben

⸻

Fehlerbehandlung
	•	RPA: PDF‑Read‑Error (z. B. Scan ohne Text) → isSuccessful=False, isSTP=False, Felder leer, Meldung in Exception.
	•	IPA: OCR-/Extractor-/Endpoint‑Fehler → analoges Vorgehen; Meldung in Exception.

⸻

Reproduzierbarkeit & Parametrisierung
	1.	Argumente aktivieren: inputFolder = in_InputFolder, logPath = in_LogPath.
	2.	Deterministische Stichprobe: festen Seed nutzen oder regelbasiert (z. B. jeder 20. Beleg).
	3.	Schwellen als Parameter (IPA): in_ConfThresh_* pro Feld.
	4.	Schreibstrategie: Batch‑Flush am Ende oder Append statt Overwrite nach jeder Datei.
	5.	Fehlerhaft‑Feld vereinheitlichen: boolescher Typ, Befüllung s. oben.
	6.	Versionskennung: in_RunLabel als zusätzliche Log‑Spalte.
	7.	Secrets: DU Endpoint/API‑Key via Orchestrator Assets/Connection Service.

⸻

Einrichtung Schritt für Schritt
	1.	Projekt in UiPath Studio öffnen.
	2.	Pakete prüfen und auflösen (Manage Packages).
	3.	Google‑Verbindung (bei Sheets): Data Service → Connections, Zugriff auf Ziel‑Spreadsheet gewähren, in Write Range (Connections) referenzieren.
	4.	DU‑Skill/Endpoint (IPA): AI Center/Cloud DU konfigurieren; API‑Key/Endpoint als Asset.
	5.	Input‑Ordner je Stufe setzen (Argument oder Config).
	6.	Testlauf mit kleiner Stichprobe.
	7.	Hauptläufe je Stufe (0/15/30/45/60 %).

⸻

Beispiele: Log‑Zeilen

RPA

Dateiname: INV_000123.pdf
Startzeit: 2025-09-12T10:44:12
Endzeit: 2025-09-12T10:44:13
Bearbeitungszeit: 0.89
Nachbearbeitungszeit: 0
Rechnungsnummer: RN-2025-000123
Betrag: 1.234,56
Datum: 2025-08-12
Erfolgreich: True
STP: True
Fehlerhaft: False
Exception:
Verfahren: RPA

IPA

Dateiname: INV_004201.pdf
Startzeit: 2025-09-12T11:03:41
Endzeit: 2025-09-12T11:03:45
Bearbeitungszeit: 3.78
Nachbearbeitungszeit: 12.41
Rechnungsnummer: RN-2025-004201
Betrag: 982,40
Datum: 2025-08-28
Erfolgreich: True
STP: False
Fehlerhaft: True
Exception:
Verfahren: IPA


⸻

Bekannte Einschränkungen
	•	RPA: kein OCR → gescannte PDFs werden nicht verarbeitet.
	•	Enge Regex‑Vorgaben (RPA) → geringe Toleranz ggü. Label/Format‑Varianten.
	•	Nicht deterministische Zufallsstichprobe (GUID‑basiert).
	•	Hart verdrahtete Pfade/Blattnamen in frühen Ständen.
	•	Inkonsistente Fehlerhaft‑Befüllung im ursprünglichen RPA‑Code.

⸻

Änderungsvorschläge
	•	Argumente aktivieren (Startsequenz):
	•	inputFolder = in_InputFolder
	•	logPath = in_LogPath
	•	Fehlerhaft vereinheitlichen:
	•	Vor Logging: hasBussinesError = needsManualValidation
	•	Schema: Fehlerhaft : boolean
	•	STP/Validierungsgründe sichtbar machen:
	•	Zusatzspalte ValidationReason (z. B. MissingField, AmbiguousInvoiceNumber, RandomSample).
	•	Schreibfrequenz reduzieren:
	•	Write Range nur am Ende oder Append.
	•	Regex robuster (RPA): Alternativen für Labels (z. B. Rechnung Nr., Re.-Nr.) und Währungen berücksichtigen.

⸻

Lizenz & Kontakt
	•	Interner Forschungs‑Workflow im Rahmen der Bachelorarbeit von Paul Schmidt (TH Brandenburg).
	•	Drittanbieter‑Pakete gemäß deren Lizenzen.
	•	Rückfragen zur Umsetzung, Parametrisierung oder Auswertung: bitte an den Autor der Arbeit.
