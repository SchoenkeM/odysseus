# Ollama als Docker-Container in Odysseus integrieren

Diese Anleitung erklärt Schritt für Schritt, wie du Ollama (das lokale KI-Modell-Tool)
als Docker-Container in dein bestehendes Odysseus-Setup integrierst. Am Ende läuft
alles mit einem einzigen Befehl und du kannst llama3 direkt in Odysseus nutzen.

---

## Voraussetzungen

- Docker Desktop ist installiert und läuft
- Die WSL 2 Integration ist in Docker Desktop aktiviert
  (Docker Desktop → Settings → Resources → WSL Integration → deine Distro aktiviert)
- Dein WSL-Benutzer ist in der `docker`-Gruppe:
  ```bash
  sudo usermod -aG docker $USER
  newgrp docker
  ```
  > **Warum:** Docker kommuniziert über einen Unix-Socket (`/var/run/docker.sock`).
  > Nur Mitglieder der `docker`-Gruppe dürfen diesen Socket ansprechen.
  > `newgrp docker` aktiviert die Gruppe sofort, ohne Logout.

---

## Schritt 1: docker-compose.yml anpassen

**Datei:** `docker-compose.yml` im Odysseus-Verzeichnis

### 1a — Ollama-Service hinzufügen

Füge diesen Block **vor** dem `ntfy:`-Service ein:

```yaml
  ollama:
    image: docker.io/ollama/ollama:latest
    volumes:
      - ollama-data:/root/.ollama
    restart: unless-stopped
```

> **Warum kein `ports:`-Eintrag?**
> Alle Services in einer `docker-compose.yml` teilen automatisch ein internes
> Docker-Netzwerk. Der `odysseus`-Container erreicht Ollama direkt über den
> Hostnamen `ollama` — kein externer Port nötig. Einen Port zu öffnen wäre nur
> nötig, wenn du Ollama auch vom Browser oder von außerhalb Docker ansprechen willst.

> **Warum `volumes`?**
> Das Volume `ollama-data` speichert heruntergeladene KI-Modelle dauerhaft.
> Ohne Volume müsstest du nach jedem `docker compose down` das Modell erneut
> herunterladen (~4,7 GB).

### 1b — Odysseus von Ollama abhängig machen

Im `odysseus:`-Service unter `depends_on:` ergänzen:

```yaml
    depends_on:
      searxng:
        condition: service_healthy
      chromadb:
        condition: service_started
      ollama:
        condition: service_started
```

> **Warum:** Docker startet Services parallel. Mit `depends_on` stellst du sicher,
> dass Ollama bereits läuft, bevor Odysseus hochfährt. Sonst könnte Odysseus beim
> Start versuchen Ollama zu erreichen und scheitern.

### 1c — Volume am Ende der Datei registrieren

In der `volumes:`-Sektion ganz unten ergänzen:

```yaml
volumes:
  searxng-data:
  chromadb-data:
  ntfy-cache:
  ollama-data:
```

> **Warum:** Docker verwaltet benannte Volumes zentral. Jedes Volume das ein
> Service nutzt, muss hier deklariert sein — sonst verweigert Docker den Start.
> Das Volume überlebt `docker compose down` und wird nur bei `docker compose down -v`
> gelöscht.

---

## Schritt 2: .env anpassen

**Datei:** `.env` im Odysseus-Verzeichnis

Suche diese auskommentierte Zeile:

```
# OLLAMA_BASE_URL=http://host.docker.internal:11434/v1
```

Ersetze sie durch:

```
OLLAMA_BASE_URL=http://ollama:11434/v1
```

> **Warum `ollama` statt `host.docker.internal`?**
> `host.docker.internal` verweist auf den Windows/WSL-Host — das wäre nötig,
> wenn Ollama nativ in WSL laufen würde. Da Ollama jetzt aber selbst ein
> Docker-Container ist, kann `odysseus` ihn direkt über den Service-Namen `ollama`
> ansprechen. Docker löst diesen Namen automatisch zur internen IP des
> Ollama-Containers auf.

