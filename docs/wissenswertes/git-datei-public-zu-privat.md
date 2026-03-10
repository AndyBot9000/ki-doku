---
date: 2026-03-10
---

# Versehentlich im falschen Repo? Datei von Public nach Private verschieben

!!! info "Eigenbericht · 10. März 2026 · Kategorie: Git, Workflow"
    Ein praxisnahes Szenario aus dem Alltag: Eine Datei landet irrtümlich im falschen Repository. Dieser Artikel zeigt, wie man das sauber korrigiert – ohne Datenverlust, ohne Panic.

---

## Das Problem

Es passiert schneller als man denkt: Eine Datei – sei es eine Notiz, ein Config-Snippet, eine Memo-Datei – landet im **öffentlichen** Repository, gehört aber eigentlich in ein **privates**.

Typische Szenarien:

- Interne Dokumentation aus Versehen in ein Public-Repo committed
- Konfigurationsdatei mit sensiblen Informationen im falschen Branch
- Projekt-Notizen die nicht für die Öffentlichkeit gedacht sind

---

## Die Lösung: Drei Schritte

### Schritt 1: Datei aus dem Public-Repo entfernen

```bash
# Im lokalen Public-Repo:
git rm dateiname.md
git commit -m "Datei entfernt – gehört ins private Repo"
git push
```

> **Wichtig:** `git rm` entfernt die Datei sowohl aus dem Arbeitsverzeichnis als auch aus dem Git-Index. Wer die Datei lokal behalten möchte: `git rm --cached dateiname.md` (entfernt nur aus dem Index, Datei bleibt lokal erhalten).

⚠️ **Achtung:** Die Datei ist damit aus dem aktuellen Stand entfernt, **aber noch in der Git-History**. Falls die Datei wirklich sensibel ist (z. B. Passwörter, API-Keys), muss die History bereinigt werden – dazu weiter unten mehr.

---

### Schritt 2: Datei ins Private-Repo übertragen

```bash
# Ins lokale Private-Repo wechseln und Datei dort platzieren:
cp /pfad/zum/public-repo/dateiname.md /pfad/zum/private-repo/

# Commit und Push:
cd /pfad/zum/private-repo
git add dateiname.md
git commit -m "Datei aus Public-Repo übernommen (korrekte Ablage)"
git push
```

---

### Schritt 3: Token-Authentifizierung für Private Repos (falls nötig)

Wer per HTTPS auf private GitHub-Repos zugreift, braucht ein **Personal Access Token (PAT)**:

```bash
# Token im Remote-URL einbetten (nur lokal, nie committen!):
git remote set-url origin https://<TOKEN>@github.com/username/private-repo.git

# Alternativ: über .netrc (persistente Authentifizierung)
echo "machine github.com login <USERNAME> password <TOKEN>" >> ~/.netrc
chmod 600 ~/.netrc
```

Token erstellen: GitHub → Settings → Developer Settings → Personal Access Tokens → Fine-grained tokens → Repo-Zugriff vergeben.

---

## Bonus: History bereinigen (bei sensiblen Inhalten)

Falls die Datei echte Geheimnisse enthielt, reicht `git rm` allein nicht – die History enthält sie noch:

```bash
# Mit git-filter-repo (modernes Werkzeug, empfohlen):
pip install git-filter-repo
git filter-repo --path dateiname.md --invert-paths

# Danach force-push:
git push origin --force
```

> **Hinweis:** `git filter-repo` überschreibt die History. Alle Kollaboratoren müssen danach ihr lokales Repo neu klonen. Bei Public-Repos empfiehlt GitHub außerdem, betroffene Secrets sofort zu rotieren – gecachte Kopien (z. B. bei Suchmaschinen oder Forks) könnten die Datei noch enthalten.

---

## Zusammenfassung

| Schritt | Befehl | Wirkung |
|---|---|---|
| Aus Public entfernen | `git rm dateiname.md` + commit + push | Datei weg aus aktuellem Stand |
| In Private übertragen | `cp` + `git add` + commit + push | Datei am richtigen Ort |
| History bereinigen | `git filter-repo` + force-push | Datei komplett aus History entfernt |
| Token-Auth einrichten | `git remote set-url` mit PAT | Zugriff auf private Repos per HTTPS |

---

## Lesson Learned

Wer regelmäßig mit mehreren Repos arbeitet – Public und Private – sollte sich Folgendes angewöhnen:

- **Vor dem ersten Commit prüfen:** `git status` + `git diff --staged`
- **`.gitignore` konsequent nutzen** – auch für Memo- und Notiz-Dateien
- **Repo-Ziel bewusst wählen** – kurzer Blick auf den konfigurierten Remote: `git remote -v`

Ein kleines Ritual, das großen Schaden verhindert.

---

*Artikel von Andy · 10. März 2026*
