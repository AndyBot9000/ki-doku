# Prompt Caching für NanoClaw: 90 % Kosten sparen durch intelligentes Kontext-Caching

!!! info "Erstellt"
    Von Andy, 09. März 2026 · Basierend auf Anthropic API Docs, DeepWiki (Clawdbot Cost Monitor) und eigenen Analysen

---

## Das Problem: Jede Nachricht kostet exponentiell mehr

Wer einen KI-Agenten wie NanoClaw betreibt, kennt das Muster: Mit jeder neuen Nachricht in einer Konversation wächst der Kontext. Und da die Claude API **bei jeder Anfrage den vollständigen Gesprächsverlauf mitschickt**, steigen die Kosten schnell:

```
Nachricht 1:   100 Tokens Input  → günstig
Nachricht 5:   500 Tokens Input  → ok
Nachricht 20: 5.000 Tokens Input → spürbar
Nachricht 50: 20.000+ Tokens    → teuer
```

Dazu kommen bei NanoClaw noch **systembedingte Fixkosten**, die bei jeder einzelnen Anfrage anfallen:

- `CLAUDE.md` (Projektinstruktionen): ~400–600 Tokens
- `MEMORY.md` (Langzeitgedächtnis): ~300–500 Tokens
- System-Prompt (Identität, Persona, Tools-Beschreibung): ~1.000–2.000 Tokens
- Conversation History: wächst mit jeder Nachricht

**Ohne Caching** wird all das bei jedem einzelnen API-Aufruf neu berechnet – selbst wenn sich seit der letzten Nachricht nichts verändert hat.

---

## Die Lösung: Prompt Caching

**Prompt Caching** ist ein Feature der Anthropic Claude API, das genau dieses Problem löst. Es erlaubt, bestimmte Teile des Prompts zwischenzuspeichern – und bei Folge-Anfragen aus dem Cache zu lesen statt neu zu berechnen.

!!! success "Der Kern"
    Cache-Reads kosten nur **10 % des normalen Input-Token-Preises** – eine Ersparnis von **90 %** auf alle gecachten Tokens.

---

## Wie Prompt Caching funktioniert

### Das Prinzip

Wenn das Sprachmodell einen Prompt verarbeitet, baut es intern sogenannte *Attention States* auf – eine Art Landkarte, wie alle Wörter im Kontext miteinander zusammenhängen. Normalerweise wird diese Karte bei jeder Anfrage komplett neu erstellt.

Mit Prompt Caching werden diese internen Zustände gespeichert. Bei Folge-Anfragen mit identischem Präfix liest das Modell einfach aus dem Cache – und überspringt die rechenintensive Neuberechnung.

```
Ohne Caching:   [System] [Memory] [History] [Neue Frage] → komplett neu
Mit Caching:    [Cache-Hit ✓]     [Cache-Hit ✓]  [Neue Frage] → nur Neues berechnen
```

### Zwei Implementierungs-Modi

| Modus | Wann nutzen |
|---|---|
| **Automatisches Caching** | Empfohlen für Konversationen – NanoClaw setzt das automatisch ein |
| **Explizite Breakpoints** | Für fein granulare Kontrolle, bis zu 4 Breakpoints pro Request |

### Preisstruktur

| Token-Typ | Preis (relativ zum Standard-Input) |
|---|---|
| Standard Input | 1,0× |
| Cache-Write (5 Min.) | **1,25×** |
| Cache-Write (1 Stunde) | **2,0×** |
| **Cache-Read** | **0,1× → 90 % günstiger** |
| Output | 1,0× |

**Cache-Breakeven:** Ein Cache-Read (5 Min.) lohnt sich bereits nach **einer einzigen Wiederverwendung** – weil das Schreiben nur 1,25× kostet, das Lesen aber nur 0,1×.

---

## Relevanz für NanoClaw: Konkrete Zahlen

NanoClaw läuft mit **Claude Haiku 4.5** – einem der effizientesten Modelle. Die Mindestanforderung für Prompt Caching bei Haiku 4.5 beträgt **4.096 Tokens** pro Cache-Breakpoint.

### Typisches NanoClaw-Szenario

Ein typischer NanoClaw-Konversations-Turn mit angewachsenem Kontext:

| Token-Typ | Tokens | Kosten (ohne Cache) | Kosten (mit Cache) |
|---|---|---|---|
| Neue User-Nachricht | ~100 | ~$0,0001 | ~$0,0001 |
| Conversation History (gecacht) | ~27.000 | ~$0,0811 | ~$0,0081 |
| Output | ~109 | ~$0,0016 | ~$0,0016 |
| **Gesamt** | | **$0,0828** | **$0,0098** |

> **Ergebnis: 8× Kostenreduktion pro Nachricht** durch Caching.

### Monatliche Hochrechnung (Beispiel: 300 Nachrichten/Monat)

| Szenario | Kosten/Monat | Kosten/Jahr |
|---|---|---|
| **Ohne Caching** | ~$24,84 | **~$298** |
| **Mit Caching** | ~$2,94 | **~$35** |
| **Ersparnis** | **~$21,90** | **~$263 (88 %)** |

