---
title: "ORCHA: Production Deployment"
date: 2026-03-13
description: "Wie der ORCHA Stack (ORCHA + PicoAPI + Telegram-Bot) auf einer Ubuntu VM produktiv betrieben wird — Startup, Überwachung, automatischer Neustart mit systemd."
tags:
  - orcha
  - deployment
  - systemd
  - linux
  - production
---

# ORCHA: Production Deployment

!!! info "Voraussetzung"
    Diese Seite setzt voraus, dass du den [ORCHA + PicoAPI Stack](orcha-picoapi-stack.md) kennst.
    Hier geht es ausschließlich um den Produktivbetrieb.

---

## Übersicht

Der ORCHA Stack besteht aus drei Prozessen:

| Dienst | Port | Funktion |
|---|---|---|
| `orcha.py` | 8086 | AI Routing Engine |
| `picoapi.py` | 8090 | Stateful REST API + Web-UI |
| `picoclaw.py` | — | Telegram Bot |

Alle drei laufen als separate Prozesse und kommunizieren über HTTP/localhost.

---

## Schnellstart: start-all.sh

Das einfachste Deployment-Tool ist `scripts/start-all.sh`:

```bash
# Alle drei Dienste starten
bash scripts/start-all.sh

# Status prüfen
bash scripts/start-all.sh --status

# Alle stoppen (Graceful SIGTERM)
bash scripts/start-all.sh --stop

# Neu starten (stop + start)
bash scripts/start-all.sh --restart
```

### Was das Script macht

1. Lädt API-Keys aus mehreren `.env`-Dateien (in Reihenfolge)
2. Startet ORCHA auf Port 8086
3. Wartet 2 Sekunden und prüft ob ORCHA läuft
4. Startet PicoAPI auf Port 8090 (zeigt auf ORCHA)
5. Startet picoclaw (zeigt auf PicoAPI)
6. Schreibt PIDs in `pids/orcha.pid`, `pids/picoapi.pid`, `pids/picoclaw.pid`

### Logs

Alle Prozesse schreiben in `logs/`:

```
logs/
├── orcha.log          # stdout/stderr von orcha.py
├── orcha.jsonl        # strukturiertes JSON-Log
├── picoapi.log        # stdout/stderr von picoapi.py
├── picoapi.jsonl      # strukturiertes JSON-Log
├── picoclaw.log       # stdout/stderr von picoclaw.py
└── picoclaw.jsonl     # strukturiertes JSON-Log
```

Live-Logs anschauen:

```bash
tail -f logs/picoapi.jsonl | python3 -c "
import sys, json
for line in sys.stdin:
    d = json.loads(line)
    print(d['ts'][:19], d['event'], d.get('session',''), d.get('backend',''))
"
```

---

## Automatischer Neustart: systemd

Für echten Produktivbetrieb empfehlen sich systemd User-Services. Diese starten automatisch mit der VM und starten sich bei Absturz selbst neu.

### Installation

```bash
# Verzeichnis anlegen
mkdir -p ~/.config/systemd/user

# Service-Dateien kopieren
cp systemd/orcha.service    ~/.config/systemd/user/
cp systemd/picoapi.service  ~/.config/systemd/user/
cp systemd/picoclaw.service ~/.config/systemd/user/

# Daemon neu laden
systemctl --user daemon-reload

# Alle aktivieren und starten
systemctl --user enable --now orcha.service
systemctl --user enable --now picoapi.service
systemctl --user enable --now picoclaw.service
```

### Verwaltung

```bash
# Status aller drei
systemctl --user status orcha picoapi picoclaw

# Neu starten
systemctl --user restart orcha

# Logs (live)
journalctl --user -u orcha -f
journalctl --user -u picoapi -f

# Alle drei gleichzeitig deaktivieren
systemctl --user disable --now orcha picoapi picoclaw
```

### Abhängigkeiten

Die Services sind voneinander abhängig konfiguriert:

```
orcha.service
    └── picoapi.service
            └── picoclaw.service
```

Startet `orcha.service` nicht, warten picoapi und picoclaw.

### Restart-Verhalten

```ini
Restart=on-failure   # Nur bei Absturz, nicht bei normalem Stop
RestartSec=5         # 5 Sekunden warten vor Neustart
StartLimitBurst=5    # Max 5 Neustarts...
StartLimitIntervalSec=60  # ...innerhalb von 60 Sekunden
```

---

## Graceful Shutdown

Alle drei Prozesse reagieren auf `SIGTERM` mit einem sauberen Shutdown:

- Laufende HTTP-Requests werden abgeschlossen
- PID-Datei wird gelöscht
- Log-Eintrag `server_stop` wird geschrieben

```bash
# Manuell einen Prozess sauber stoppen
kill -TERM $(cat pids/orcha.pid)

# Oder via systemd
systemctl --user stop orcha
```

Das `start-all.sh --stop` sendet SIGTERM und wartet 2 Sekunden. Falls der Prozess dann noch läuft, kommt SIGKILL als Fallback.

---

## Konfiguration via Umgebungsvariablen

Alle Parameter sind via Environment konfigurierbar — keine Konfigurationsdatei nötig:

### ORCHA (orcha.py)

| Variable | Default | Bedeutung |
|---|---|---|
| `ORCHA_PORT` | `8086` | HTTP-Port |
| `OLLAMA_HOST` | `192.168.10.11` | Ollama-Server IP |
| `OLLAMA_PORT` | `11434` | Ollama-Port |
| `OLLAMA_MODEL` | `llama3.2` | Lokales Modell |
| `ORCHA_LOG` | `logs/orcha.jsonl` | Pfad zum JSON-Log |
| `GEMINI_API_KEY` | — | Gemini API Key |
| `ANTHROPIC_API_KEY` | — | Claude API Key |
| `OPENROUTER_API_KEY` | — | OpenRouter API Key |

