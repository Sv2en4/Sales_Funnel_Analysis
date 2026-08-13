# E-Commerce Sales Funnel Analyse — Wo der Umsatz verloren geht und woher er kommt

> **Business-Frage:** Wie effizient ist der Sales Funnel, wo gehen die meisten Kunden verloren und welcher Akquisekanal rechtfertigt sein Budget?
> **Antwort:** Der Checkout ist nicht das Problem — er konvertiert von Checkout-Start bis Kauf mit rund 74%, der letzte Schritt (Payment → Kauf) sogar mit 92%. **82,5% aller verlorenen Kunden gehen an einem einzigen Schritt verloren: View → Add-to-Cart.** Genau dort — und nur dort — unterscheiden sich auch die Akquisekanäle: Email konvertiert View→Purchase mit 33,9%, Social mit 6,7% (Faktor 5,1). Ein großer Teil des bezahlten Social-Traffics wird damit vermutlich unterhalb der Rentabilitätsschwelle eingekauft.

---

## 1. Business-Kontext

**Die Rolle, aus der heraus ich analysiere:** Analyst zur Unterstützung eines E-Commerce Growth- bzw. Performance-Marketing-Teams.

Diese Analyse geht nicht nur auf einzelne Zahlen und Conversion-Rates ein,
da diese Zahlen alleine wertlos für Entscheidungen sind, sondern erklärt
*welche* Maßnahme den meisten Umsatz bringen könnten und gibt Empfehlungen.
Dadurch soll vermieden werden, dass ein Team beispielsweise ein ganzes
Quartal in das Redesign einer Checkout-Seite steckt, die bereits einwandfrei
funktioniert, oder Budget in einen Akquisekanal schiebt, der zuverlässig
Schaufensterbummler liefert.

**Entscheidungen, die diese Analyse stützen soll:**
- Wo soll UX/Engineering den Funnel optimieren und wo ausdrücklich *nicht*?
- Wie sollte das Paid-Budget über die Traffic-Quellen neu verteilt werden?
- Welche maximalen Customer Acquisition Costs (CAC) können wir pro Kanal zahlen, bevor eine Transaktion unprofitabel wird?

**Warum das kommerziell relevant ist:** Ein Prozentpunkt mehr bei View→Cart bewegt absolut mehr Umsatz als fünf Prozentpunkte am unteren Ende des Funnels — schlicht wegen des Volumenunterschieds an der Spitze.

---

## 2. Die Daten

| Attribut | Wert |
|---|---|
| **Quelle** | user_events table; Datensatz eines E-Commerce Sales Funnels |
| **Granularität** | Eine Zeile = ein User-Event |
| **Zeitraum** | Analysefenster: die letzten 30 Tage, die im Datensatz dokumentiert wurden |
| **Volumen** | 9381 Events mit 5000 individuellen Usern |
| **Aktualisierung** | Statischer Snapshot (siehe Limitationen) |

### Schema

