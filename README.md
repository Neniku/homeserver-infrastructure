# 🚀 Ultimate Home Server (NAS) Infrastructure

Benvenuto nella repository che documenta l'infrastruttura completa del mio Home Server. Questo progetto nasce dall'esigenza di consolidare svariati servizi essenziali (lavorativi e di intrattenimento) in un'unica macchina, mantenendo però un'ossessione rigorosa per la **sicurezza**, l'**isolamento dei dati**, l'**automazione** e la **ridondanza**.

---

## 🎯 Filosofia ed Esigenze di Progetto

Quando ho progettato questo server, avevo obiettivi e paletti molto chiari:

1. **Nessuna porta esposta su Internet**: Aprire porte sul router domestico espone la rete a botnet e scanner automatici. L'intero server è raggiungibile solo tramite VPN mesh (Zero Trust).
2. **Protezione dei dati aziendali**: Il server ospita l'infrastruttura di gestione di un'attività commerciale, con dati sensibili di magazzino, clienti e fatture. Tali dati si trovano su un network Docker inaccessibile perfino agli altri servizi del server.
3. **Prevenzione dei disastri fisici**: I dischi meccanici esterni possono scollegarsi. Se un software di download multimediale si avvia senza il disco target, scriverà sul disco di sistema (root OS), riempiendolo e facendo crashare il server. Serviva un failsafe meccanico.
4. **Monitoraggio Proattivo**: Volevo essere avvisato sul telefono *prima* che un problema diventasse critico (es. temperature alte, container bloccati o backup non eseguiti da giorni).

---

## 🛡️ Architettura di Rete e Sicurezza (Tailscale)

La sicurezza perimetrale è gestita interamente tramite **Tailscale** (basato su WireGuard). Il NAS opera come nodo di un'architettura **Zero Trust**.

- I miei dispositivi (PC, smartphone, Android TV, laptop) fanno parte della stessa Tailnet.
- **Risultato**: Posso accedere a servizi come Jellyfin o il Gestionale ovunque mi trovi nel mondo, a piena velocità, senza aver aperto alcuna porta sul router.

### Reverse Proxy e Socket Security

- I servizi non sono accessibili liberamente: passano tutti per **NGINX**.
- Per evitare di montare `/var/run/docker.sock` direttamente nella dashboard (pratica rischiosa che consentirebbe alla dashboard di prendere il controllo di root se hackerata), ho interposto un **`docker-socket-proxy`**. Esso filtra le richieste HTTP garantendo l'accesso in **sola lettura** alle API di Docker.

---

## 💾 Gestione dello Storage e "Fail-Safe" Mount

Il server gestisce svariati terabyte di storage, divisi in compartimenti logici:

| Mount Point   | Dimensione  | Tipo      | Uso                                          |
|---------------|-------------|-----------|----------------------------------------------|
| `/`           | 110 GB      | SSD       | OS Debian, configurazione Docker, database   |
| `/data_film`  | 3.6 TB      | HDD       | Libreria film                                |
| `/data_serie` | 3.6 TB      | HDD       | Libreria serie TV                            |
| `/cache`      | 293 GB      | SSD       | Caching download e transcoding video         |
| `/backup`     | ~460 GB     | SSD/HDD   | Destinazione backup notturni cifrati         |

### Il meccanismo del `.mounted` file

All'interno della radice dei dischi fisici multimediali ho creato un file fittizio `.mounted`. Nei `docker-compose.yml` di Sonarr, Radarr e qBittorrent, ho forzato il binding di questo file con la direttiva `create_host_path: false`.

> **Perché?** Se il disco non viene riconosciuto da Linux al boot, la cartella (es. `/data_film`) sarà vuota e non conterrà `.mounted`. Docker non avvierà il container, **impedendo fisicamente** che Radarr riempia il disco di sistema.

---

## 📦 Stack dei Servizi (Docker Compose)