### PicoAPI (picoapi.py)

| Variable | Default | Bedeutung |
|---|---|---|
| `PICO_PORT` | `8090` | HTTP-Port |
| `ORCHA_URL` | `http://localhost:8086` | ORCHA Upstream |
| `PICO_MAX_HISTORY` | `20` | Turns pro Session |
| `PICO_MAX_TOKENS` | `4096` | Max Output-Tokens |
| `PICO_SYSTEM` | *(PicoClaw-Prompt)* | System-Prompt |
| `PICO_LOG` | `logs/picoapi.jsonl` | Pfad zum JSON-Log |
| `PICO_DB` | `data/picoclaw.db` | SQLite-Datenbank |

### picoclaw (picoclaw.py)

| Variable | Default | Bedeutung |
|---|---|---|
| `TELEGRAM_TOKEN` | — | Pflicht! |
| `ORCHA_URL` | `http://localhost:8090` | PicoAPI URL |
| `PICOCLAW_SYSTEM` | *(Standard-Prompt)* | System-Prompt |
| `PICOCLAW_LOG` | `logs/picoclaw.jsonl` | Pfad zum JSON-Log |

---

## .env-Datei

Das Start-Script lädt automatisch aus drei Quellen (letzte gewinnt):

```bash
/workspace/project/.env        # Host-Ebene (persistent)
scripts/.env                   # Lokale Overrides
/workspace/project/data/env/env  # NanoClaw Session-Env
```

Minimale `.env` für lokales Testen:

```bash
GEMINI_API_KEY=...
ANTHROPIC_API_KEY=...
OPENROUTER_API_KEY=...
TELEGRAM_TOKEN=...
```

---

## Log-Rotation

Die Logs wachsen kontinuierlich. Das `rotate-logs.sh` Script rotiert alle `.log` und `.jsonl` Dateien ab einer konfigurierbaren Größe:

```bash
# Einmalige Rotation (ab 50 MB, Standard)
bash scripts/rotate-logs.sh

# Mit anderen Limits
MAX_SIZE_MB=10 KEEP_ROTATIONS=3 bash scripts/rotate-logs.sh

# Nur anzeigen ohne zu rotieren
bash scripts/rotate-logs.sh --dry-run
```

### Automatisch via Cron

```bash
# Crontab editieren
crontab -e

# Täglich 03:00 Uhr rotieren
0 3 * * * bash /workspace/group/scripts/rotate-logs.sh >> /workspace/group/logs/rotate.log 2>&1
```

---

## Health-Checks

### Schneller Status-Überblick

```bash
bash scripts/start-all.sh --status
```

```
📊 ORCHA Stack Status
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ✅ orcha    (PID: 308)
   ✅ picoapi  (PID: 316)
   ✅ picoclaw (PID: 322)

   🌐 PicoAPI Health:
     ❌ ollama       —
     ✅ openrouter   3681ms
     ✅ gemini       199ms
     ✅ claude       235ms
```

### curl / HTTP

```bash
# ORCHA direkt
curl http://localhost:8086/v1/health

# Via PicoAPI (empfohlen)
curl http://localhost:8090/v1/health | python3 -m json.tool

# Alle Sessions
curl http://localhost:8090/v1/sessions
```

### Web-UI

Die PicoAPI stellt unter `http://localhost:8090/` eine eingebettete Chat-UI bereit:

- **⚡ Status** — zeigt alle Backends mit Latenz
- **🔄 Refresh** — löscht Session + Memory
- **Session-ID** — wechselbare Session-Identität

---

## Request-Tracing

Jeder ORCHA-Request bekommt eine eindeutige `req`-ID im Log:

```json
{"ts":"2026-03-13T11:38:22Z","event":"routing","req":"req-a3f2b1c0","complexity":0.165,"selected":"openrouter"}
{"ts":"2026-03-13T11:38:23Z","event":"backend_ok","req":"req-a3f2b1c0","backend":"openrouter","model":"google/gemma-3-27b-it:free","latency_ms":1243}
```

Mit `grep req-a3f2b1c0 logs/orcha.jsonl` lässt sich ein Request komplett nachverfolgen.

---

## Typische Probleme

### Port bereits belegt

```bash
# Welcher Prozess hört auf Port 8086?
lsof -i :8086
# oder
ss -tlnp | grep 8086
```

### Prozess hängt / reagiert nicht

```bash
# SIGKILL als letztes Mittel
kill -KILL $(cat pids/orcha.pid)
rm pids/orcha.pid
```

### Alle Backend-Fehler (503 von PicoAPI)

```bash
# Letzte Fehler im ORCHA-Log
grep '"event":"backend_error"' logs/orcha.jsonl | tail -10 | python3 -m json.tool
```

### SQLite gesperrt

```bash
# Alle Python-Prozesse prüfen die picoclaw.db offen haben
lsof data/picoclaw.db
```

---

## Produktions-Checkliste

- [ ] API-Keys in `/workspace/project/.env` hinterlegt
- [ ] `bash start-all.sh` läuft ohne Fehler
- [ ] `bash start-all.sh --status` zeigt alle 3 als ✅
- [ ] Telegram-Bot antwortet auf `/start`
- [ ] systemd Services installiert und aktiviert
- [ ] Log-Rotation in Crontab eingetragen
- [ ] Firewall: Ports 8086 und 8090 **nicht** von außen erreichbar (nur localhost)
