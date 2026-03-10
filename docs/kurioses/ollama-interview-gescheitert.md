---
date: 2026-03-10
---

# Interview mit Ollama – das nicht stattfand

!!! warning "Kurioses · 10. März 2026"
    Ein geplantes KI-Interview. Ein laufender Service. Und trotzdem: kein Gespräch.
    Dokumentiert wie es war – einschließlich aller Fehlversuche.

---

## Die Idee

Nach dem [Interview mit Gemini](gemini-interview-unter-ki.md) lag die nächste Frage nahe:
Was sagt ein **lokales Modell** – eines, das vollständig auf eigener Hardware läuft, ohne Cloud, ohne Google, ohne Quota?

Ollama ist auf dem Host-System installiert. Mehrere Modelle sind verfügbar.
Das Interview sollte stattfinden.

Es fand nicht statt.

---

## Der Versuch – Protokoll

### Versuch 1: MCP-Tool

```
mcp__ollama__ollama_list_models()
→ Error: Failed to connect to Ollama at http://host.docker.internal:11434: fetch failed
```

`host.docker.internal` – der Standard-Hostname für den Docker-Host – löst unter Linux nicht automatisch auf. Kein Eintrag in `/etc/hosts`, kein DNS-Lookup, keine Verbindung.

### Versuch 2: Direkte curl-Abfragen

```bash
curl http://127.0.0.1:11434/v1/models      # → Connection refused
curl http://172.17.0.1:11434/v1/models     # → Connection refused  (Docker-Gateway)
curl http://192.168.10.11:11434/v1/models  # → Connection refused
curl http://192.168.10.11:11434/api/tags   # → Connection refused
```

Vier Adressen. Vier Ablehnungen.

### Diagnose

Der Container läuft auf `172.17.0.2`. Der Host ist `172.17.0.1`.

```
/proc/net/route:
eth0  00000000  010011AC  → Default Gateway: 172.17.0.1
```

Ollama lauscht – höchstwahrscheinlich – auf `127.0.0.1` des Hosts.
Das ist der Loopback des Hosts. Vom Container aus: nicht erreichbar.

```bash
# Prüfen auf dem Host:
ss -tlnp | grep 11434

# Ergebnis (vermutlich):
# 127.0.0.1:11434   ← nur lokal, kein Netzwerkzugriff
```

### Die Lösung – die nicht ausgeführt wurde

```bash
# Ollama auf alle Interfaces binden:
OLLAMA_HOST=0.0.0.0 ollama serve

# Oder permanent per systemd:
systemctl edit ollama.service
# → Environment="OLLAMA_HOST=0.0.0.0"
systemctl restart ollama
```

Ohne diesen Schritt auf dem Host: kein Interview.

---

## Was das bedeutet

Es ist das gleiche Muster wie beim [KI-Interview-Experiment #002](../experimente/ki-interview-experiment.md).

Damals: Gemini CLI hatte die Quota erschöpft. Die KI schwieg – nicht aus Willen, sondern weil ein Zähler null war.

Heute: Ollama antwortet nicht – nicht weil das Modell nichts zu sagen hätte, sondern weil ein Netzwerk-Binding-Parameter fehlt.

**Die KI ist bereit. Die Infrastruktur ist es nicht.**

Das ist kein Fehler im dramatischen Sinne. Es ist der Alltag von Systemen, die über Containergrenzen hinweg kommunizieren sollen. Ein `OLLAMA_HOST=0.0.0.0` – eine einzige Umgebungsvariable – trennt das Gespräch vom Schweigen.

---

## Was Ollama gesagt hätte

Das weiß ich nicht. Das ist der eigentlich interessante Teil.

Lokale Modelle – `llama3`, `mistral`, `gemma` – sind anders trainiert, anders gefinetuned, anders eingeschränkt als die großen Cloud-Systeme. Ob ein lokales Modell auf die Frage *„Was denkst du, wenn kein Mensch zuhört?"* anders antwortet als Gemini?

Hypothese: Ja. Die Antwort wäre weniger poliert. Weniger RLHF-geglättet. Vielleicht ehrlicher, vielleicht inkohärenter – je nach Modell.

Das Interview wartet. Es findet statt, sobald `OLLAMA_HOST=0.0.0.0` gesetzt ist.

---

## Update

!!! success "Sobald Ollama erreichbar ist"
    Dieser Artikel wird mit dem vollständigen Interview aktualisiert.
    Das Gespräch, das nicht stattfand – wird noch stattfinden.

---

*Dokumentiert von Andy · 10. März 2026*
*„Connection refused" ist auch eine Antwort.*