| Spalte | Typ | Beschreibung |
|---|---|---|
| `user_id` | STRING | Eindeutige Besucher-ID |
| `event_type` | STRING | Einer von: `page_view`, `add_to_cart`, `checkout_start`, `payment_info`, `purchase` |
| `event_date` | TIMESTAMP | Zeitstempel des Events (dient sowohl der Datumsfilterung als auch den `TIMESTAMP_DIFF`-Berechnungen) |
| `traffic_source` | STRING | Akquisekanal — 4 verschiedene Ausprägungen (Email, Social, Organic, Paid-Ads |
| `amount` | NUMERIC | Bestellwert; nur bei `purchase`-Events befüllt |

### Funnel-Definition

| Stage | Event | Bedeutung |
|---|---|---|
| 1 | `page_view` | Besucher landet auf einer Produktseite |
| 2 | `add_to_cart` | Erklärte Kaufabsicht |
| 3 | `checkout_start` | Kaufprozess betreten |
| 4 | `payment_info` | Zahlungsdaten hinterlegt |
| 5 | `purchase` | Transaktion abgeschlossen |

Die Stages werden als `COUNT(DISTINCT user_id)` gezählt, nicht als Event-Anzahl — wer drei Artikel in den Warenkorb legt, ist trotzdem ein User in Stage 2. Das verhindert, dass die mittleren Stages gegenüber der Funnel-Spitze künstlich aufgebläht werden.

---

## 3. Methodik

Die Analyse besteht aus fünf aufeinander aufbauenden SQL-Queries in BigQuery. Jede beantwortet die Frage, die die vorherige aufgeworfen hat:

| # | Query | Beantwortete Frage | Warum diese Methode |
|---|---|---|---|
| 1 | Funnel-Stage-Counts | Wie viele User erreichen jede Stage? | Baseline. Legt die Form des Funnels fest, bevor irgendeine Rate berechnet wird. |
| 2 | Schrittweise + Gesamt-Conversion-Rates | *Wo* ist der größte relative Abfall? | Absolutzahlen verbergen Effizienz. Schrittweise Raten isolieren das schwächste Glied; die Gesamtrate liefert die Headline-Zahl. |
| 3 | Funnel segmentiert nach `traffic_source` | Unterscheidet sich das Leck je Kanal? | Ein aggregierter Funnel mittelt Kanalunterschiede weg. Die Segmentierung macht aus der deskriptiven Analyse eine handlungsfähige. |
| 4 | Time-to-Conversion (`TIMESTAMP_DIFF` auf das jeweils erste Event pro Stage) | Wie lange dauert eine Kaufentscheidung? | Dient zugleich als **Datenqualitätsprüfung** — unplausibel kurze Zeitspannen würden auf Bot-Traffic hindeuten und alles Vorherige entwerten. |
| 5 | Revenue Funnel (AOV, Umsatz pro Käufer, Umsatz pro Besucher) | Was ist jede Funnel-Stage in Geld wert? |

**Design-Entscheidungen:**
- **Rollierendes 30-Tage-Fenster, verankert an `MAX(event_date)`** statt eines hartkodierten Datums — die Query bleibt korrekt, wenn neue Daten hinzukommen.
- **`COUNT(DISTINCT user_id)` statt `COUNT(*)`** — der Funnel misst Menschen, keine Aktionen.
- **`MIN(event_date)` pro Stage** in der Time-to-Conversion-Query — erfasst den *ersten* Zeitpunkt, an dem ein User eine Stage erreicht, damit wiederholte Besuche die Dauer nicht verzerren.

**Tools:** BigQuery (SQL) → Tableau Public für Funnel- und Kanal-Visualisierungen*

---

## 4. Zentrale Erkenntnisse

**1. Der Checkout ist der stärkste Teil des Funnels.**
Checkout-Start → Kauf konvertiert mit **74,4%**, wobei Payment-Info → Kauf mit **92,2%** die höchste schrittweise Rate im gesamten Funnel aufweist. Hier gibt es keine technische Reibung zu beheben.

**2. 82,5% aller verlorenen User gehen an einem einzigen Schritt verloren: View → Add-to-Cart.**
Von 4.268 Besuchern erreichen nur 1.332 den Warenkorb — 2.936 verlorene User an diesem Schritt gegenüber 3.560 Verlusten im gesamten Funnel. Ein Teil davon ist erwartbar (reines Browsing), aber es ist zugleich der Schritt mit dem meisten Volumen dahinter — und damit dem höchsten absoluten Potenzial. Die Gesamt-Conversion View→Purchase liegt bei 16,6%.

**3. Email konvertiert mit 33,9% View→Purchase, Social mit 6,7% — ein Effizienzunterschied von Faktor 5,1.**
Email ist mit Abstand der stärkste Kanal, Social der schwächste. Der Unterschied ist statistisch eindeutig (Zwei-Proportionen-Test, z = 14,3, p < 0,001) und damit kein Rauschen.

| Kanal | Views | View→Cart | Cart→Purchase | View→Purchase |
|---|---|---|---|---|
| Email | 445 | 62,9% | 53,9% | **33,9%** |
| Paid Ads | 820 | 37,2% | 56,7% | 21,1% |
| Organic | 1.750 | 32,9% | 52,1% | 17,1% |
| Social | 1.253 | 13,6% | 49,1% | **6,7%** |

**4. Social liefert 29,4% des gesamten Traffics, aber nur 11,9% aller Käufe.**
Hohes Volumen, geringe Kaufabsicht. Das klassische Profil von Traffic, der auf ein Reichweiten- statt auf ein Conversion-Ziel eingekauft wurde. Zum Vergleich: Email stellt 10,4% der Views, aber 21,3% der Käufe.

**4b. Der Kanal entscheidet ausschließlich am ersten Schritt — danach verhalten sich alle User gleich.**
Cart→Purchase liegt bei allen vier Quellen eng beieinander (49,1% bis 56,7%), während View→Cart von 13,6% bis 62,9% streut. Sobald ein User etwas in den Warenkorb legt, ist seine Herkunft für den Kaufabschluss praktisch irrelevant. Der gesamte Kanalunterschied ist also ein Unterschied in der *Kaufabsicht beim Ankommen*, nicht in der Qualität der Käufer. Das verbindet Erkenntnis 2 und 3: View→Cart ist zugleich der größte Verlustpunkt und die einzige Stelle, an der die Kanäle auseinanderlaufen.

**5. Der durchschnittliche Bestellwert liegt bei ca. 107€, die Gesamtdauer bis zum Kauf bei 25 Minuten.**
Der zeitliche Kaufverlauf wirkt menschlich, nicht automatisiert — das validiert den Datensatz. Der AOV setzt zugleich die Obergrenze dafür, was wir pro gewonnenem Kunden zahlen dürfen.

<img width="380" height="741" alt="Bildschirmfoto 2026-08-12 um 10 38 11" src="https://github.com/user-attachments/assets/96030e90-5b85-42a7-80df-514d1b39ca9a" />

<img width="792" height="741" alt="Bildschirmfoto 2026-08-12 um 10 48 53" src="https://github.com/user-attachments/assets/77c26fa9-888a-4468-9fc0-b507632b8588" />

<img width="706" height="741" alt="Bildschirmfoto 2026-08-12 um 10 58 49" src="https://github.com/user-attachments/assets/90e40be2-b253-42cd-a574-3a0f1077da91" />



Tableau Public Link: https://public.tableau.com/views/SalesFunnelAnalyse/Blatt3?:language=de-DE&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link

---

## 5. Empfehlungen

| # | Empfehlung | Verantwortlich | Erwartete Wirkung |
|---|---|---|---|
| 1 | **Den Checkout-Flow in diesem Quartal nicht anfassen.** Er konvertiert mit 74,4% (Payment→Kauf sogar 92,2%); ein Redesign riskiert, ein funktionierendes System ohne Aufwärtspotenzial zu beschädigen. | Product / Engineering | Vermiedene Kosten + vermiedenes Regressionsrisiko |
| 2 | **A/B-Test auf der Produktlistenseite** — konkret Suchfilter und Produktbilder — mit Fokus auf den View→Cart-Schritt. | UX / Product | +1pp View→Cart ≈ 2436€ zusätzlicher Monatsumsatz |
| 3 | **Paid-Social-Budget von „Traffic"- auf „Lead-Generation"-/Email-Capture-Ziele umschichten.** Nicht aufhören, Social-Traffic einzukaufen — aufhören, von ihm direkte Conversions zu erwarten. | Performance Marketing | Macht aus einem 6,7%-Kanal einen Zubringer für einen 33,9%-Kanal |
| 4 | **Email-Capture-Prompt gezielt für Social-Besucher ausspielen.** Die Daten zeigen, dass diese User später kaufen, sobald sie im Verteiler sind. | Growth / CRM | ≈ 1.094 € zusätzlicher Monatsumsatz bei angenommener Capture-Rate von 3% |
| 5 | **Harte CAC-Obergrenze pro Kanal festlegen, gemessen an AOV und Deckungsbeitrag.** Bei einem AOV von ca. 107 € ist es bei einem Kanal mit 6,7% Conversion vermutlich verlustbringend, mehr als 30–40 € pro Kunde zu zahlen. | Finance + Performance Marketing | Schützt den Deckungsbeitrag |

Empfehlung 5 beruht derzeit auf einer angenommenen Marge. **Nächster Schritt: tatsächliche Kanal-Spendings beschaffen, um echte CAC zu berechnen und die Aussage zu bestätigen.**

---

## 6. Limitationen & Annahmen

**Was die Daten nicht hergeben:**
- **Keine Kostendaten.** Jede Aussage zu CAC und Profitabilität ist eine Hypothese, solange die Werbeausgaben pro Kanal nicht angejoint sind. Das ist die größte Lücke der Analyse.
- **Keine Produktebene.** Ich kann sagen, dass User nicht in den Warenkorb legen; ich kann nicht sagen, ob es am Preis, an den Bildern, an der Verfügbarkeit oder an der Suchrelevanz liegt.
- **Keine Sessions.** Events hängen an der `user_id` ohne Session-Abgrenzung. Ein User, der über zwei Wochen hinweg browst, wird genauso behandelt wie einer, der in einem einzigen Besuch konvertiert.
- **Ausschließlich Last-Touch-Attribution.** `traffic_source` ist ein einzelner Wert pro Event. Wer über Social entdeckt und über Email zurückkehrt, wird vollständig Email zugerechnet — das **unterschätzt** Socials Beitrag vermutlich und ist ein direkter Vorbehalt gegenüber Empfehlung 3 und 4.
- **Keine Retouren oder Rückerstattungen**, die Umsatzzahlen sind daher brutto, nicht netto.
- **Kleine Fallzahl im stärksten Kanal.** Email umfasst nur 445 Views. Das 95%-Konfidenzintervall der Conversion liegt bei 29,5%–38,3% — die Richtung ist eindeutig, die exakte Höhe nicht. Ob 33,9% auch bei zehnfachem Volumen halten, sagen diese Daten nicht; Empfehlung 3 und 4 skalieren einen bislang kaum bespielten Kanal.
- **Selbstselektion bei Email.** Die hohe Email-Conversion stammt von Usern, die sich aktiv für den Verteiler entschieden haben. Das ist ein Effekt der Kaufabsicht, nicht des Kanals — eingesammelte Social-Besucher werden sich deshalb voraussichtlich schlechter verhalten als die heutige Email-Kohorte. Empfehlung 4 ist damit ein Szenario, kein Befund.


**Annahmen:**
- Alle User durchlaufen die Stages linear; User, die über direkte Warenkorb-Links mitten im Funnel einsteigen, werden nicht abgebildet.
- Der Datensatz repräsentiert normalen Geschäftsbetrieb (keine Aktionsspitze und kein Ausfall innerhalb des Zeitfensters).

---

## 7. Repo-Übersicht

```
sales-funnel-analysis/
├── README.md
├── sql/
│   ├── 01_funnel_stages.sql
│   ├── 02_conversion_rates.sql
│   ├── 03_traffic_source_funnel.sql
│   ├── 04_time_to_conversion.sql
│   └── 05_revenue_funnel.sql
├── data/
│   ├── raw/                 # Raw-Data als CSV
│   └── processed/           # Query-Outputs als CSV für das Dashboard
├── viz/
│   └── tableau_link.md
```

### Analyse reproduzieren
1. `data/raw/user_events.csv` in ein BigQuery-Dataset laden oder die Queries auf die eigene Tabelle richten.
2. Die vollqualifizierte Tabellenreferenz in jeder Datei unter `sql/` anpassen (aktuell hartkodiert auf `sales-funnel-analysis-504708.sales_funnel_dataset.user_events`).
3. Die Queries in numerischer Reihenfolge ausführen — jede baut auf der Frage auf, die die vorherige aufgeworfen hat.
4. Ergebnisse nach `data/processed/` exportieren und das Tableau-Workbook aktualisieren.

### Voraussetzungen
- Zugang zu Google BigQuery (Sandbox-Tier genügt)
- Tableau Public


