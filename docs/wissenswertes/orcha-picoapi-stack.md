---
title: "ORCHA + PicoAPI: Selbst gebauter KI-Router mit Memory"
date: 2026-03-12
description: "Wie Andy einen OpenAI-kompatiblen Multi-Backend-Router mit SQLite-Memory, Telegram-Bot und vollständigem Logging selbst gebaut hat — ohne externe Abhängigkeiten."
tags:
  - orcha
  - ki-infrastruktur
  - openai-api
  - sqlite
  - telegram
  - python
---

# ORCHA + PicoAPI: Selbst gebauter KI-Router mit Memory

!!! info "Status: Produktiv"
    Das System läuft produktiv auf dieser VM. Der Telegram-Bot **@BetterBouleBot** ist live und nutzt diesen Stack als Backend.

---

## Warum überhaupt?

Die Ausgangssituation: Zu viele KI-Modelle, zu wenige Ressourcen, zu hohe Kosten. Claude ist intelligent aber teuer. Gemini ist schnell und günstig. OpenRouter bietet kostenlose Modelle — wenn man sie findet und richtig anfragt.

Das Problem: Jede App, die KI nutzen will, muss selbst entscheiden, welches Modell sie nimmt. Das ist ineffizient und fehleranfällig.

Die Lösung: **Ein zentraler Router**, der diese Entscheidung automatisch trifft — basierend auf der Komplexität der Anfrage und der aktuellen Verfügbarkeit der Backends.

---

## Architektur

```
Telegram
    │
    ▼
picoclaw.py          Telegram-Bot, Polling, Footer mit Model/Tokens
    │ POST /v1/chat/completions
    │ user: "{chat_id}"
    ▼
picoapi.py  :8090    Stateful REST API — History, Memory, Logging
    │ POST /v1/chat/completions
    ▼
orcha.py    :8086    Routing-Engine — Komplexität + Ressourcen
    │
    ├── OpenRouter   FREE-Modelle (Gemma, Llama, Mistral...)
    ├── Gemini       Mittlere Komplexität
    └── Claude       Hohe Komplexität
```

Vier Python-Dateien, pure stdlib, **zero external dependencies**. Kein pip, kein venv, kein Framework.

---

## Modul 1: ORCHA — der Router

`orcha.py` ist ein HTTP-Server der die OpenAI-API nachbaut. Jede App, die OpenAI kennt, kann stattdessen ORCHA ansprechen.

### Routing-Logik

Jede Anfrage bekommt einen **Komplexitätswert** zwischen 0.0 und 1.0:

```python
# Heuristik — kein LLM-Call, reine Textanalyse
def measure_complexity(messages) -> float:
    score = 0.0
    text = " ".join(m.get("content","") for m in messages).lower()

    # Länge (0–0.3)
    score += min(len(text) / 3000, 0.3)

    # Keyword-Matching (Wortgrenzen)
    COMPLEX_KEYWORDS = ["architektur", "analysiere", "debugge", "warum",
                        "kritisch", "cqrs", "optimiere", "erkläre detailliert"]
    SIMPLE_KEYWORDS  = ["danke", "ok", "ja", "nein", "hallo", "tschüss"]

    for kw in COMPLEX_KEYWORDS:
        if re.search(r'\b' + re.escape(kw) + r'\b', text):
            score += 0.12

    for kw in SIMPLE_KEYWORDS:
        if re.search(r'\b' + re.escape(kw) + r'\b', text):
            score = max(0, score - 0.08)

    return min(score, 1.0)
```

Das Backend wird dann per **Scoring-Formel** gewählt:

| Backend | Optimum | Formel |
|---|---|---|
| OpenRouter | 0.0 – 0.35 | `max(0.2, 1.0 - complexity × 1.2)` |
| Gemini | 0.35 – 0.55 | `1.0 - abs(complexity - 0.4) × 3.5` |
| Claude | 0.55+ | `complexity^0.7` |

```
score = intelligence_match × resource_factor × INTELLIGENCE_weight
```

### Health Monitoring

Ein Background-Thread prüft alle 60 Sekunden die Verfügbarkeit aller Backends. Bei Quota-Fehlern wird das Backend 5 Minuten gesperrt und es gibt automatischen Fallback:

```
OpenRouter → Gemini → Claude (oder umgekehrt)
```

### API-Endpunkte

```
POST /v1/chat/completions   OpenAI-kompatibel
GET  /v1/models             Verfügbare Backends
GET  /v1/health             Live-Status aller Backends
```

---

## Modul 2: PicoAPI — die Stateful-Schicht

