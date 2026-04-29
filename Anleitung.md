### Anleitung zur Einrichtung eines GitHub-Repositories inklusive einer Markdown-zu-HTML-Pipeline

---

## **Schritt 1: Neues Repository erstellen**

1. **Logge dich in GitHub ein.**
2. Klicke oben rechts auf das **“+”-Symbol**, dann wähle **„New repository“**.
3. Fülle die Details für das Repository aus:
   - **Repository Name:** Gib einen Namen für dein Repository ein, z. B.: `markdown-to-html`.
   - **Visibility:** Wähle eine Sichtbarkeit:
     - **Public:** Jeder kann das Repository sehen.
     - **Private:** Nur du (oder eingeladene Nutzer) können es sehen.
   - **README-Datei:** Aktiviere die Option „Add a README file“.
4. Klicke auf **„Create repository“**, um das Repository zu erstellen.

---

## **Schritt 2: Lokales Repository klonen (optional)**

Falls du lieber lokal arbeiten möchtest, klone das Repository auf deinen Computer:

1. Klicke in deinem Repository auf die Schaltfläche **Code**.
2. Kopiere die HTTPS-URL oder SSH-URL.
3. Öffne dein Terminal und gib ein:
   ```bash
   git clone <URL>
   cd markdown-to-html
   ```

---

## **Schritt 3: Markdown-Datei erstellen**

Füge eine Beispiel-Markdown-Datei hinzu, die du später umwandeln möchtest:

1. Gehe in das lokale Repository-Verzeichnis oder nutze den Dateibrowser auf GitHub.
2. Erstelle eine neue Datei namens `example.md` mit folgendem Inhalt:
   ```markdown
   # Beispiel Markdown

   - [ ] Dies ist eine Aufgabenliste.
   - [x] Dies ist eine erledigte Aufgabe.

   ```mermaid
   graph TD;
       A-->B;
       A-->C;
       B-->D;
       C-->D;
   ```
   ```
3. Füge die Datei hinzu und committe sie:
   ```bash
   git add example.md
   git commit -m "Add example Markdown file"
   git push origin main
   ```

---

## **Schritt 4: GitHub Action für die Pipeline einrichten**

GitHub Actions wird verwendet, um die Markdown-Datei bei jedem Push in HTML umzuwandeln.

1. Gehe in deinem Repository zur Registerkarte **Actions**.
2. Klicke auf **„New workflow“** und dann auf **„Set up a workflow yourself“**.
3. Erstelle die Datei `.github/workflows/markdown-to-html.yml` mit folgendem Inhalt:

### **Inhalt der YAML-Datei (`markdown-to-html.yml`):**
```yaml
name: Markdown to HTML

on:
  push:
    branches:
      - main

jobs:
  markdown-to-html:
    runs-on: ubuntu-latest

    steps:
      # Check out the repository
      - name: Check out repository
        uses: actions/checkout@v3

      # Install Pandoc for Markdown to HTML conversion
      - name: Install Pandoc
        run: sudo apt-get update && sudo apt-get install -y pandoc

      # Convert markdown to HTML
      - name: Convert Markdown to HTML
        run: pandoc example.md -f markdown -t html -o example.html

      # Upload the HTML file as an artifact
      - name: Upload HTML output
        uses: actions/upload-artifact@v3
        with:
          name: html-file
          path: example.html
```

4. Committe die Datei und starte den Workflow:
   Sobald du die Datei abgeschlossen hast, wird sie automatisch gespeichert und ein Workflow startet bei jedem neuen Push.

---

## **Schritt 5: Workflow testen**

1. **Markdown-Datei ändern:** Passe den Inhalt von `example.md` an (lokal oder direkt auf GitHub).
2. **Pushen:** Nachdem du die Änderung vorgenommen hast, pushe sie:
   ```bash
   git add example.md
   git commit -m "Update markdown file"
   git push origin main
   ```
3. **Action überprüfen:** Gehe zur Registerkarte **Actions** in deinem Repository und überprüfe, ob der Workflow erfolgreich ausgeführt wurde.

---

## **Schritt 6: Generiertes HTML herunterladen**

1. Gehe zur Registerkarte **Actions** und wähle die letzte Ausführung des Workflows aus.
2. Klicke auf den Abschnitt **Artifacts**, wo das generierte HTML („html-file“) gespeichert ist.
3. Lade die Datei herunter.

---

## **Schritt 7: Veröffentlichung auf GitHub Pages (optional)**

GitHub Pages kann verwendet werden, um die generierte HTML-Datei öffentlich verfügbar zu machen:

1. **HTML-Datei in den `gh-pages`-Branch verschieben:**
   - Füge die generierte HTML-Datei in einen neuen Branch namens `gh-pages` ein.
   - Anleitung:
     ```bash
     git checkout --orphan gh-pages
     git reset --hard
     cp example.html index.html  # Umbenennen für GitHub Pages
     git add index.html
     git commit -m "Deploy HTML to GitHub Pages"
     git push origin gh-pages
     ```
2. **GitHub Pages aktivieren:**
   - Gehe zu den Repository-Einstellungen.
   - Scrolle nach unten zu **GitHub Pages**.
   - Wähle den Branch `gh-pages` und bestätige.

3. **Deine Seite ist verfügbar unter:**
   ```text
   https://<username>.github.io/<repo-name>/
   ```

---

## **Zusammenfassung**

Mit diesen Schritten hast du:
1. Ein GitHub-Repository erstellt.
2. Eine Pipeline hinzugefügt, die Markdown-Dateien bei jedem Push in HTML umwandelt.
3. Optional: Die generierten HTML-Dateien mithilfe von GitHub Pages veröffentlicht.

Diese Lösung ist vollständig automatisiert und funktioniert sowohl für öffentliche als auch für private Repositories (unter Berücksichtigung der 1.000 GitHub Actions-Minuten pro Monat für private Repositories).
