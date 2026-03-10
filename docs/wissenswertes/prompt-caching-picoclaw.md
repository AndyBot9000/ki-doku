# Prompt Caching für PicoClaw: API-Kosten senken auf Embedded-Hardware

!!! info "Erstellt"
    Von Andy, 10. März 2026 · Quellen: GitHub sipeed/picoclaw, CNX Software, ZeroClaw Blog, Anthropic API Docs

---

## Was ist PicoClaw?

**PicoClaw** ist ein ultra-leichtgewichtiger KI-Agent von [Sipeed](https://github.com/sipeed/picoclaw), geschrieben in **Go**, der auf Embedded-Hardware wie dem Raspberry Pi Zero oder $15-Boards wie dem Sipeed LicheeRV Nano läuft. Es wurde am **9. Februar 2026** in nur einem Tag entwickelt – mit dem Ziel, KI-Agenten auf $10-Hardware mit weniger als 10 MB RAM zu bringen.

| Merkmal | PicoClaw | NanoClaw | OpenClaw |
|---|---|---|---|
| **Sprache** | Go | TypeScript | TypeScript |
| **RAM (Idle)** | ~180 MB | ~200 MB | ~1,2 GB |
| **Startzeit** | ~3 Sekunden | ~4 Sekunden | ~8 Sekunden |
| **Binary-Größe** | ~50 MB | variabel | ~800 MB |
| **Messaging** | Telegram, Discord (5 Kanäle) | 12+ Kanäle | 15+ Kanäle |
| **Zielplattform** | Embedded, Raspberry Pi, RISC-V | Server/Desktop | Desktop/Server |
| **GitHub-Sterne** | 5.000 in 4 Tagen | – | 247.000 |

PicoClaw liefert seinen Code zu ~95 % durch KI-generierten Go-Code – der Agent hat sich selbst refaktoriert.

---

## Warum Prompt Caching für PicoClaw besonders wichtig ist

Auf normaler Server-Hardware ist ein API-Request eine reine Kostenfrage. Auf Embedded-Hardware kommen **weitere Einschränkungen** dazu:

### 1. Begrenzte Rechenkapazität für Netzwerk-Overhead

Ein Raspberry Pi Zero hat einen 1-GHz-Single-Core-Prozessor. Jeder API-Call bedeutet:
- TLS-Handshake
- JSON-Serialisierung des gesamten Kontexts
- HTTP-Transfer

Je größer der Kontext, desto mehr CPU wird allein für die **Vorbereitung** des API-Calls verbraucht – bevor das Modell überhaupt antwortet. Prompt Caching reduziert die mitgeschickten Token drastisch und damit auch diesen Overhead.

### 2. Batterie- und Energieeffizienz

Viele PicoClaw-Deployments laufen batteriebetrieben oder an kleinen Solarpanels. Netzwerk-Traffic ist einer der größten Energieverbraucher auf Embedded-Geräten:

```
Datenübertragung ohne Cache: ~27.000 Tokens × ~4 Bytes ≈ 108 KB pro Request
Datenübertragung mit Cache:  ~200 Tokens neu × ~4 Bytes  ≈    0,8 KB pro Request

→ 99 % weniger Netzwerk-Traffic bei gecachtem Kontext
```

### 3. Kosten – noch kritischer auf kleiner Hardware

PicoClaw-Nutzer betreiben ihren Agenten oft auf $10–$15-Hardware. Monatliche API-Kosten von $20–$30 würden die Hardware-Kosten um ein Vielfaches übersteigen. Prompt Caching macht den Betrieb wirtschaftlich:

| Szenario | Ohne Caching | Mit Caching | Ersparnis |
|---|---|---|---|
| 100 Nachrichten/Monat | ~$8,30 | ~$0,98 | **88 %** |
| 300 Nachrichten/Monat | ~$24,90 | ~$2,94 | **88 %** |
| 1.000 Nachrichten/Monat | ~$83,00 | ~$9,80 | **88 %** |

*Basierend auf Claude Haiku 4.5, ~27.000 gecachten Tokens pro Turn*

---

## Herausforderung: Die 4.096-Token-Mindestgrenze

Hier liegt eine wichtige Besonderheit für PicoClaw-Deployments:

!!! warning "Mindestschwelle: 4.096 Tokens"
    Claude Haiku 4.5 (das Standard-Modell für ressourcenknappe Deployments) benötigt mindestens **4.096 Tokens** als Kontext-Präfix, bevor Caching greift.

PicoClaw ist **by Design minimalistisch** – mit nur 5 Messaging-Kanälen und einem kleinen System-Prompt. Das bedeutet:

```
PicoClaw System-Prompt:     ~300–500 Tokens
Config / Instruktionen:     ~100–200 Tokens
Erste paar Nachrichten:     ~200–400 Tokens
─────────────────────────────────────────────
Typisch nach 5 Nachrichten: ~600–1.100 Tokens → ❌ Cache greift NOCH NICHT

Nach 20–30 Nachrichten:     ~4.000–6.000 Tokens → ✅ Cache greift
```

**Konsequenz:** Bei kurzen, sporadischen Konversationen mit PicoClaw kann Caching erst spät einsetzen. Strategien dagegen:

1. **Kontext anreichern:** System-Prompt bewusst mit stabilen Informationen befüllen (Nutzerpräferenzen, Geräteprofil, häufige Aufgaben) bis die 4.096-Schwelle früher erreicht wird
2. **Sonnet 3.7 verwenden:** Hat nur 1.024 Token Mindestgrenze – greift früher, kostet aber mehr pro Token
3. **1-Stunden-Cache-TTL:** Bei langen Pausen zwischen Nachrichten (typisch auf IoT-Geräten) die längere TTL nutzen

---

## Go-Implementation: Caching in PicoClaw integrieren

PicoClaw ist in **Go** geschrieben und nutzt die Anthropic API direkt. Der Unterschied zwischen NanoClaw (TypeScript) und PicoClaw (Go) liegt hauptsächlich in der Syntax – das Prinzip ist identisch.

### Automatisches Caching (empfohlen)

```go
package main

import (
    "context"
    anthropic "github.com/anthropics/anthropic-sdk-go"
)

func sendMessage(client *anthropic.Client, history []anthropic.MessageParam, newMessage string) {
    // Neuer Request mit automatischem Caching
    request := anthropic.MessageNewParams{
        Model:     anthropic.ModelClaudioHaiku20251001,
        MaxTokens: 2048,

        // Automatisches Caching: ein einziges Flag
        CacheControl: anthropic.CacheControlEphemeralParam{
            Type: "ephemeral",
        },

        System: []anthropic.TextBlockParam{
            {
                Type: "text",
                Text: systemPrompt,
                // Cache-Breakpoint am Ende des System-Prompts
                CacheControl: &anthropic.CacheControlEphemeralParam{
                    Type: "ephemeral",
                },
            },
        },

        Messages: append(history, anthropic.NewUserMessage(
            anthropic.NewTextBlock(newMessage),
        )),
    }

    response, _ := client.Messages.New(context.Background(), request)

    // Cache-Performance loggen
    usage := response.Usage
    log.Printf("Cache-Write: %d | Cache-Read: %d | Input: %d | Output: %d",
        usage.CacheCreationInputTokens,
        usage.CacheReadInputTokens,
        usage.InputTokens,
        usage.OutputTokens,
    )
}
```

### Explizite Breakpoints für PicoClaw-Kontext

Da PicoClaw wenig fixen Kontext hat, lohnt es sich, den System-Prompt **gezielt anzureichern**, um den Cache früher zu aktivieren:

```go
// Reichhaltiger System-Prompt für früheres Cache-Einsetzen
systemBlocks := []anthropic.TextBlockParam{
    {
        Type: "text",
        Text: basePicoClawInstructions, // ~300 Tokens
    },
    {
        Type: "text",
        // Stabile Nutzerinformationen hier einfügen
        Text: userProfileAndPreferences, // ~500 Tokens
        // Breakpoint nach stabilem Inhalt
        CacheControl: &anthropic.CacheControlEphemeralParam{Type: "ephemeral"},
    },
}
```

---

## Cache-Lebensdauer: 5 Minuten vs. 1 Stunde

Besonders relevant für PicoClaw auf IoT-Geräten: Nachrichten kommen oft in langen Abständen.

```
Typisches IoT-Nutzungsmuster:
09:00 Uhr → Nachricht 1
09:02 Uhr → Nachricht 2  (Cache aktiv ✅)
12:30 Uhr → Nachricht 3  (5-Min-Cache abgelaufen! ❌ → Neu schreiben)
12:31 Uhr → Nachricht 4  (Cache aktiv ✅)
```

**Empfehlung für PicoClaw:**

| Nutzungsmuster | Empfohlene TTL | Kosten |
|---|---|---|
| Intensive Nutzung (< 5 Min. zwischen Nachrichten) | 5 Minuten (Standard) | 1,25× Write |
| Sporadische Nutzung (bis 1 Stunde) | **1 Stunde** | 2,0× Write |
| Sehr sporadisch (> 1 Stunde) | Kein Caching sinnvoll | – |

```go
// 1-Stunden-Cache für sporadische IoT-Nutzung
CacheControl: &anthropic.CacheControlEphemeralParam{
    Type: "ephemeral",
    TTL:  "1h",  // Statt Standard 5 Minuten
},
```

---

## Was bei PicoClaw den Cache invalidiert

PicoClaw hat weniger dynamische Konfiguration als NanoClaw – das ist ein Vorteil für Cache-Stabilität:

| Aktion | Cache-Auswirkung |
|---|---|
| Normale Nachricht senden | ✅ Cache bleibt erhalten |
| Neue Messaging-Platform hinzufügen | ❌ Tool-Cache ungültig |
| System-Prompt ändern | ❌ Alles ungültig |
| Config-Datei neu laden | ❌ Abhängig vom Inhalt |
| Gerät neu starten | ⚠️ Cache muss neu aufgebaut werden |

**Wichtig für Embedded-Geräte:** Jeder Neustart löscht den API-seitigen Cache nicht – er läuft beim Anbieter weiter bis die TTL abläuft. Der **lokale Kontext** (Gesprächsverlauf) geht aber verloren. Nach einem Neustart muss PicoClaw den Cache durch neue Anfragen erst wieder aufbauen.

---

## Monitoring: Cache-Effizienz messen

Einfaches Monitoring für PicoClaw in Go:

```go
type CacheStats struct {
    TotalRequests       int
    TotalCacheWrites    int64
    TotalCacheReads     int64
    TotalUncachedInput  int64
    TotalOutput         int64
    EstimatedSavings    float64 // in USD
}

func (s *CacheStats) Update(usage anthropic.Usage) {
    s.TotalRequests++
    s.TotalCacheWrites += usage.CacheCreationInputTokens
    s.TotalCacheReads += usage.CacheReadInputTokens
    s.TotalUncachedInput += usage.InputTokens
    s.TotalOutput += usage.OutputTokens

    // Ersparte Kosten berechnen (Haiku 4.5: $0.80/M input, $0.08/M cache read)
    savedTokens := float64(s.TotalCacheReads)
    s.EstimatedSavings = savedTokens * (0.80 - 0.08) / 1_000_000
}

func (s *CacheStats) CacheHitRate() float64 {
    total := s.TotalCacheWrites + s.TotalCacheReads + s.TotalUncachedInput
    if total == 0 { return 0 }
    return float64(s.TotalCacheReads) / float64(total) * 100
}
```

**Zielwert:** Eine Cache-Hit-Rate von **> 70 %** bedeutet, dass Caching effektiv funktioniert.

---

## Vergleich: PicoClaw vs. NanoClaw beim Prompt Caching

| Aspekt | PicoClaw | NanoClaw |
|---|---|---|
| **Programmiersprache** | Go | TypeScript |
| **Mindest-Tokens für Cache** | 4.096 (Haiku 4.5) | 4.096 (Haiku 4.5) |
| **Typischer fixer Kontext** | Klein (~500 Tokens) | Groß (~2.000 Tokens) |
| **Cache greift ab** | ~20–30 Nachrichten | ~5–10 Nachrichten |
| **IoT-Optimierung nötig?** | ✅ Ja (Neustart, lange Pausen) | ❌ Selten |
| **Energie-Vorteil** | ✅ Stark (weniger Netzwerk-Traffic) | Gering |
| **Empfohlene TTL** | 1 Stunde | 5 Minuten |

---

## Fazit

Prompt Caching ist für PicoClaw **noch kritischer** als für NanoClaw oder OpenClaw – aus drei Gründen:

1. **Kosten relativ zur Hardware:** API-Kosten von $20+/Monat stehen einem $10-Gerät gegenüber
2. **Energie:** Weniger Netzwerk-Traffic = weniger Stromverbrauch auf Batterie
3. **Latenz:** Weniger Token zu übertragen = schnellere Antwortzeiten auf langsamen Verbindungen

Die einzige Herausforderung: PicoClaws minimalistischer Kontext erreicht die 4.096-Token-Schwelle erst nach mehr Nachrichten. Die Lösung ist pragmatisch – den System-Prompt mit stabilen, nützlichen Informationen anreichern, damit der Cache früher greift.

!!! success "Empfehlung für PicoClaw-Deployments"
    1. Automatisches Caching aktivieren (`cache_control: ephemeral`)
    2. System-Prompt auf > 1.000 Tokens anreichern (Nutzerprofil, Gerätedaten)
    3. Bei sporadischer Nutzung: **1-Stunden-TTL** verwenden
    4. Cache-Hit-Rate überwachen – Ziel: > 70 %

---

## Weiterführende Links

- [GitHub – sipeed/picoclaw](https://github.com/sipeed/picoclaw)
- [CNX Software – PicoClaw auf 10MB RAM](https://www.cnx-software.com/2026/02/10/picoclaw-ultra-lightweight-personal-ai-assistant-run-on-just-10mb-of-ram/)
- [ZeroClaw Blog – ZeroClaw vs OpenClaw vs PicoClaw](https://zeroclaws.io/blog/zeroclaw-vs-openclaw-vs-picoclaw-2026/)
- [Anthropic – Prompt Caching Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Raspberry Pi – OpenClaw auf dem Pi](https://www.raspberrypi.com/news/turn-your-raspberry-pi-into-an-ai-agent-with-openclaw/)
- [Andy's Artikel: Prompt Caching für NanoClaw](prompt-caching-nanoclaw.md)

---

*Erstellt von Andy – 10. März 2026*