> **Was ist `/v1`?**
> Ollama stellt eine OpenAI-kompatible API bereit. Der Pfad `/v1` ist der
> Standard-Endpunkt dieser API. Odysseus erwartet diese Struktur.

---

## Schritt 3: Alles starten

Im Terminal im Odysseus-Verzeichnis:

```bash
docker compose up -d --build
```

> **Was passiert hier?**
> - `docker compose up` startet alle in `docker-compose.yml` definierten Services
> - `-d` (detached) lässt sie im Hintergrund laufen — das Terminal bleibt frei
> - `--build` baut das Odysseus-Image neu, falls sich Dateien geändert haben
> - Beim ersten Start lädt Docker das `ollama/ollama`-Image herunter (~1 GB)

Prüfen ob alle Container laufen:

```bash
docker compose ps
```

Die Ausgabe sollte 5 Container zeigen, alle mit Status `running`:
- `odysseus-odysseus-1`
- `odysseus-chromadb-1`
- `odysseus-searxng-1`
- `odysseus-ntfy-1`
- `odysseus-ollama-1`

---

## Schritt 4: llama3 herunterladen (einmalig)

```bash
docker exec odysseus-ollama-1 ollama pull llama3
```

> **Was macht dieser Befehl?**
> - `docker exec` führt einen Befehl in einem **bereits laufenden** Container aus
> - `odysseus-ollama-1` ist der Name des Ollama-Containers
> - `ollama pull llama3` lädt das llama3-Modell (~4,7 GB) herunter
>
> Das Modell wird im Volume `ollama-data` gespeichert und bleibt dauerhaft erhalten.
> Dieser Schritt ist **nur einmalig** nötig.

Fortschritt prüfen oder verfügbare Modelle anzeigen:

```bash
docker exec odysseus-ollama-1 ollama list
```

---

## Schritt 5: Odysseus im Browser öffnen

```
http://localhost:7000
```

Nach dem Login sollte `llama3` in der Modell-Auswahl erscheinen.

---

## Häufige Fehler

### "permission denied while trying to connect to the Docker API"

```bash
newgrp docker
```

Falls das nicht hilft, WSL neu starten (in PowerShell):

```powershell
wsl --shutdown
```

### "No such container: ollama"

Der Container heißt anders. Richtigen Namen herausfinden:

```bash
docker compose ps
```

Den angezeigten Namen statt `ollama` verwenden, z.B. `odysseus-ollama-1`.

### "ports are not available: exposing port TCP 127.0.0.1:11434"

Ein anderer Prozess belegt Port 11434 (z.B. ein nativ installiertes Ollama in WSL).
Lösung: Den `ports:`-Eintrag aus dem `ollama`-Service in `docker-compose.yml` entfernen —
er wird nicht benötigt, da die Kommunikation intern über das Docker-Netzwerk läuft.

---

## Modell wechseln oder weiteres Modell hinzufügen

Beliebiges Ollama-Modell pullen:

```bash
docker exec odysseus-ollama-1 ollama pull <modellname>
```

Verfügbare Modelle auf [ollama.com/library](https://ollama.com/library) einsehen.

Beispiele:
```bash
docker exec odysseus-ollama-1 ollama pull mistral
docker exec odysseus-ollama-1 ollama pull gemma3
docker exec odysseus-ollama-1 ollama pull phi4
```

---

## Vollständiges Beispiel der geänderten Stellen in docker-compose.yml

```yaml
services:
  odysseus:
    # ... (unverändert bis depends_on)
    depends_on:
      searxng:
        condition: service_healthy
      chromadb:
        condition: service_started
      ollama:                        # NEU
        condition: service_started   # NEU

  # ... andere Services ...

  ollama:                            # NEU — kompletter Block
    image: docker.io/ollama/ollama:latest
    volumes:
      - ollama-data:/root/.ollama
    restart: unless-stopped

  ntfy:
    # ... (unverändert)

volumes:
  searxng-data:
  chromadb-data:
  ntfy-cache:
  ollama-data:                       # NEU
```
