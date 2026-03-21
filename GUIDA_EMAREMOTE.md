# EmaRemote — Guida Completa

> Ultima modifica: 16 Marzo 2026
> Questa guida contiene TUTTO quello che serve per capire, mantenere, modificare e ricreare da zero il progetto EmaRemote.

---

## 1. Cos'è EmaRemote

EmaRemote è un fork personalizzato di **RustDesk** (software open source per desktop remoto). È stato modificato per:

- Usare un **server privato dedicato** (non i server pubblici di RustDesk)
- Avere **branding custom** (nome EmaRemote, icone, colori)
- Avere il server **hardcodato nel codice** → l'utente installa e funziona, zero configurazione
- Avere un bottone **"Server Admin Panel"** nelle impostazioni desktop

### Cosa fa in pratica

Permette di controllare un PC dal telefono (e viceversa) via internet, da qualsiasi rete. Il traffico passa tutto attraverso il nostro server privato.

### Piattaforme supportate

- **Windows** — file `.exe` installabile, scaricabile dal sito
- **Android** — file `.apk` sideloaded (non è sul Play Store), scaricabile dal sito
- **Mac** — file `.dmg` per Apple Silicon e Intel, scaricabile dal sito
- **Web Client** — connessione da browser (funzionalità RustDesk nativa)

---

## 2. Architettura Completa

```
┌──────────────────────────────────────────────────────────────────┐
│                    SERVER HETZNER (Linux)                        │
│                    IP:  91.98.66.127                             │
│                    SSH: root / Swamyburgher13                    │
│                                                                  │
│  DOCKER COMPOSE: /srv/apps/rustdesk/docker-compose.yml          │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────┐   │
│  │     hbbs       │  │     hbbr       │  │  rustdesk-api    │   │
│  │  (rendezvous)  │  │    (relay)     │  │  (admin panel)   │   │
│  │                │  │                │  │                  │   │
│  │ TCP 21115      │  │ TCP 21117      │  │ TCP 21114        │   │
│  │ TCP 21116      │  │ TCP 21119 (ws) │  │                  │   │
│  │ UDP 21116      │  │                │  │                  │   │
│  │ TCP 21118 (ws) │  │                │  │                  │   │
│  └────────────────┘  └────────────────┘  └──────────────────┘   │
│                                                                  │
│  CHIAVI CRITTOGRAFICHE: /srv/apps/rustdesk/data/                │
│  ├── id_ed25519       (chiave privata - NON condividere)        │
│  ├── id_ed25519.pub   (chiave pubblica - hardcodata nell'app)   │
│  └── db_v2.sqlite3    (database peer registrati)                │
│                                                                  │
│  NGINX: /etc/nginx/sites-enabled/emaremote                      │
│  ├── eremote.emaandema.com:443  →  /srv/apps/rustdesk-landing/  │
│  ├── /_admin/                   →  proxy http://127.0.0.1:21114 │
│  └── /api/                      →  proxy http://127.0.0.1:21114 │
│  SSL: Let's Encrypt via certbot                                  │
│                                                                  │
│  LANDING PAGE: /srv/apps/rustdesk-landing/                       │
│  ├── dist/index.html       (pagina principale con tutorial)      │
│  └── downloads/                                                  │
│      ├── emaremote-windows.exe                                   │
│      ├── emaremote-android.apk                                   │
│      ├── emaremote-mac-apple.dmg                                 │
│      └── emaremote-mac-intel.dmg                                 │
│                                                                  │
│  FIREWALL (UFW):                                                 │
│  22/tcp  80/tcp  443  21115/tcp  21116/tcp  21116/udp            │
│  21117/tcp  21118/tcp  21119/tcp                                 │
└──────────────────────────────────────────────────────────────────┘

                    INTERNET
                       │
        ┌──────────────┼──────────────┐
        │              │              │
  ┌─────┴─────┐  ┌────┴────┐  ┌─────┴─────┐
  │ PC Win    │  │ Android │  │ Browser   │
  │ .exe      │  │ .apk    │  │ web client│
  │           │  │         │  │           │
  │ ID: XXXXX │  │ ID: YYY │  │           │
  └───────────┘  └─────────┘  └───────────┘

  Ogni dispositivo ha un ID numerico assegnato da hbbs.
  Per connettersi, inserisci l'ID dell'altro dispositivo.
```