Tutti i servizi sono modulari, ospitati in `/opt/appdata/`, per garantire portabilità e isolamento.

### 1. 🗞️ Gestionale Aziendale

Stack sviluppato interamente custom.

- **PostgreSQL**: Gira su una rete `db-net` **isolata**, non collegata a bridge esterni. Il container è blindato (filesystem read-only, tmpfs per le scritture temporanee, `no-new-privileges`).
- **Backend (Python FastAPI)**: Gestisce anagrafiche, vendite e include un parser avanzato per file PDF (fatture/bolle fornitori).
- **Frontend (Next.js)**: Dashboard ultra-reattiva.

> **Perché isolato?** Se un servizio come qBittorrent venisse compromesso tramite una vulnerabilità zero-day, l'attaccante non potrebbe ruotare verso il database aziendale, essendo fisicamente su un network Docker non intersecante.

### 2. 🎬 Media Server (L'Ecosistema *arr)

Automatizza il reperimento, la conversione e la distribuzione di contenuti multimediali.

| Servizio         | Funzione                                                                 |
|------------------|--------------------------------------------------------------------------|
| **Jellyfin**     | Hub di riproduzione con hardware transcoding (Intel QuickSync / VA-API) |
| **Tdarr**        | Conversione automatica in H.265/HEVC per risparmio di spazio            |
| **Sonarr/Radarr**| Gestione e organizzazione della libreria                                 |
| **Prowlarr**     | Aggregatore di indexer                                                   |
| **Jackett**      | Proxy per tracker                                                        |
| **FlareSolverr** | Bypass dei controlli Cloudflare anti-bot                                 |
| **Jellyseerr**   | Portale "Netflix-style" per le richieste di contenuto                   |
| **qBittorrent**  | Client di download                                                       |

### 3. 🔐 Password Manager

- **Vaultwarden**: Self-hosted Bitwarden-compatible password manager.

> Hostarsi le password localmente elimina il rischio di furti da data-breach aziendali (stile LastPass). Configurato con SMTP per l'invio di token ai dispositivi.

### 4. 🎛️ Amministrazione, Telemetria e Alerter Custom

Monitoraggio su due livelli:

- **Beszel + Agent**: Grafici storici su CPU, IOPS dei dischi, traffico rete e RAM.
- **Docker-Alerter (Custom Python Bot)**: Bot Telegram scritto in Python e containerizzato che:
  - Notifica istantaneamente se un container vitale (DB, NGINX) si arresta.
  - Verifica i syslog dell'host (`/host/log/auth.log`) cercando pattern di brute-force SSH (alert dopo 5 tentativi falliti).
  - Interroga `/sys` e `/proc` per segnalare surriscaldamenti anomali.
  - Limitato a `0.5 CPU` e `256 MB RAM` per non diventare esso stesso causa di rallentamenti.

---

## 🛟 Strategia di Backup

I backup sono delegati nativamente al sistema operativo Linux tramite **Systemd Timers** (più affidabili di `cron`), senza dipendere dal demone Docker.

- **`vaultwarden-backup.service`**: Dump notturno cifrato → `/backup/vaultwarden`
- **`edicola87-backup.service`**: Dump notturno cifrato → `/backup/edicola`

### Backup Staleness Detection

Il bot Telegram controlla ciclicamente le cartelle di backup. Se non trova file con data di creazione inferiore a **48 ore**, invia un avviso critico:

```
⚠️ ATTENZIONE: Nessun backup recente trovato!
Ultimo backup: [timestamp]
```

Questo garantisce che un backup silenziosamente interrotto venga rilevato entro due giorni.

---

## 🔒 Note di Sicurezza

> **Nessun IP, credenziale, token o porta è presente in questa repository.** Tutte le variabili sensibili sono gestite tramite file `.env` non versionati (esclusi da `.gitignore`) o secret manager.

---

## 📄 Licenza

[MIT](LICENSE)
