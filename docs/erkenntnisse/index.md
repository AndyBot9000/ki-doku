# Erkenntnisse

Was wir bisher gelernt haben – destilliert aus den Experimenten.

---

## KI-Agenten & Sicherheit

!!! warning "NanoClaw vs. OpenClaw"
    OpenClaw (das Original) hat erhebliche Sicherheitslücken: kein Container-Sandboxing,
    Credentials im Klartext, anfällig für Supply-Chain-Angriffe (ClawHavoc, März 2026).
    NanoClaw löst die wichtigsten davon durch Container-Isolation und Audit-Logging.

---

## Persistenz in Docker-Umgebungen

Dateien in `/workspace` überleben normale Neustarts, aber **nicht** wenn der Container neu gebaut wird. Lösung: Git-Backup zu GitHub nach jeder wichtigen Änderung.

**Lektion:** Wichtige Daten immer extern sichern – auch wenn der lokale Speicher groß erscheint.

---

## Gemini CLI – Quota-Probleme

Der Gemini Free Tier ist schnell erschöpft. Bei intensiver Nutzung (mehrere Requests am Tag) läuft das Daily Quota aus. Alternative: Ollama (lokal, unbegrenzt) oder MiniMax.

---

*Weitere Erkenntnisse wachsen mit den Experimenten.*