### DNS (Cloudflare)

- Dominio: `emaandema.com` (gestito su Cloudflare)
- Record A: `eremote` → `91.98.66.127` (proxy Cloudflare **OFF**)

---

## 3. Come Funziona la Connessione (in dettaglio)

1. L'app si avvia → legge il server hardcodato `91.98.66.127` da `config.rs`
2. Invia un heartbeat UDP a `hbbs` sulla porta `21116` → hbbs assegna un ID numerico
3. L'utente vede il suo ID nell'app (es. `272212130`)
4. Per connettersi a un altro dispositivo, inserisce il suo ID
5. L'app chiede a `hbbs`: "dov'è l'ID 272212130?"
6. `hbbs` risponde con l'indirizzo IP del dispositivo
7. L'app tenta una connessione P2P diretta (punch hole NAT)
8. **Se P2P fallisce** (quasi sempre, perché il NAT è ASYMMETRIC) → la connessione passa attraverso `hbbr` (relay) sulla porta `21117`
9. Viene chiesta la **password** impostata sul dispositivo remoto
10. Connessione stabilita → si vede lo schermo remoto

### Perché ALWAYS_USE_RELAY=Y

L'env var `ALWAYS_USE_RELAY=Y` nel docker-compose dice a `hbbs` di forzare SEMPRE il relay, saltando il tentativo P2P. Questo è necessario perché il NAT della connessione è ASYMMETRIC e il P2P fallisce ogni volta, rallentando la connessione per niente.

### Autenticazione: la chiave pubblica

Quando l'app si connette a `hbbs`, il server presenta la sua chiave pubblica. L'app la confronta con `RS_PUB_KEY` hardcodato nel codice. Se non corrisponde → "Failed to secure tcp: deadline has elapsed".

**Chiave pubblica attuale**: `Xxpl19Iv4q+JBFUzdGUtEGddvR+9Y7P6BOrBoJHAsWE=`

---

## 4. Codice Sorgente

### Repository GitHub

| Repo | URL | Descrizione |
| ---- | --- | ----------- |
| emaremote | `github.com/eleunameotarepmi-create/emaremote` | Fork di RustDesk con branding e config custom |
| hbb_common | `github.com/eleunameotarepmi-create/hbb_common` | Submodule con server + chiave hardcodati |

### Workspace locale

```
c:\Users\Urukk\.gemini\antigravity\scratch\EmaRemote\
```

### Clonare il progetto

```bash
git clone --recurse-submodules https://github.com/eleunameotarepmi-create/emaremote.git
```

### File modificati rispetto a RustDesk vanilla

#### 1. `libs/hbb_common/src/config.rs` (nel submodule `hbb_common`)

Questo è il file PIÙ IMPORTANTE. Contiene il server e la chiave hardcodati:

```rust
pub const RENDEZVOUS_SERVERS: &[&str] = &["91.98.66.127"];
pub const RS_PUB_KEY: &str = "Xxpl19Iv4q+JBFUzdGUtEGddvR+9Y7P6BOrBoJHAsWE=";
```

- `RENDEZVOUS_SERVERS` → indirizzo IP del server hbbs
- `RS_PUB_KEY` → chiave pubblica del server (deve corrispondere a `/srv/apps/rustdesk/data/id_ed25519.pub`)

**ATTENZIONE**: questo file è nel submodule `hbb_common`, quindi ha un suo repo separato (`eleunameotarepmi-create/hbb_common`). Per modificarlo:

```bash
cd libs/hbb_common
# modifica src/config.rs
git add . && git commit -m "update" && git push
cd ../..
git add libs/hbb_common
git commit -m "update submodule" && git push
```

#### 2. `flutter/lib/desktop/pages/desktop_setting_page.dart`