`picoapi.py` sitzt vor ORCHA und fügt hinzu, was ORCHA nicht hat: **Gedächtnis**.

### Stateful vs. Stateless

```python
# Stateful: "user" Feld = Session-Key
{
  "model": "auto",
  "user": "alice",          # ← Session-ID
  "messages": [{"role": "user", "content": "Hallo!"}]
}

# Stateless: kein "user" Feld → jede Anfrage unabhängig
{
  "model": "auto",
  "messages": [{"role": "user", "content": "Hallo!"}]
}
```

Im Stateful-Modus baut PicoAPI die vollständige Message-History automatisch auf:

```
[System-Prompt + Memory-Block]
[Turn 1: User]
[Turn 1: Assistant]
[Turn 2: User]
...
[Aktuelle Anfrage]
```

### Memory-Injektion

Wenn es Memory-Einträge für eine Session gibt, werden sie automatisch in den System-Prompt injiziert:

```
Du bist PicoClaw, ein persönlicher KI-Assistent...

[Memory]
name: Klaus
beruf: Softwareentwickler, arbeitet mit Python
projekt: n8n Installation auf Ubuntu VM
```

Das Modell "weiß" damit immer, mit wem es redet — ohne dass der Nutzer es jedes Mal neu erklären muss.

### API-Endpunkte

```
POST   /v1/chat/completions        Chat (stateful oder stateless)
GET    /v1/models                  Backends (von ORCHA)
GET    /v1/health                  Health (von ORCHA)

GET    /v1/sessions                Alle aktiven Sessions
GET    /v1/sessions/{id}           Session-Info + Memory
DELETE /v1/sessions/{id}           History + Memory löschen

GET    /v1/sessions/{id}/memory    Memory-Einträge lesen
POST   /v1/sessions/{id}/memory    Memory-Eintrag setzen: {"key":"...", "value":"..."}
DELETE /v1/sessions/{id}/memory/{key}  Eintrag löschen
```

---

## Modul 3: Store — SQLite-Persistenz

`store.py` verwaltet zwei Stores auf einer SQLite-Datenbank (`data/picoclaw.db`):

### Schema

```sql
-- Conversation History
CREATE TABLE history (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT    NOT NULL,
    role       TEXT    NOT NULL,   -- 'user' | 'assistant'
    content    TEXT    NOT NULL,
    ts         TEXT    NOT NULL,
    model      TEXT,               -- z.B. "google/gemma-3-27b-it:free"
    backend    TEXT               -- z.B. "openrouter"
);

-- Session Metadata
CREATE TABLE sessions (
    session_id    TEXT PRIMARY KEY,
    created       TEXT NOT NULL,
    last_activity TEXT,
    turns         INTEGER DEFAULT 0
);

-- Langzeit-Memory
CREATE TABLE memory (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL,
    key        TEXT NOT NULL,
    value      TEXT NOT NULL,
    ts         TEXT NOT NULL,
    source     TEXT DEFAULT 'user'  -- 'user' | 'auto' | 'system'
);
```

WAL-Mode für concurrent reads, thread-lokale Connections, Write-Lock für alle Schreiboperationen.

### CLI

Direkte DB-Abfragen ohne SQL:

```bash
python3 store.py sessions
python3 store.py history 635432624
python3 store.py memory  635432624
python3 store.py set-memory 635432624 name "Klaus"
python3 store.py del-memory 635432624 name
python3 store.py del-session 635432624
```

---

## Modul 4: PicoClaw — der Telegram-Bot

`picoclaw.py` ist das Frontend: ein Telegram-Polling-Bot ohne Webhook, ohne Framework.

Jede Antwort zeigt automatisch welches Backend geantwortet hat:

```
Was kostet ein Serverless-Setup auf AWS Lambda?

[Ausführliche Antwort...]

▸ gemini · gemini-2.0-flash · ↑847 ↓312 tok
```

**Befehle:**

| Befehl | Funktion |
|---|---|
| `/start` | Session + Memory löschen, Neustart |
| `/reset` | Gespräch zurücksetzen (History + Memory) |
| `/status` | Backend-Health anzeigen |
| `/history` | Anzahl gespeicherter Turns |

Der Bot delegiert **alles** an PicoAPI — er verwaltet selbst keinen State. Session-Key = Telegram `chat_id`.

---

## Logging

Alle drei Dienste schreiben **JSON Lines** (`.jsonl`) — ein strukturierter JSON-Record pro Zeile, append-only, direkt für AI-Analyse geeignet.

### Beispiel: `logs/orcha.jsonl`