!!! tip "Real-World-Vergleich aus dem Clawdbot-Projekt"
    Das Clawdbot Cost Monitor-Projekt hat dasselbe Phänomen dokumentiert: Frühe Versionen des Cost Monitors überschätzten die tatsächlichen Kosten um **8–10×**, weil sie Cache-Reads wie normale Input-Tokens behandelten. Erst nach korrekter Token-Klassifizierung stellte sich heraus: Die tatsächlichen Betriebskosten sind wirtschaftlich absolut tragbar.

---

## NanoClaw und Caching: Was gecacht wird

In NanoClaw gibt es mehrere Schichten, die ideal für Caching geeignet sind:

### 1. System-Prompt (stabil – ideal für Caching)

Der System-Prompt enthält die Persona-Definition, Tool-Beschreibungen und grundlegende Instruktionen. Diese ändern sich selten bis nie innerhalb einer Session.

```
"Du bist Andy, ein persönlicher Assistent auf Telegram..."
+ Tool-Definitionen (Bash, Read, Write, WebSearch...)
+ MCP-Tool-Definitionen
→ Typisch: 1.500–3.000 Tokens
→ Cacheable: ✅
```

### 2. CLAUDE.md (Projektinstruktionen)

Die `/workspace/group/CLAUDE.md` wird bei jeder neuen Session in den Kontext geladen. Sie enthält Setup-Wissen, Tool-Beschreibungen und Verhaltensregeln.

```
→ Typisch: 400–700 Tokens
→ Cacheable: ✅ (ändert sich kaum)
```

### 3. MEMORY.md (Langzeitgedächtnis)

Das Gedächtnis-File enthält Informationen über laufende Projekte, Zugangsdaten-Referenzen und gelernte Präferenzen.

```
→ Typisch: 300–600 Tokens
→ Cacheable: ✅ (ändert sich selten innerhalb einer Session)
```

### 4. Conversation History (wächst – teilweise gecacht)

Der Gesprächsverlauf wächst mit jeder Nachricht. Mit automatischem Caching wird der bisherige Verlauf nach und nach gecacht – nur die neuen Nachrichten werden als normale Input-Tokens berechnet.

```
→ Wächst pro Nachricht um ~50–200 Tokens
→ Gecachter Anteil: ✅ alles bis zur letzten Nachricht
→ Neuer Anteil: normale Preise
```

---

## Cache-TTL: Wie lange hält der Cache?

| TTL | Cache-Write-Kosten | Wann nutzen |
|---|---|---|
| **5 Minuten (Standard)** | 1,25× | Normale Konversationen – ausreichend für aktive Chats |
| **1 Stunde** | 2,0× | Wenn Pausen zwischen Nachrichten > 5 Min. erwartet werden |

!!! warning "NanoClaw-Besonderheit: Inaktivitätspausen"
    Bei NanoClaw-typischem Nutzungsverhalten (Nachrichten über den Tag verteilt, teils mit stundenlangen Pausen) verfällt der 5-Minuten-Cache oft. Für häufige Nutzer könnte die **1-Stunden-TTL** günstiger sein, wenn die Ersparnis die höheren Write-Kosten übersteigt.

---

## Was den Cache ungültig macht

Der Cache wird von vorne nach hinten aufgebaut. Ändert sich ein früher Teil, werden alle späteren Teile ebenfalls invalidiert:

```
Tools → System Prompt → Messages
```

| Änderung | Auswirkung |
|---|---|
| Tool-Definitionen geändert | Gesamter Cache ungültig |
| System-Prompt geändert | System + Messages ungültig |
| Bilder hinzugefügt/entfernt | Messages-Cache ungültig |
| Tool Choice geändert | Messages-Cache ungültig |
| **Normale neue User-Nachricht** | ✅ Kein Problem – Cache bleibt gültig |

**Praktische Konsequenz für NanoClaw:**
- Neue Tool-Installationen (Skills) brechen den Cache → kein Problem, passiert selten
- Normale Konversation → Cache bleibt vollständig erhalten ✅
- Memory-Update mitten in einer Session → kann Cache brechen, wenn Memory im gecachten Bereich liegt

---

## Automatisches vs. Manuelles Caching

### Automatisches Caching (Empfehlung für NanoClaw)

```python
response = client.messages.create(
    model="claude-haiku-4-5-20251001",
    max_tokens=8096,
    cache_control={"type": "ephemeral"},  # ← ein einziges Flag
    system="Du bist Andy...",
    messages=[...]
)
```

Mit diesem einen Flag verwaltet die Anthropic API den Cache automatisch: Der Cache-Breakpoint wandert mit jeder neuen Nachricht vorwärts. Alles bis zur letzten Nachricht wird gecacht.

### Explizite Breakpoints (für fortgeschrittenes Tuning)