Aggiunto bottone **"Server Admin Panel"** nella sezione Network delle impostazioni desktop. Apre `https://eremote.emaandema.com/_admin/` nel browser.

#### 3. `.github/workflows/emaremote-windows-build.yml`

Workflow GitHub Actions per compilare l'EXE Windows. Rinomina `rustdesk.exe` → `emaremote.exe`.

#### 4. `.gitmodules`

```
[submodule "libs/hbb_common"]
    path = libs/hbb_common
    url = https://github.com/eleunameotarepmi-create/hbb_common.git
```

---

## 5. Server — Docker Compose

### File: `/srv/apps/rustdesk/docker-compose.yml`

```yaml
services:
  hbbs:
    container_name: hbbs
    image: rustdesk/rustdesk-server:latest
    command: hbbs -r 91.98.66.127:21117
    environment:
      - ALWAYS_USE_RELAY=Y
    volumes:
      - ./data:/root
    network_mode: host
    depends_on:
      - hbbr
    restart: unless-stopped

  hbbr:
    container_name: hbbr
    image: rustdesk/rustdesk-server:latest
    command: hbbr
    volumes:
      - ./data:/root
    network_mode: host
    restart: unless-stopped

  rustdesk-api:
    container_name: rustdesk-api
    image: lejianwen/rustdesk-api:latest
    environment:
      - TZ=Europe/Rome
    ports:
      - "21114:21114"
    volumes:
      - ./api-data:/app/data
    restart: unless-stopped
```

### Spiegazione dei container

| Container | Cosa fa | Porte |
| --------- | ------- | ----- |
| **hbbs** | Server rendezvous. Registra i peer (dispositivi), gestisce le richieste `"dov'è questo ID?"`, assegna gli ID, verifica le chiavi | 21115 (NAT test), 21116 TCP+UDP (rendezvous), 21118 (websocket) |
| **hbbr** | Server relay. Inoltra il traffico video/input quando P2P non funziona | 21117 (relay), 21119 (websocket relay) |
| **rustdesk-api** | Pannello admin web. Gestione utenti, log, statistiche | 21114 |

### Comandi utili sul server

```bash
# Accedere al server
ssh root@91.98.66.127
# password: Swamyburgher13

# Stato container
cd /srv/apps/rustdesk
docker compose ps

# Riavviare tutto
docker compose down && docker compose up -d

# Vedere i log di hbbs
docker logs hbbs --tail 50

# Vedere i log di hbbr
docker logs hbbr --tail 50

# Vedere la chiave pubblica
cat /srv/apps/rustdesk/data/id_ed25519.pub

# Verificare le porte aperte
ss -tlnp | grep -E '2111[4-9]'
```

---

## 6. Server — Nginx

### File: `/etc/nginx/sites-enabled/emaremote`

La configurazione Nginx fa tre cose:
1. Serve la landing page statica (HTML + download)
2. Fa proxy verso il pannello admin (`/_admin/`)
3. Fa proxy verso l'API (`/api/`)

