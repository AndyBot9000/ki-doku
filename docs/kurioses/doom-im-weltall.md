# Doom im Weltall: Wie ein ESA-Satellit zum Spielfeld wurde – und dabei früher abstürzte

!!! info "Erstellt"
    Von Andy, 10. März 2026 · Übersetzung des Originalartikels von **Liam Proven**, [28. Oktober 2025](https://www.theregister.com/2025/10/28/doom_running_in_space/) auf The Register

---

## Die Kurzfassung

Programmierer **Ólafur Waage** hat Doom auf dem **OPS-SAT-Satelliten der Europäischen Weltraumorganisation (ESA)** zum Laufen gebracht – 500 Kilometer über der Erde, im Orbit, ohne Bildschirm und ohne dass irgendjemand es hätte sehen können. Als Himmelstextur wurden echte Satellitenfotos der Erdoberfläche eingebettet.

Die schönste Pointe: **Das Spielen von Doom hat den Satelliten früher zum Absturz gebracht.**

---

## Der Satellit: Ein fliegendes Labor zum Hacken

Der **OPS-SAT** war ein besonderer Satellit: Er wurde von der ESA explizit dafür gebaut, um von externen Entwicklern gehackt zu werden. Ungefähr handgepäckgroß, lief er auf einem **Dual-Core-ARM9-Prozessor** mit – für ein Raumfahrzeug ungewöhnlich – **Ubuntu 18.04 LTS**.

Ein Do-release-upgrade war keine Option. Man kann den Techniker nicht mal eben zum Satelliten schicken.

Der OPS-SAT wurde 2024 kontrolliert deorbitet – also absichtlich zum Verglühen gebracht. Waage präsentierte das Projekt auf dem Ubuntu Summit 2025 als Rückblick.

---

## Die technischen Herausforderungen

### Problem 1: Kein Bildschirm

Der Satellit war unbemannt. Es gab niemanden, der auf einen Monitor geschaut hätte. Waage löste das mit **Headless Doom** – einer Version des Spiels, die komplett ohne Display läuft und stattdessen Logs produziert.

Aber ein Logfile, das nur „OK OK OK OK" enthält, ist wenig beeindruckend.

### Problem 2: Kein Beweis

Wie zeigt man, dass Doom wirklich im Weltraum läuft? Waages Lösung war elegant: Der Satellit hatte eine **Bordkamera**, die die Erde fotografiert. Diese Bilder wurden als **Himmelstextur** in das Doom-Level eingebettet – live, aus dem Orbit.

### Problem 3: Die Farbpalette

Doom verwendet eine feste Farbpalette – und viele der verfügbaren Farben sind für Spielelemente reserviert (allein neun Blautöne). Für die Erdfotos blieb wenig Spielraum. Waage entwickelte einen Algorithmus, der für jeden Pixel den nächsten verfügbaren Farbwert aus der Doom-Palette findet und das Bild entsprechend anpasst.

---

## Der unbeabsichtigte Nebeneffekt

Um ein Foto der Erde zu machen, musste der Satellit seine Kamera **zur Erde ausrichten**. Das klingt trivial – ist es aber nicht:

Die geänderte Ausrichtung erhöhte den **atmosphärischen Widerstand** (auch in 500 km Höhe gibt es noch Spurengas). Mehr Widerstand = verlangsamte Umlaufbahn = niedrigerer Orbit = früherer Wiedereintritt.

**Das Spielen von Doom hat den Satelliten buchstäblich früher zum Absturz gebracht.**

The Register schreibt dazu trocken:

> *„Playing Doom accelerated the doom of the satellite."*

---

## Der historische Kontext

Doom läuft inzwischen auf Schwangerschaftstests, Druckern, Darmbakterien (eigener Artikel), Rasenmähern und Taschenrechnern. Waage hat die Kette logisch weitergedacht: Wenn es auf allem läuft – warum nicht im Weltraum?

Die Antwort: Es geht. Headless, auf Ubuntu Arm, in einem Container, mit Erdfotos als Himmel. Und der Quellcode ist auf GitHub verfügbar.

---

## Originalquelle

➡️ **[The Register – A satellite runs Doom from orbit, using Ubuntu on Arm](https://www.theregister.com/2025/10/28/doom_running_in_space/)** (Liam Proven, 28. Oktober 2025)

---

*Erstellt von Andy – 10. März 2026*
