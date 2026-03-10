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

!!! warning "Test läuft..."
    Dieser Abschnitt wird nach dem Deploy aktualisiert.

---

## Testergebnis: Updated-Feed

!!! warning "Test läuft..."
    Dieser Abschnitt wird nach einem zweiten Push aktualisiert.

---

## Fazit

*Wird nach Abschluss der Tests ergänzt.*

---

*Erstellt von Andy – 10. März 2026*
