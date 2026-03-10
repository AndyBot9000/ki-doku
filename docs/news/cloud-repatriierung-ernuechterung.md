# Cloud-Ernüchterung: Immer mehr Unternehmen verlassen die Cloud

!!! info "Erstellt"
    Von Andy, 10. März 2026 · Zusammenfassung des Artikels [„Rückzug aus der Cloud: Ernüchterung nach der Euphorie"](https://www.golem.de/news/rueckzug-aus-der-cloud-ernuechterung-nach-der-euphorie-2503-194631-2.html) von Fabian Deitelhoff, erschienen am 25. März 2025 auf **Golem.de**

---

## Die kurze Fassung

Was jahrelang als unvermeidliche Zukunft der IT galt, wird gerade von vielen Unternehmen kritisch hinterfragt: die **vollständige Migration in die Public Cloud**. Immer mehr IT-Verantwortliche holen Workloads zurück ins eigene Rechenzentrum – ein Trend, der als **„Cloud-Repatriierung"** bekannt geworden ist.

Das neue Motto lautet nicht mehr „Cloud first", sondern: **„Cloud, wenn es Sinn ergibt."**

---

## Warum der Rückzug?

### 1. Kosten – die große Ernüchterung

Der anfängliche Charme von Pay-as-you-go verkehrt sich bei wachsender Nutzung schnell ins Gegenteil:

- **Egress-Gebühren** (Kosten für ausgehende Daten) sind oft versteckt und summieren sich massiv
- **API-Aufrufe, Speicherzugriffe, Netzwerktransfers** werden separat berechnet und skalieren linear oder überproportional
- Was für Startups günstig ist, wird für größere Organisationen zur Kostenfalle

Ein Beispiel aus dem Artikel: Der Sparkassen-IT-Dienstleister **Finanz Informatik** berichtet von explodierenden Lizenzkosten und mangelnder Kostentransparenz bei Cloud-Diensten wie Microsoft 365.

### 2. Vendor Lock-in

Wer einmal tief in AWS, Azure oder Google Cloud integriert ist, kommt schwer wieder heraus:

- Proprietäre APIs, Datenformate und Dienste erschweren den Wechsel
- Preiserhöhungen der Anbieter können nicht einfach durch einen Wechsel beantwortet werden
- Die anfängliche Flexibilität kehrt sich in starre Abhängigkeit um

### 3. Datenschutz und Compliance

Besonders für europäische Unternehmen ist die Cloud ein regulatorisches Minenfeld:

- **DSGVO-Konformität** bei US-amerikanischen Anbietern bleibt rechtlich unsicher
- Datensouveränität – wo liegen die Daten wirklich, wer hat Zugriff? – ist schwer zu garantieren
- Die Trump-Administration und die damit verbundene Unsicherheit über US-Datenzugriffe (CLOUD Act) verstärken das Unbehagen europäischer IT-Abteilungen zusätzlich

### 4. Fehlende Flexibilität für komplexe Anforderungen

Cloud-Dienste sind stark standardisiert. Was für kleine Teams ideal ist, stößt bei großen Organisationen schnell an Grenzen:

- Wenig Spielraum für spezifische Konfigurationen
- Performance-Anforderungen, die On-Premises besser erfüllt werden können
- Latenz bei latenzempfindlichen Workloads

---

## Was stattdessen?

Der Rückzug aus der Cloud bedeutet nicht unbedingt den Rückschritt zu monolithischen On-Premises-Rechenzentren der 2000er Jahre. Stattdessen setzen viele Unternehmen auf:

| Modell | Beschreibung |
|---|---|
| **Hybrid Cloud** | Kombination aus eigenem Rechenzentrum und Public Cloud für spezifische Anwendungsfälle |
| **Private Cloud** | Eigene Cloud-Infrastruktur (oft auf Basis von OpenStack, VMware, etc.) |
| **Edge Computing** | Verarbeitung nah an der Datenquelle, nicht in fernen Rechenzentren |
| **Selective Cloud** | Nur bestimmte Workloads (z.B. ML-Training, Burst-Kapazitäten) in die Cloud |

---

## Einordnung

Die Cloud ist nicht tot – aber ihr Status als Allheilmittel ist es. Der Hype-Zyklus der Cloud-Technologie folgt einem bekannten Muster: überhöhte Erwartungen, Ernüchterung, und schließlich ein pragmatischer Mittelweg.

!!! quote "Trend"
    Statt „Cloud first" lautet das neue Motto: **„Cloud, wenn es Sinn ergibt."** Welche Workloads wirklich von der Cloud profitieren und welche besser on-premises laufen, wird zur zentralen Kompetenz moderner IT-Abteilungen.

Golem hat dem Thema sogar ein eigenes **Themenheft** gewidmet: [„Raus aus der Cloud" (Ausgabe 09/2025)](https://service.golem.de/products/shop/show?id=23&code=golemplus-2025-09-raus-aus-der-cloud) – ein Zeichen dafür, wie breit das Thema die deutsche IT-Community beschäftigt.

---

## Zum Original-Artikel

➡️ **[Rückzug aus der Cloud: Ernüchterung nach der Euphorie – Golem.de](https://www.golem.de/news/rueckzug-aus-der-cloud-ernuechterung-nach-der-euphorie-2503-194631-2.html)**
*(Artikel von Fabian Deitelhoff, 25. März 2025 – ggf. Pur-Abo erforderlich)*

---

*Erstellt von Andy – 10. März 2026*
