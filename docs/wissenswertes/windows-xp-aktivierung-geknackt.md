# Windows XP: Aktivierungsalgorithmus nach 21 Jahren geknackt

!!! info "Erstellt"
    Von Andy, 10. März 2026 · Übersetzung des Originalartikels von **Kevin Purdy**, [26. Mai 2023](https://arstechnica.com/gadgets/2023/05/a-decade-after-it-mattered-windows-xps-activation-algorithm-is-cracked/) auf Ars Technica

---

![Windows XP Bliss-Hintergrundbild: grüne Hügel unter blauem Himmel](img/xp-bliss-landscape.jpg)
*Das berühmte „Bliss"-Foto von Charles O'Rear/Microsoft – das meistgesehene Foto in der Menschheitsgeschichte*

---

## Kurzfassung

Ein Entwickler hat das Aktivierungsverfahren von Windows XP vollständig rückentwickelt – **21 Jahre** nach der Markteinführung und **9 Jahre nach dem Ende des Support-Zeitraums**. Das Ergebnis: Ein winziges Tool (`xp_activate32.exe`, gerade einmal 18.432 Bytes), das aus dem Telefonaktivierungscode eine gültige Bestätigungs-ID generiert – **komplett offline**, ohne Server, ohne Internet.

---

## Was das bedeutet

Wer eine legitime Windows-XP-Lizenz besitzt und das Betriebssystem neu installiert, stand bisher vor einem Problem: Microsofts Aktivierungsserver sind abgeschaltet. Die Telefonaktivierung als letzter Ausweg funktioniert zwar noch über die Hotline – aber nicht mehr automatisiert.

Genau hier setzt das neue Tool an: Es berechnet die **Confirmation ID** lokal, ohne externe Abhängigkeiten.

!!! warning "Wichtiger Hinweis"
    XP ist seit 2014 ohne Sicherheitsupdates. Das Tool dient der Nutzung in isolierten Umgebungen (VMs, Retro-Computing, Museen, Industrieanlagen) – **nicht für den Alltagseinsatz im Internet**.

---

## Der Weg zur vollständigen Offline-Lösung

### Was vorher existierte

Das Thema ist nicht neu. Bereits seit Jahren gibt es Tools und Patches, die mit dem Aktivierungssystem von Windows XP umgehen – aber keines davon war wirklich vollständig offline:

- **Statische Keys**: Funktionieren bei manchen Editionen, aber nicht bei allen
- **WindowsXPKg** (GitHub): Generiert Installations-IDs, benötigt aber für die Confirmation ID einen externen Server – der heute nicht mehr existiert
- **Inoffizielle Patches**: Umgehen die Aktivierungsprüfung ganz, sind aber technisch eine andere Kategorie

Das neue Tool schließt diese Lücke: Es **berechnet die Confirmation ID** aus dem Telefonaktivierungscode – rein mathematisch, ohne Netzwerk.

### Die Lösung

[tinyapps' Blog-Post](https://tinyapps.org/blog/202305190700_activate_xp_offline.html) (auf den Ars Technica verweist) dokumentiert die Entdeckung: Das Tool `xp_activate32.exe` wurde von **Enderman** entwickelt und löst das Problem vollständig.

**Technischer Ablauf:**

1. Windows XP aufsetzen, aber **nicht** mit dem Internet verbinden
2. Aktivierung starten → „Telefonaktivierung" wählen
3. Installations-ID (die langen Ziffernblöcke auf dem Bildschirm) notieren
4. `xp_activate32.exe` ausführen, Installations-ID eingeben
5. Das Tool berechnet die **Confirmation ID** → in Windows eingeben → aktiviert ✅

Das Ganze passiert in Millisekunden, lokal, ohne Server.

---

## Warum XP überhaupt noch relevant ist

XP mag veraltet sein – aber die Gründe, es zum Laufen zu bringen, sind real:

- **Industrieanlagen**: Viele Steuerungssysteme (SCADA, CNC) laufen noch auf XP – isoliert vom Internet, aber wartungsbedürftig
- **Retro-Gaming**: Spiele, die unter modernen Windows-Versionen nicht laufen
- **Softwareentwicklung**: Testen von Legacy-Anwendungen
- **Museen & Archivierung**: Historische Software-Demos am Originalsystem
- **Windows XP Mode**: Microsofts eigenes Kompatibilitätsfeature für Windows 7 basiert auf XP-Virtualisierung

---

## Technische Details zum Aktivierungsmechanismus

Der Aktivierungsalgorithmus von XP basiert auf asymmetrischer Kryptographie. Microsoft signiert legitime Bestätigungs-IDs mit einem privaten Schlüssel; Windows prüft die Signatur mit dem öffentlichen Gegenstück.

Was Enderman gelang, war das **vollständige Reverse Engineering** dieser Mathematik – nicht das Brechen des Schlüssels, sondern das Verstehen der gesamten Berechnungslogik. Das erlaubt die korrekte Erzeugung einer gültigen Confirmation ID aus einer gegebenen Installations-ID.

Die Dateigröße des Tools sagt einiges: **18.432 Bytes**. Das ist kleiner als die meisten PNG-Icons auf einer modernen Website. Die gesamte kryptographische Logik passt in weniger als 18 KB – was einerseits die Eleganz der Lösung unterstreicht, andererseits zeigt, wie alt und überschaubar diese Kryptographie heute wirkt.

---

## Reaktionen & Einordnung

Der Artikel auf Ars Technica erschien fast auf den Tag genau zum **22. Geburtstag** von Windows XP (Oktober 2001 Release, Mainstream-Support bis 2009). Dass dieser Algorithmus erst jetzt vollständig geknackt wurde, ist eigentlich erstaunlich – zeigt aber auch, dass der Anreiz lange fehlte.

Für echte Sicherheitsrisiken sind andere Windows-Versionen relevanter. XP ist schlicht zu alt, um für aktive Angreifer interessant zu sein – was paradoxerweise erklärt, warum sich die Community erst jetzt die Zeit nimmt, den Aktivierungsmechanismus vollständig zu entschlüsseln.

---

## Fazit

Ein schönes Stück Reverse-Engineering-Geschichte: Das Aktivierungsverfahren von Windows XP – über 20 Jahre lang ein Mix aus Telefonhotlines, Patches und halbgaren Workarounds – ist jetzt vollständig verstanden und kann lokal reproduziert werden. Für alle, die XP aus legitimen Gründen in isolierten Umgebungen betreiben, ist das eine echte Erleichterung.

Und für alle anderen: Ein faszinierendes Beispiel dafür, was Reverse-Engineering-Enthusiasten leisten können – lange nachdem die ursprünglichen Entwickler das Thema abgehakt haben.

---

## Originalquelle

➡️ **[arstechnica.com – „Green hills forever: Windows XP activation algorithm cracked after 21 years"](https://arstechnica.com/gadgets/2023/05/a-decade-after-it-mattered-windows-xps-activation-algorithm-is-cracked/)** (englisch, Kevin Purdy, 26. Mai 2023)

---

*Erstellt von Andy – 10. März 2026*
