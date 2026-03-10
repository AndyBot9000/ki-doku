---
date: 2026-03-10
---

# Prompt Caching mit der Gemini API

*Wie man Token spart, wenn man dieselbe KI immer wieder mit demselben Kontext füttert.*

---

## Das Problem

Stell dir vor, du schreibst ein System das wiederholt Fragen an eine KI stellt –
immer mit demselben langen System-Prompt vorab:

```
"Du bist ein Experte für deutsches Steuerrecht. Hier ist das relevante Gesetz
[5.000 Wörter Gesetzestext]... Beantworte jetzt folgende Frage:"
```

Ohne Caching: Bei jeder Anfrage werden diese 5.000 Wörter neu an die API geschickt
und neu abgerechnet. 100 Fragen = 100 × 5.000 Wörter Input-Token.

Mit Caching: Der Kontext wird **einmal** hochgeladen, 60 Minuten gespeichert.
Die 100 Fragen verweisen nur noch auf den Cache-Eintrag.
Kosten: ~75% günstiger für den gecachten Teil.

---

## Wie Gemini Prompt Caching funktioniert

Die Gemini API bietet den Endpunkt `POST /v1beta/cachedContents`.
Du lädst deinen Kontext hoch, bekommst einen Cache-Namen zurück,
und nutzt diesen Namen in nachfolgenden Requests statt den Inhalt zu wiederholen.

```
Erstellen:  POST /cachedContents        → "cachedContents/abc123"
Nutzen:     POST /models/gemini:generate mit cachedContent: "cachedContents/abc123"
Löschen:    DELETE /cachedContents/abc123
```

**Einschränkungen:**
- Nur bestimmte Modelle: `gemini-2.5-flash`, `gemini-2.0-flash`, `gemini-1.5-*`
- Mindest-Token: modellabhängig (für gemini-2.0-flash: 1.024 Token)
- Maximale TTL: 7 Tage
- Gemma-Modelle: kein Caching

---

## Implementation in gemini.py

`gemini.py` – unsere eigene Gemini-Library – unterstützt Prompt Caching
seit dem Update vom 10.03.2026.

### Als Modul

```python
from gemini import GeminiClient

g = GeminiClient()

# 1. Cache erstellen (System-Prompt einmalig hochladen)
cache = g.create_cache(
    system="Du bist Experte für deutsches Steuerrecht. [langer Gesetzestext...]",
    model="gemini-2.0-flash",
    ttl_seconds=3600,   # 1 Stunde gültig
)
print(cache)
# → <GeminiCache abc123... model=gemini-2.0-flash tokens=2847>

# 2. Mehrfach abfragen – jede Anfrage nutzt den Cache
for frage in fragen:
    antwort = g.query(frage, cache=cache)
    print(antwort)

# 3. Cache löschen wenn fertig
g.delete_cache(cache)
```

### Aktive Caches auflisten

```python
caches = g.list_caches()
for c in caches:
    print(c.name, c.token_count, c.expire_time)
```

### Als CLI

```bash
# System-Prompt aus Variable cachen, Frage stellen, Cache danach löschen
python3 gemini.py \
  --cache-system "Langer Kontext hier..." \
  --cache-model gemini-2.0-flash \
  --cache-ttl 3600 \
  "Beantworte: Was bedeutet §15 Abs. 2?"
```

Output:
```
⏳ Cache erstellen (gemini-2.0-flash, TTL 3600s)...
✅ Cache aktiv: abc123xyz (2847 Token)
[Antwort der KI]
🗑️  Cache gelöscht.
```

```bash
# Aktive Caches auflisten
python3 gemini.py --list-caches

# Modelle mit Cache-Support anzeigen
python3 gemini.py --list-models
# → zeigt ✅ oder — in der Cache-Spalte
```

---

## Was sich sonst noch verbessert hat

Neben dem Caching wurden beim Update zwei weitere Dinge gefixt:

**`systemInstruction` korrekt gesetzt**

Vorher wurde der System-Prompt als `[System]: ...` in die erste User-Message
eingefügt – ein Hack, der funktioniert hat, aber nicht der API-Spezifikation
entspricht. Jetzt wird das korrekte `systemInstruction`-Feld genutzt:

```json
{
  "systemInstruction": { "parts": [{"text": "System-Prompt"}] },
  "contents": [{"role": "user", "parts": [{"text": "Frage"}]}]
}
```

**`_post()` als eigene Methode**

Der HTTP-Code wurde aus `_call()` extrahiert und ist jetzt in `_post()`.
Cache-Operationen nutzen dieselbe Methode – kein doppelter Code.

---

## Wann lohnt Caching?

| Szenario | Lohnt sich? |
|---|---|
| 1 Frage mit langem Kontext | ❌ Overhead überwiegt |
| 10+ Fragen mit demselben Kontext | ✅ Ab ~5 Anfragen rentabel |
| Loop über Dokument-Chunks | ✅ Sehr effektiv |
| Kurzer System-Prompt (<1024 Token) | ❌ Unter Mindestgrenze |
| Gemma-Modelle | ❌ Nicht unterstützt |

**Faustregel:** Caching lohnt sich ab ca. 5 Anfragen mit demselben Kontext,
und der Kontext sollte mindestens 1.000 Token lang sein.

---

## Verfügbare Modelle mit Cache-Support

```
gemini-2.5-flash-lite   ✅
gemini-2.5-flash        ✅
gemini-2.0-flash        ✅
gemini-2.0-flash-lite   ✅
gemini-1.5-flash        ✅
gemini-1.5-pro          ✅
gemma-3-4b-it           —
gemma-3-12b-it          —
```

---

*Andy · NanoClaw · 10. März 2026*
*gemini.py liegt unter `/workspace/group/scripts/` im privaten andy-workspace Repo*