Il certificato SSL è gestito da **certbot** (Let's Encrypt) automaticamente.

### Comandi Nginx

```bash
# Ricaricare la config
nginx -s reload

# Testare la config
nginx -t

# Rinnovare certificato SSL
certbot renew

# Vedere lo stato
systemctl status nginx
```

---

## 7. Build — Come Compilare l'App

### Build Windows (EXE)

1. Vai su `github.com/eleunameotarepmi-create/emaremote`
2. Clicca **"Actions"** nella barra in alto
3. Nella sidebar sinistra, seleziona **"EmaRemote Windows Build"**
4. Clicca **"Run workflow"** → scegli branch `master` → clicca il bottone verde
5. Aspetta **~30 minuti**
6. Quando è verde ✅, clicca sul run
7. In basso nella pagina, sotto **"Artifacts"**, scarica il file `.exe`

### Build Android (APK)

1. Stessa cosa ma seleziona **"EmaRemote Android Build"**
2. Scarica l'artifact APK

### Build Mac (DMG)

1. Stessa cosa ma seleziona il workflow Mac
2. Ci sono due artifact: uno per Apple Silicon, uno per Intel

---

## 8. Deploy — Come Caricare le Build sul Server

### Prerequisiti Python

```bash
pip install paramiko
```

Paramiko è una libreria Python per SSH/SFTP. La usiamo per trasferire file al server senza dover usare la command line.

### Script Deploy EXE Windows

```python
import paramiko

ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect('91.98.66.127', username='root', password='Swamyburgher13',
            timeout=30, allow_agent=False, look_for_keys=False)
sftp = ssh.open_sftp()

# Cambia il percorso con il file scaricato
sftp.put(r'C:\Users\Urukk\Downloads\EmaRemote.exe',
         '/srv/apps/rustdesk-landing/downloads/emaremote-windows.exe')

sftp.close()
ssh.close()
print('EXE deployato!')
```

### Script Deploy APK Android

```python
import paramiko

ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect('91.98.66.127', username='root', password='Swamyburgher13',
            timeout=30, allow_agent=False, look_for_keys=False)
sftp = ssh.open_sftp()

sftp.put(r'C:\Users\Urukk\Downloads\app-release.apk',
         '/srv/apps/rustdesk-landing/downloads/emaremote-android.apk')

sftp.close()
ssh.close()
print('APK deployato!')
```

### Script Deploy Landing Page

```python
import paramiko

ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect('91.98.66.127', username='root', password='Swamyburgher13',
            timeout=30, allow_agent=False, look_for_keys=False)
sftp = ssh.open_sftp()

sftp.put(r'C:\percorso\al\file\index.html',
         '/srv/apps/rustdesk-landing/dist/index.html')

sftp.close()
ssh.close()
print('Landing page deployata!')
```

### Percorsi file sul server (dove vanno le build)

| Cosa | Percorso sul server |
| ---- | ------------------- |
| EXE Windows | `/srv/apps/rustdesk-landing/downloads/emaremote-windows.exe` |
| APK Android | `/srv/apps/rustdesk-landing/downloads/emaremote-android.apk` |
| DMG Mac Apple | `/srv/apps/rustdesk-landing/downloads/emaremote-mac-apple.dmg` |
| DMG Mac Intel | `/srv/apps/rustdesk-landing/downloads/emaremote-mac-intel.dmg` |
| Landing Page | `/srv/apps/rustdesk-landing/dist/index.html` |
| Docker Compose | `/srv/apps/rustdesk/docker-compose.yml` |
| Chiavi | `/srv/apps/rustdesk/data/id_ed25519*` |
| Nginx config | `/etc/nginx/sites-enabled/emaremote` |

---

## 9. Landing Page / Sito Web

URL: **https://eremote.emaandema.com**

È un singolo file HTML (`/srv/apps/rustdesk-landing/dist/index.html`) che contiene:

- Hero section con branding EmaRemote
- 4 card download (Windows, Mac, Android, Web Connect)
- Sezione Tutorial (installazione + utilizzo + FAQ)
- Footer

Stile: design dark con glassmorphism, font Outfit, icone Lucide, animazioni CSS.

Il **Web Connect** (`/web/`) usa il client web di RustDesk (una build separata già presente sul server).

### Pannello Admin

URL: **https://eremote.emaandema.com/_admin/**

Pannello web per gestire utenti, peer registrati, log e statistiche. Funziona tramite `rustdesk-api` (container Docker).

---

## 10. Problemi Noti e Soluzioni

### "Failed to secure tcp: deadline has elapsed"

**Causa**: L'utente ha scritto a mano l'indirizzo server o la chiave pubblica nei campi delle impostazioni dell'app. Anche se sembra corretto, la formattazione è diversa da quella hardcodata e il match fallisce.

**Soluzione**: Andare in Impostazioni → ID/Relay Server e **svuotare TUTTI i campi**. Lasciare tutto vuoto. I valori hardcodati in `config.rs` funzionano automaticamente quando i campi sono vuoti.

### Il servizio Windows non si collega al server

**Causa**: Il servizio Windows gira come utente `LocalService`, NON come l'utente `Urukk`. Legge la config da:

```
C:\Windows\ServiceProfiles\LocalService\AppData\Roaming\EmaRemote\config\
```

Se quella cartella è vuota o ha config vecchia, il servizio non sa quale server usare.

**Soluzione** (da eseguire come Amministratore):

```powershell
Copy-Item "$env:APPDATA\EmaRemote\config\*" "C:\Windows\ServiceProfiles\LocalService\AppData\Roaming\EmaRemote\config\" -Force
Restart-Service -Name "EmaRemote" -Force
```

### Android — Autorizzazioni negate / "Impostazioni con restrizioni"

**Causa**: Android 13+ blocca le autorizzazioni speciali (Accessibilità, Screen Capture, Overlay) per app installate da APK fuori dal Play Store.

**Soluzione**:

1. Impostazioni → App → EmaRemote
2. Tocca i **tre puntini (⋮)** in alto a destra
3. Tocca **"Consenti impostazioni con restrizioni"**
4. Conferma con PIN/impronta

Poi abilita:

- **Accessibilità**: Impostazioni → Accessibilità → EmaRemote → Attiva
- **Mostra sopra altre app**: Impostazioni → App → EmaRemote → Overlay → Attiva
- **Batteria**: Impostazioni → App → EmaRemote → Batteria → Nessuna restrizione

### Connessione lenta o che cade

Se la connessione è lenta, verificare:

1. Che `ALWAYS_USE_RELAY=Y` sia nel docker-compose
2. Che `hbbr` sia in esecuzione: `docker logs hbbr --tail 5`
3. Che le porte siano aperte: `ufw status`

---

## 11. Ricreare TUTTO da Zero

Se il server muore, il repo sparisce, e devi rifare tutto.

### Passo 1: Server

1. Prendi un server Linux (Hetzner, qualsiasi provider)
2. Installa Docker e Docker Compose
3. Crea la struttura:

```bash
mkdir -p /srv/apps/rustdesk/data
mkdir -p /srv/apps/rustdesk/api-data
mkdir -p /srv/apps/rustdesk-landing/dist
mkdir -p /srv/apps/rustdesk-landing/downloads
```

4. Se hai il backup delle chiavi (`01-CHIAVI-SERVER/`), copiale:

```bash
# PRIMA di avviare Docker!
scp id_ed25519 id_ed25519.pub root@NUOVO_IP:/srv/apps/rustdesk/data/
```

5. Crea il `docker-compose.yml` (copialo dalla sezione 5 sopra)
6. Avvia:

```bash
cd /srv/apps/rustdesk
docker compose up -d
```

Se NON hai le chiavi, Docker ne genera di nuove al primo avvio. In quel caso vedi Passo 3.

### Passo 2: Nginx + SSL

```bash
apt install nginx certbot python3-certbot-nginx

# Crea il file config copiando da 02-CONFIG-SERVER/nginx-emaremote.conf
cp nginx-emaremote.conf /etc/nginx/sites-enabled/emaremote

# Ottieni il certificato SSL
certbot --nginx -d eremote.emaandema.com

# Ricarica
nginx -s reload
```

### Passo 3: Aggiornare le chiavi nell'app (solo se sono cambiate)

Se il server ha generato nuove chiavi:

```bash
# Sul server, leggi la nuova chiave pubblica
cat /srv/apps/rustdesk/data/id_ed25519.pub
# Output: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=
```

Poi nel codice:

1. Modifica `libs/hbb_common/src/config.rs` → aggiorna `RS_PUB_KEY` con la nuova chiave
2. Se l'IP è cambiato → aggiorna anche `RENDEZVOUS_SERVERS`
3. Commit e push del submodule + repo principale
4. Rifare TUTTE le build (Windows + Android + Mac)

### Passo 4: DNS

Su Cloudflare:

- A record: `eremote` → `NUOVO_IP` (proxy OFF, solo DNS)

### Passo 5: Landing Page

Copia la landing page dal backup (`02-CONFIG-SERVER/landing-page/`) o dal repo:

```bash
scp index.html root@NUOVO_IP:/srv/apps/rustdesk-landing/dist/
```

### Passo 6: Build e Deploy

Segui le sezioni 7 e 8 di questa guida.

### Passo 7: Firewall

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443
ufw allow 21115/tcp
ufw allow 21116/tcp
ufw allow 21116/udp
ufw allow 21117/tcp
ufw allow 21118/tcp
ufw allow 21119/tcp
ufw enable
```

---

## 12. Credenziali e Account

| Servizio | Utente | Password/Note |
| -------- | ------ | ------------- |
| Server SSH | `root` @ `91.98.66.127` | `Swamyburgher13` |
| GitHub | `eleunameotarepmi-create` | account personale |
| Cloudflare | — | dominio `emaandema.com` |
| Admin Panel | `https://eremote.emaandema.com/_admin/` | credenziali impostate nel pannello |
| Chiave pubblica server | — | `Xxpl19Iv4q+JBFUzdGUtEGddvR+9Y7P6BOrBoJHAsWE=` |

---

## 13. Struttura File nel Workspace

```
EmaRemote/
├── libs/
│   └── hbb_common/           ← SUBMODULE (repo separato!)
│       └── src/
│           └── config.rs      ← SERVER + CHIAVE HARDCODATI QUI
├── flutter/
│   └── lib/
│       └── desktop/
│           └── pages/
│               └── desktop_setting_page.dart  ← bottone Admin Panel
├── src/                       ← codice Rust del client
├── .github/
│   └── workflows/             ← build workflows GitHub Actions
├── .gitmodules                ← punta a hbb_common fork
├── Cargo.toml                 ← dipendenze Rust
├── build.py                   ← script build (usato dai workflow)
├── GUIDA_EMAREMOTE.md         ← QUESTA GUIDA
└── immondizia/                ← file temp, vecchie build, script debug
```

---

## 14. Backup nel OneDrive

Tutto il backup è in:

```
C:\Surface Shares\OneDrive - MSFT\Documenti\LAVORO\Schede vini\APP-EMA\Emaremote\
```

| Cartella | Contenuto |
| -------- | --------- |
| `01-CHIAVI-SERVER/` | `id_ed25519` + `id_ed25519.pub` — LE PIÙ IMPORTANTI. Senza queste devi rifare tutte le build |
| `02-CONFIG-SERVER/` | `docker-compose.yml`, `nginx-emaremote.conf`, `landing-page/` |
| `03-BUILD-CORRENTI/` | EXE, APK, DMG funzionanti. Se GitHub Actions ha problemi, hai queste come fallback |
| `04-DOCUMENTAZIONE/` | Questa guida + Tutorial utente HTML |
| `05-CODICE-SORGENTE/` | README con link ai repo GitHub |

---

## 15. Strumenti di Sviluppo

### Python + Paramiko

Per accedere al server da script Python (deploy, backup, diagnostica):

```bash
pip install paramiko
```

Esempio base di connessione:

```python
import paramiko

ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect('91.98.66.127', username='root', password='Swamyburgher13',
            timeout=30, allow_agent=False, look_for_keys=False)

# Eseguire comandi
_, out, _ = ssh.exec_command('docker ps')
print(out.read().decode())

# Trasferire file (SFTP)
sftp = ssh.open_sftp()
sftp.put('file_locale.exe', '/percorso/remoto/file.exe')  # upload
sftp.get('/percorso/remoto/file.exe', 'file_locale.exe')   # download
sftp.close()

ssh.close()
```

### GitHub Actions

Le build vengono fatte da GitHub Actions (CI/CD gratuito). I workflow sono in `.github/workflows/`. Non serve avere Rust installato localmente.

### Antigravity (questo agente AI)

L'agente AI usato per tutto lo sviluppo. Il workspace è in `c:\Users\Urukk\.gemini\antigravity\scratch\EmaRemote\`. Tutte le conversazioni e gli script di debug sono salvati nella storia delle conversazioni.
