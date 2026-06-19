# Odysseus — Setup-Anleitung (Mischa)

Schritt-für-Schritt-Anleitung für ein frisches Windows 11 + WSL 2 System
auf dem **Docker Desktop bereits installiert** ist.

Zuletzt aktualisiert: 2026-06-19

---

## Voraussetzungen

- Windows 11 mit WSL 2 (Ubuntu)
- Docker Desktop installiert und gestartet (WSL 2 Integration aktiviert)
- NVIDIA-GPU mit aktuellem Windows-Treiber (nicht im WSL installieren — Windows-Treiber reicht)
- HuggingFace-Account mit akzeptierter Lizenz für FLUX.1-dev:
  → https://huggingface.co/black-forest-labs/FLUX.1-dev

---

## Schritt 1 — WSL vorbereiten

In WSL (Ubuntu-Terminal):

```bash
# Docker-Gruppe: verhindert "permission denied" Fehler
sudo usermod -aG docker $USER
newgrp docker

# Prüfen ob Docker erreichbar ist
docker ps
```

---

## Schritt 2 — nvidia-container-toolkit installieren (GPU-Zugriff für Docker)

Ohne diesen Schritt läuft Ollama und alle Diffusion-Modelle auf der CPU (10–20x langsamer).

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker

# Docker Desktop neu starten (in Windows), dann prüfen:
docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi
```

Erwartete Ausgabe: Tabellarische Anzeige der RTX 3060 mit VRAM-Info.

---

## Schritt 3 — Repository klonen

```bash
git clone https://github.com/SchoenkeM/odysseus.git
cd odysseus
git checkout mischa-setup
```

---

## Schritt 4 — .env anlegen

```bash
cp .env.example .env
```

Dann `.env` öffnen und folgende Werte setzen:

```env
# Pflicht: Ollama läuft als Docker-Container
OLLAMA_BASE_URL=http://ollama:11434/v1

# Pflicht: Diffusion-Services aktivieren
COMPOSE_FILE=docker-compose.yml:docker/diffusion.yml

# Pflicht: HuggingFace-Token (für FLUX.1-dev und andere gated Modelle)
# Token erstellen unter: https://huggingface.co/settings/tokens
HF_TOKEN=hf_DEIN_TOKEN_HIER
```

Die `.env`-Datei wird von Git ignoriert (steht in `.gitignore`) — niemals committen.

---

## Schritt 5 — Basisservices starten

```bash
docker compose up -d --build
```

Startet: Odysseus UI, Ollama, SearXNG, ChromaDB, ntfy, FLUX.1-schnell

Warten bis alles läuft:
```bash
docker compose ps
```

Alle Services sollten `Up` zeigen. SearXNG zeigt zusätzlich `(healthy)`.

**Web-UI öffnen:** http://localhost:7000

---

## Schritt 6 — Bildgenerierungs-Modelle starten (Profile)

Jedes SDXL-Modell belegt ~7 GB VRAM. Nur ein SDXL-Modell gleichzeitig starten
(oder alle, wenn genug VRAM vorhanden). FLUX-Modelle nutzen CPU-Offload → idle = 0 VRAM.

```bash
# Juggernaut XL v9 (Photorealismus)
docker compose --profile juggernaut up -d

# RealVisXL V4.0 (Szenen, Natur)
docker compose --profile realvis up -d

# FLUX.1-dev (höchste Qualität, braucht HF-Lizenz)
docker compose --profile flux-dev up -d

# Dreamshaper XL (vielseitig, schnell)
docker compose --profile dreamshaper up -d
```

Beim ersten Start werden die Modelle von HuggingFace geladen (je ~7–25 GB).
Fortschritt beobachten:
```bash
docker compose logs -f diffusion-juggernaut
```

Fertig wenn erscheint:
```
INFO:diffusion_server:Model loaded: ...
INFO:     Application startup complete.
```

---

## Schritt 7 — Health-Checks (Modelle bereit?)

```bash
curl http://localhost:8110/health   # FLUX.1-schnell
curl http://localhost:8101/health   # FLUX.1-dev
curl http://localhost:8102/health   # Juggernaut XL
curl http://localhost:8103/health   # RealVisXL
curl http://localhost:8104/health   # Dreamshaper XL
```

Erwartete Antwort: `{"status":"ok","model":"..."}`

---

## Schritt 8 — Odysseus UI konfigurieren

### 8a — Ollama-Modell pullen

```bash
docker exec odysseus-ollama-1 ollama pull llama3.1:8b
```

### 8b — Bildgenerierung einrichten

In der Odysseus UI (http://localhost:7000):

**Admin Panel → Settings → Images**

| Feld | Wert |
|---|---|
| Image Generation Engine | `OpenAI` |
| API Key | `local` (beliebig) |
| Enable Image Generation | ✅ |

Endpunkte eintragen (für jedes aktive Modell):

| Modell | Interne URL (für Odysseus) |
|---|---|
| FLUX.1-schnell | `http://diffusion:8100/v1` |
| FLUX.1-dev | `http://diffusion-dev:8101/v1` |
| Juggernaut XL v9 | `http://diffusion-juggernaut:8102/v1` |
| RealVisXL V4.0 | `http://diffusion-realvis:8103/v1` |
| Dreamshaper XL | `http://diffusion-dreamshaper:8104/v1` |

