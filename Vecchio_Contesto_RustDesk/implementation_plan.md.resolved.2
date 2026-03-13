# EmaRemote — Piano Completo

Deploy del server RustDesk + build APK personalizzato + landing page su `rustdesk.emaandema.com`.

> [!CAUTION]
> **Ianua Vini (`/srv/apps/ianua-vini/`) non viene toccata in nessun modo.** Tutto va in directory separate, container separati, porte separate.

## User Review Required

> [!IMPORTANT]
> **DNS Cloudflare**: Serve un record A `rustdesk` → `91.98.66.127` con proxy **DNS only** (grigio). Lo fai tu su Cloudflare o ti guido passo passo?

---

## Fase 1 — RustDesk Server (Docker)

Dove: `/srv/apps/rustdesk/`

- Creo `docker-compose.yml` con `hbbs` + `hbbr` (`network_mode: host`)
- Apro porte UFW: 21115-21119
- Avvio container
- Recupero chiave pubblica generata automaticamente

**Impatto su Ianua Vini: ZERO**

---

## Fase 2 — Build APK "EmaRemote"

Dove: `/tmp/emaremote-build/` (temporaneo, cancellato dopo il build)

### Cosa cambia dal RustDesk originale

| File | Modifica |
|------|----------|
| `strings.xml` | Nome app: `RustDesk` → `EmaRemote` |
| `build.gradle` | Package: `com.carriez.flutter_hbb` → `com.emaandema.emaremote` |
| `AndroidManifest.xml` | Label: `EmaRemote` |
| Icone `res/mipmap-*` | Icona custom EmaRemote (generata) |
| Config server | IP `91.98.66.127` + chiave pubblica preconfigurati |

### Processo build (tutto su Hetzner)

1. Installo build tools in `/tmp/`: Rust, Flutter 3.24.5, Android SDK/NDK, cargo-ndk
2. Clono RustDesk da GitHub
3. Applico le modifiche di branding
4. Compilo: `flutter build apk --release`
5. Risultato: `emaremote-1.0.0-universal.apk` (~70MB)
6. Copio APK in `/srv/apps/rustdesk-landing/downloads/`
7. **Cancello tutto `/tmp/emaremote-build/`** → spazio recuperato

**Impatto su Ianua Vini: ZERO** (tutto in /tmp/)

---

## Fase 3 — Landing Page

Dove: `/srv/apps/rustdesk-landing/`

Pagina HTML statica premium:
- Hero con logo EmaRemote + nome
- Card download: Android (APK dal server), Windows/Mac (link GitHub), iOS (App Store)
- Sezione "Come configurare" con IP server e chiave pre-compilati
- Dark theme, glassmorphism, micro-animazioni

**Impatto su Ianua Vini: ZERO**

---

## Fase 4 — Nginx + SSL

Dove: `/etc/nginx/sites-available/emaremote`

- `server_name rustdesk.emaandema.com`
- Root: `/srv/apps/rustdesk-landing/dist/`
- Location `/downloads/` per servire APK/EXE/DMG
- SSL via Certbot
- **Non modifica il file `ianuavini`** — è un file Nginx separato

**Impatto su Ianua Vini: ZERO**

---

## Struttura finale server

```
/srv/apps/
  ├── ianua-vini/              ← INTATTO, non toccato
  ├── rustdesk/                ← Server RustDesk (Docker)
  │     ├── docker-compose.yml
  │     └── data/
  └── rustdesk-landing/        ← Landing page + downloads
        ├── dist/
        │   ├── index.html
        │   └── style.css
        └── downloads/
            └── emaremote-1.0.0-universal.apk
```

---

## Verification Plan

1. `docker ps --filter name=hbb` → hbbs e hbbr `Up`
2. `curl https://rustdesk.emaandema.com` → landing page `200`
3. Download APK → file si scarica correttamente
4. Installo APK su Android → app si apre con branding "EmaRemote"
5. `curl https://ianuavini.emaandema.com` → **ancora funzionante, 200**

---

## Fase 5 — Sviluppo App Custom "EmaRemote" (Hard Fork)

**Obiettivo:** Il cliente richiede un'app completamente sua ("mia che mi fai tu") basata sul motore RustDesk, ma senza i blocchi imposti dall'API server ufficiale (es. impossibilità di cambiare ID dall'interfaccia senza licenza Pro). L'app deve nascere col nome EmaRemote, l'icona personalizzata e i parametri del server (IP `91.98.66.127` e Public Key) già cablati all'interno.

**Fattibilità e Approccio Tecnico:**
Scrivere da zero un'app Desktop/Mobile che usa le librerie native C++/Rust di RustDesk (tramite FFI) richiederebbe mesi di sviluppo a causa della complessità del protocollo di codifica video. 
L'approccio industriale corretto e realistico è quello di eseguire un **"Hard Fork" del repository ufficiale di RustDesk** e modificarne minuziosamente il codice sorgente del client (scritto in Dart/Flutter e Rust).

**Modifiche architetturali al codice sorgente:**
1. **Sblocco nativo "Cambia ID":** Intercetteremo la funzione nel codice frontend (Dart) che disabilita il pulsante "Cambia ID" in assenza di convalida dal finto server API. Toglieremo la validazione cloud e faremo in modo che il salvataggio modifichi direttamente il file di configurazione locale (`RustDesk.toml`).
2. **Rimozione Bloatware & Upselling:** Rimuoveremo programmaticamente i controlli dell'interfaccia utente (UI) per la "Console", il tab "Account" (che ora non servirà più), e i fastidiosi inviti a "Comprare RustDesk Pro". L'interfaccia diventerà minimale.
3. **Hardcoding Server & Branding:** Applicheremo al codice sorgente (`strings.xml`, `build.gradle`, e asset vari) il logo EmaRemote. Inietteremo nel file di configurazione base di RustDesk (`config.rs`) l'IP del tuo server Hetzner e la Public Key associata crittograficamente per l'End-to-End Encryption.

**Processo di Compilazione (Build Pipeline):**
Nel log precedente abbiamo tentato e fallito la compilazione sul server Hetzner a causa di crash nel download delle librerie C++ native (vcpkg per Android/Windows).
Per risolvere questo problema, l'intero processo di compilazione verrà demandato al Cloud tramite **GitHub Actions**:
1. Creeremo un repository sul tuo account GitHub contenente il codice sorgente modificato.
2. Sfrutteremo l'infrastruttura di GitHub per avviare dei worker (macchine virtuali) su Windows, MacOS e Linux che compileranno nativamente i vari client.
3. I file compilati (`.apk`, `.exe`, `.dmg`) verranno scaricati e caricati sulla tua Landing Page (`rustdesk.emaandema.com/downloads`).