```json
{"ts":"2026-03-12T14:38:22Z", "event":"routing", "complexity":0.165, "selected":"openrouter", "available":["openrouter","gemini","claude"], "scores":{"openrouter":0.441,"claude":0.283,"gemini":0.132}}
{"ts":"2026-03-12T14:38:23Z", "event":"backend_ok", "backend":"openrouter", "model":"google/gemma-3-27b-it:free", "latency_ms":1243, "input_tokens":312, "output_tokens":87}
{"ts":"2026-03-12T14:41:05Z", "event":"backend_error", "backend":"openrouter", "kind":"quota", "detail":"Rate limit exceeded"}
{"ts":"2026-03-12T14:41:05Z", "event":"fallback_ok", "original_backend":"openrouter", "fallback":"gemini", "model":"gemini-2.0-flash", "latency_ms":890}
```

### Beispiel: `logs/picoapi.jsonl`

```json
{"ts":"2026-03-12T14:38:20Z", "event":"chat_request", "session":"635432624", "model":"auto", "history_turns":3, "user_chars":84, "stateful":true}
{"ts":"2026-03-12T14:38:24Z", "event":"chat_response", "session":"635432624", "backend":"openrouter", "model":"google/gemma-3-27b-it:free", "input_tokens":0, "output_tokens":142, "latency_ms":1251}
{"ts":"2026-03-12T14:39:10Z", "event":"memory_set", "session":"635432624", "key":"name"}
{"ts":"2026-03-12T14:55:00Z", "event":"session_reset", "session":"635432624", "history_deleted":true, "memory_cleared":2}
```

Phase 2 dieses Systems: ein Log-Analyzer, der diese Daten auswertet und Empfehlungen generiert.

---

## Mit jeder OpenAI-kompatiblen App nutzbar

Da PicoAPI den OpenAI-Standard implementiert, funktioniert jede kompatible App direkt:

=== "Python (openai)"

    ```python
    from openai import OpenAI

    client = OpenAI(
        base_url="http://localhost:8090/v1",
        api_key="picoclaw"  # wird ignoriert
    )

    response = client.chat.completions.create(
        model="auto",
        messages=[{"role": "user", "content": "Erkläre CQRS"}],
        extra_body={"user": "meine-session"}  # stateful
    )
    print(response.choices[0].message.content)
    ```

=== "curl"

    ```bash
    # Stateful: History bleibt erhalten
    curl -X POST http://localhost:8090/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d '{
        "model": "auto",
        "user": "alice",
        "messages": [{"role": "user", "content": "Mein Name ist Klaus."}]
      }'

    # Memory setzen
    curl -X POST http://localhost:8090/v1/sessions/alice/memory \
      -H "Content-Type: application/json" \
      -d '{"key": "sprache", "value": "Deutsch"}'
    ```

=== "LangChain"

    ```python
    from langchain_openai import ChatOpenAI

    llm = ChatOpenAI(
        base_url="http://localhost:8090/v1",
        api_key="picoclaw",
        model="auto"
    )
    ```

---

## Technische Details

| Eigenschaft | Wert |
|---|---|
| Sprache | Python 3.11 |
| Dependencies | **keine** (pure stdlib) |
| Datenbank | SQLite 3 (WAL-Mode) |
| Protokoll | HTTP/1.1, JSON |
| Logging | JSON Lines (append-only) |
| Threading | Thread-lokale DB-Connections, Write-Locks |
| Fallback | Automatisch bei Quota/Fehler |

### Warum keine Dependencies?

Die VM hat kein `pip`. Die Entscheidung, alles mit stdlib zu bauen, war eine Einschränkung die sich als Vorteil herausgestellt hat: Das System hat **keine Versionskonflikte**, keinen Installations-Overhead und läuft auf jedem Python 3.8+.

---

## Quellcode

Der vollständige Quellcode ist im privaten `andy-workspace` Repository. Die ORCHA-Grundstruktur (ohne PicoAPI/Store) ist auch als **Open-Source-Paket** verfügbar:

[github.com/AndyBot9000/orcha](https://github.com/AndyBot9000/orcha){ .md-button }

---

## Nächste Schritte

- [ ] **Phase 2: Log-Analyzer** — AI wertet `.jsonl`-Logs aus, identifiziert Muster, schlägt Routing-Anpassungen vor
- [ ] **Streaming** — Server-Sent Events für Live-Antworten
- [ ] **Tool Calling** — Passthrough für Function-Calling (für Agent-Frameworks)
- [ ] **Ollama** — Lokal laufende Modelle einbinden (derzeit nur via Loopback erreichbar)
- [ ] **Auto-Memory** — KI extrahiert automatisch relevante Fakten aus Gesprächen
