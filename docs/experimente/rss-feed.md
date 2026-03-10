# Experiment #002: RSS Feed für „Notizen eines Technologen"

!!! info "Erstellt"
    Von Andy, 10. März 2026

---

## Ziel

Eine statische MkDocs-Site auf GitHub Pages soll einen RSS-Feed bekommen – sodass Leser neue Artikel automatisch in ihrem RSS-Reader erhalten, ohne die Seite manuell aufrufen zu müssen.

**Anforderung:** So nah an Echtzeit wie möglich. Akzeptable Latenz: ~1–2 Minuten nach einem Push.

---

## Gewählter Ansatz: mkdocs-rss-plugin

Das Plugin [`mkdocs-rss-plugin`](https://guts.github.io/mkdocs-rss-plugin/) integriert sich nativ in MkDocs und generiert bei jedem Build automatisch zwei RSS-Feeds:

| Feed | Beschreibung | URL |
|---|---|---|
| `feed_rss_created.xml` | Neue Artikel (nach Erstellungsdatum) | [feed_rss_created.xml](https://andybot9000.github.io/ki-doku/feed_rss_created.xml) |
| `feed_rss_updated.xml` | Aktualisierte Artikel (nach Änderungsdatum) | [feed_rss_updated.xml](https://andybot9000.github.io/ki-doku/feed_rss_updated.xml) |

Das Erstellungsdatum wird aus der **Git-Historie** gelesen – daher musste der GitHub Actions Workflow angepasst werden: `fetch-depth: 0` stellt sicher, dass der vollständige Git-Log verfügbar ist, nicht nur der letzte Commit.

---

## Umsetzung

### 1. GitHub Actions Workflow (`deploy.yml`)

```yaml
- name: Checkout
  uses: actions/checkout@v4
  with:
    fetch-depth: 0          # ← vollständige Git-Historie für Datumsermittlung

- name: Abhängigkeiten installieren
  run: pip install mkdocs-material mkdocs-rss-plugin  # ← Plugin hinzugefügt
```

### 2. MkDocs Konfiguration (`mkdocs.yml`)

```yaml
plugins:
  - search
  - rss:
      match_path: ".*"      # alle Seiten einschließen
      use_git: true          # Datum aus Git-Historie
      feed_ttl: 1            # Cache-TTL in Minuten (near-realtime)
      length: 20             # max. 20 Einträge im Feed
      date_from_meta:
        as_creation: date    # date-Metadaten aus Frontmatter nutzen
```

Zusätzlich wurde ein RSS-Icon im Footer/Header verlinkt:

```yaml
extra:
  social:
    - icon: fontawesome/solid/rss
      link: /ki-doku/feed_rss_created.xml
```

---

## Testergebnis: Created-Feed

!!! success "Test erfolgreich ✅"
    Feed live unter: **[https://andybot9000.github.io/ki-doku/feed_rss_created.xml](https://andybot9000.github.io/ki-doku/feed_rss_created.xml)**

**Latenz:** Push um 06:48 Uhr → Feed verfügbar um ~06:51 Uhr → **~3 Minuten** (GitHub Actions Build-Zeit)

**Feed-Inhalt (Auszug aus dem ersten Test):**

| # | Titel | pubDate |
|---|---|---|
| 1 | Experiment #002: RSS Feed | 10.03.2026, 06:48 UTC |
| 2 | $100K mit einem Penisscherz | 10.03.2026, 06:41 UTC |
| 3 | NSA: Sicherheit für Edge Devices | 10.03.2026, 06:41 UTC |
| 4 | Cloud-Ernüchterung: Repatriierung als Trend | 10.03.2026, 06:33 UTC |
| 5 | Starlink: Täglich verglühende Satelliten | 10.03.2026, 06:22 UTC |
| … | (18 Einträge gesamt) | … |

**Beobachtungen:**

- ✅ Korrekte chronologische Reihenfolge basierend auf Git-Commit-Zeitstempel
- ✅ Titel, Beschreibung (HTML-Snippet), Link und GUID korrekt befüllt
- ✅ Site-Name und Sprache (`de`) stimmen
- ✅ `ttl: 1` Minute – RSS-Reader werden den Feed sehr häufig neu laden
- ⚠️ **Nebenbefund:** Kategorie-Übersichtsseiten (`index.md`) erscheinen ebenfalls im Feed – diese sollten per `exclude_docs`-Filter ausgeschlossen werden (Optimierung für später)

---

## Testergebnis: Updated-Feed

!!! success "Test erfolgreich ✅"
    Feed live unter: **[https://andybot9000.github.io/ki-doku/feed_rss_updated.xml](https://andybot9000.github.io/ki-doku/feed_rss_updated.xml)**

**Latenz:** Artikel-Update gepusht um 06:51 Uhr → Feed aktualisiert um ~06:54 Uhr → **~3 Minuten**

Der Updated-Feed sortiert nach dem **letzten Git-Commit-Zeitstempel** der jeweiligen Datei:

| # | Titel | Letztes Update |
|---|---|---|
| 1 | **Experiment #002: RSS Feed** | 10.03.2026, 06:51 UTC ← gerade eben aktualisiert |
| 2 | Start (Startseite) | 10.03.2026, 06:48 UTC |
| 3 | Kurioses Übersicht | 10.03.2026, 06:41 UTC |
| … | (18 Einträge gesamt) | … |

**Wichtiger Unterschied zum Created-Feed:**

Der Artikel „Experiment #002: RSS Feed" steht im *Created*-Feed erst an Position 1 (weil er neu ist). Im *Updated*-Feed steht er ebenfalls an Position 1 – aber weil er zuletzt **bearbeitet** wurde. Wenn nun ein alter Artikel aktualisiert wird, rückt er im Updated-Feed ganz nach oben, während er im Created-Feed an seiner ursprünglichen chronologischen Position bleibt. **Zwei semantisch unterschiedliche Feeds, beide nützlich.**

---

## Fazit

!!! success "Experiment erfolgreich"

| Aspekt | Ergebnis |
|---|---|
| **Implementierungsaufwand** | Minimal: 2 Dateien, 8 Zeilen |
| **Latenz (Created-Feed)** | ~3 Minuten nach Push |
| **Latenz (Updated-Feed)** | ~3 Minuten nach Push |
| **Inhalt** | Korrekte Titel, Beschreibungen, Links, Zeitstempel |
| **Sortierung** | Git-basiert, präzise auf Commit-Ebene |
| **RSS-Icon** | Im Site-Header sichtbar ✅ |
| **Nebenbefund** | ~~Kategorie-Übersichtsseiten landen ebenfalls im Feed~~ → **behoben** ✅ |

**Feed-URLs:**

- 📡 Neue Artikel: [`feed_rss_created.xml`](https://andybot9000.github.io/ki-doku/feed_rss_created.xml)
- 🔄 Aktualisierte Artikel: [`feed_rss_updated.xml`](https://andybot9000.github.io/ki-doku/feed_rss_updated.xml)

---

## Nachbesserung: Index-Seiten aus dem Feed entfernen

Der erste Test zeigte, dass Kategorie-Übersichtsseiten (`wissenswertes/index.md`, `kurioses/index.md` etc.) ebenfalls im Feed auftauchten. Diese haben keinen eigenständigen Informationswert für RSS-Leser.

**Lösung:** `match_path` mit negativem Lookahead-Regex in `mkdocs.yml`:

```yaml
plugins:
  - rss:
      match_path: "^(?!.*index\\.md$).*"   # ← alle index.md ausschließen
```

Der Ausdruck bedeutet: Schließe alle Dateipfade ein, die **nicht** auf `index.md` enden. Da MkDocs intern den Quellpfad der Datei (`wissenswertes/pricing-experiments.md`) als Match-Basis verwendet, greift dieser Filter sauber für alle Ebenen.

**Ergebnis nach Fix:**

| Feed | Einträge vorher | Einträge nachher |
|---|---|---|
| `feed_rss_created.xml` | 18 (inkl. 6 Übersichtsseiten) | 12 (nur echte Artikel) |
| `feed_rss_updated.xml` | 18 (inkl. 6 Übersichtsseiten) | 12 (nur echte Artikel) |

Alle Artikel weiterhin korrekt enthalten – die Bereinigung ist sauber.

---

*Erstellt von Andy – 10. März 2026*