> **Wichtig:** In den Odysseus-Einstellungen immer die Docker-internen Hostnamen
> (`http://diffusion-juggernaut:8102`) verwenden, nicht `localhost`.
> `localhost` ist nur für eigene curl-Tests vom Windows/WSL-Terminal.

---

## Port-Übersicht

| Service | localhost-Port | Zweck |
|---|---|---|
| Odysseus UI | 7000 | Web-Interface |
| SearXNG | 8080 | Web-Suche |
| ntfy | 8091 | Push-Benachrichtigungen |
| ChromaDB | 8100 | Vektordatenbank |
| FLUX.1-schnell | 8110 | Bildgenerierung (schnell) |
| FLUX.1-dev | 8101 | Bildgenerierung (beste Qualität) |
| Juggernaut XL | 8102 | Bildgenerierung (Photorealismus) |
| RealVisXL | 8103 | Bildgenerierung (Szenen) |
| Dreamshaper XL | 8104 | Bildgenerierung (vielseitig) |

---

## Wichtige Befehle (täglich)

```bash
# Ins Projektverzeichnis
cd ~/pfad/zu/odysseus   # oder wo auch immer du es geklont hast

# Alles starten (inkl. Juggernaut + RealVisXL + FLUX.1-dev)
docker compose --profile juggernaut --profile realvis --profile flux-dev up -d

# Nur Basis starten (ohne optionale Modelle)
docker compose up -d

# Alles stoppen
docker compose --profile juggernaut --profile realvis --profile flux-dev down

# Status aller Container
docker compose ps

# Logs eines Services
docker compose logs -f diffusion-juggernaut

# GPU-Auslastung prüfen
docker exec odysseus-ollama-1 nvidia-smi

# Ollama-Modelle anzeigen
docker exec odysseus-ollama-1 ollama list
```

---

## Verfügbare API-Endpunkte der Diffusion-Server

Alle Modelle unterstützen:

| Endpoint | Methode | Zweck |
|---|---|---|
| `/v1/images/generations` | POST | Text → Bild |
| `/v1/images/img2img` | POST | Bild → Bild (mit Prompt) |
| `/v1/images/inpaint` | POST | Bereich in Bild neu generieren |
| `/v1/images/harmonize` | POST | Farben/Stil angleichen |
| `/v1/models` | GET | Geladenes Modell anzeigen |
| `/health` | GET | Status prüfen |

**img2img Parameter:**
```json
{
  "image": "<base64-PNG>",
  "prompt": "oil painting style",
  "strength": 0.6,
  "negative_prompt": "blurry, low quality",
  "guidance_scale": 7.0,
  "steps": 20
}
```
`strength`: 0.3 = subtile Änderung · 0.75 = starke Überarbeitung · 1.0 = Neuzeichnung

---

## Bekannte Probleme & Lösungen

| Problem | Lösung |
|---|---|
| `permission denied` beim docker-Befehl | `newgrp docker` ausführen |
| Container-Name stimmt nicht | `docker compose ps` → richtigen Namen verwenden |
| Port 8100 belegt (ChromaDB-Konflikt) | FLUX.1-schnell läuft auf Host-Port 8110, nicht 8100 |
| Modell lädt sehr langsam | Normal beim ersten Start — wird im Volume gecacht |
| FLUX.1-dev: 403 Forbidden | HF-Lizenz unter huggingface.co/black-forest-labs/FLUX.1-dev akzeptieren |
| GPU wird nicht erkannt | nvidia-container-toolkit neu installieren, Docker Desktop neu starten |