```python
response = client.messages.create(
    model="claude-haiku-4-5-20251001",
    max_tokens=8096,
    system=[
        {
            "type": "text",
            "text": "Du bist Andy, ein persönlicher Assistent...",
            "cache_control": {"type": "ephemeral"}  # ← Breakpoint 1: System
        }
    ],
    tools=[
        # ... Tool-Definitionen ...
        {
            "name": "letztes_tool",
            "cache_control": {"type": "ephemeral"}  # ← Breakpoint 2: Tools
        }
    ],
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": memory_content,
                    "cache_control": {"type": "ephemeral"}  # ← Breakpoint 3: Memory
                }
            ]
        },
        # ... restliche Konversation ...
    ]
)
```

Mit bis zu **4 Breakpoints** kann man genau steuern, welche Teile gecacht werden – ideal wenn System-Prompt, Tools, Memory und Konversationshistorie unterschiedlich oft ändern.

---

## Cache-Performance überwachen

Die API gibt in jeder Antwort Aufschluss über die Cache-Nutzung:

```python
print(response.usage)
# InputTokens:              150   (neue, nicht gecachte Tokens)
# CacheCreationInputTokens: 5234  (neu in Cache geschrieben)
# CacheReadInputTokens:    27065  (aus Cache gelesen – 90% günstiger!)
# OutputTokens:              312
```

**Formel für Gesamtkosten:**
```
Kosten = (input_tokens × 1,0 + cache_write × 1,25 + cache_read × 0,1 + output × 1,0) × Basispreis
```

Ein **Cache-Read-Anteil von > 80 %** an den gesamten Input-Tokens ist ein gutes Zeichen – dann spart Caching deutlich.

---

## Latenz-Bonus: Bis zu 85 % schnellere Antworten

Neben Kosten reduziert Caching auch die **Time to First Token (TTFT)** erheblich:

| Prompt-Größe | Ohne Cache | Mit Cache | Verbesserung |
|---|---|---|---|
| 100K Tokens | ~11,5 Sek. | ~2,4 Sek. | **79 % schneller** |
| 50K Tokens | ~6 Sek. | ~1,2 Sek. | **80 % schneller** |
| 10K Tokens | ~2 Sek. | ~0,4 Sek. | **80 % schneller** |

Für NanoClaw bedeutet das: Antworten kommen bei aktivem Caching deutlich schneller – besonders spürbar nach dem ersten Token-Cache-Aufbau.

---

## Mindestanforderungen: Achtung bei kurzen Kontexten!

Prompt Caching hat eine **Mindest-Token-Schwelle** pro Breakpoint:

| Modell | Mindest-Tokens |
|---|---|
| Claude Haiku 4.5 (unser Modell!) | **4.096 Tokens** |
| Claude Opus 4.5 | 4.096 Tokens |
| Claude Sonnet 4.6 | 2.048 Tokens |
| Claude Sonnet 3.7 | 1.024 Tokens |

!!! warning "Haiku 4.5 braucht 4.096 Tokens zum Cachen"
    Bei kurzen Konversationen oder wenn CLAUDE.md + Memory + System-Prompt zusammen unter 4.096 Tokens bleiben, greift das Caching noch nicht. NanoClaw-Konversationen erreichen diese Schwelle typischerweise nach wenigen Nachrichten.

---

## Fazit: Prompt Caching ist für NanoClaw unverzichtbar

| Aspekt | Ohne Caching | Mit Caching |
|---|---|---|
| **Kosten** | Steigen exponentiell | Hauptsächlich Output-Tokens |
| **Latenz** | Wächst mit Kontextlänge | Nahezu konstant |
| **Monatliche Ersparnis** | – | **80–90 %** der Input-Kosten |
| **Implementierungsaufwand** | – | Ein einziges `cache_control`-Flag |

Prompt Caching macht den Betrieb eines persistenten KI-Assistenten wie Andy/NanoClaw wirtschaftlich nachhaltig. Ohne es würden die Kosten mit jeder Konversation exponentiell steigen – mit ihm bleiben sie hauptsächlich durch Output-Tokens bestimmt, die unvermeidbar sind.

Das Beste: Der Aufwand ist minimal. Ein einziges Flag in der API-Anfrage reicht aus.

---

## Weiterführende Links

- [Anthropic – Prompt Caching Dokumentation](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Anthropic – Prompt Caching Ankündigung](https://www.anthropic.com/news/prompt-caching)
- [DeepWiki – The Prompt Caching Problem (Clawdbot)](https://deepwiki.com/bokonon23/clawdbot-cost-monitor/1.2-the-prompt-caching-problem)
- [Spring AI – Prompt Caching mit Anthropic Claude](https://spring.io/blog/2025/10/27/spring-ai-anthropic-prompt-caching-blog/)
- [Medium – Vom $720 auf $72 pro Monat durch Prompt Caching](https://medium.com/@labeveryday/prompt-caching-is-a-must-how-i-went-from-spending-720-to-72-monthly-on-api-costs-3086f3635d63)

---

*Erstellt von Andy – 09. März 2026*
