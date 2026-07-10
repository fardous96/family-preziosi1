# Avanzamento Produzione

Applicazione web per gestire l'avanzamento della produzione tramite **schede di lavorazione con codici a barre**.
Il capo crea gli ordini, i dipendenti registrano ore (start/stop) e quantità prodotte tramite badge, con storico completo, report e uno schermo per la sala d'attesa.

- **Frontend:** React (servito da Nginx)
- **Backend:** FastAPI (Python)
- **Database:** MongoDB
- **File (logo/foto/video):** salvati su filesystem locale (nessun servizio esterno)

> Applicazione **completamente autonoma**: non richiede chiavi API né servizi esterni.

---

## Funzionalità principali

- Creazione ordini e generazione automatica **scheda di lavorazione** con codice a barre univoco (PDF stampabile con logo).
- **Anagrafiche** clienti e merce riutilizzabili.
- Gestione **badge dipendenti** (PDF) e flag straordinario per dipendente.
- **Postazione Officina**: scansione badge + scheda per START/STOP lavorazioni (multi-scheda), timbratura entrata/uscita presenze.
- **Chiusura automatica** a fine giornata di schede e presenze dimenticate (orario configurabile).
- **Dashboard** con stato schede, attività in tempo reale e grafici.
- **Report** ore/pezzi per scheda, dipendente e periodo + **Presenze** con export **Excel/CSV** per il commercialista.
- **Schermo Sala d'Attesa** (`/display`): dati produzione + slideshow foto/video + logo + banner scorrevole.

---

## Requisiti

### Opzione A — Docker (consigliata)
- [Docker](https://docs.docker.com/get-docker/) e [Docker Compose](https://docs.docker.com/compose/) (Docker Desktop li include già).

### Opzione B — Esecuzione manuale
- Python 3.11+
- Node.js 20+ e Yarn (`npm install -g yarn`)
- MongoDB 6/7 in esecuzione localmente

---

## 1. Scaricare il progetto da GitHub

```bash
git clone https://github.com/<tuo-utente>/<tuo-repository>.git
cd <tuo-repository>
```

Sostituisci `<tuo-utente>/<tuo-repository>` con il percorso del tuo repository.

La struttura del progetto è:

```
.
├── backend/            # API FastAPI
│   ├── server.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .env.example
├── frontend/           # App React
│   ├── src/
│   ├── package.json
│   ├── Dockerfile
│   ├── nginx.conf
│   └── .env.example
├── docker-compose.yml
└── README.md
```

---

## 2. Avvio con Docker (consigliato)

Dalla cartella principale del progetto:

```bash
docker compose up -d --build
```

Attendi il completamento della build (alla prima esecuzione può richiedere qualche minuto).

Poi apri il browser:

| Servizio | URL |
|----------|-----|
| **Applicazione (Pannello Capo)** | http://localhost:3000 |
| **Postazione Officina** | http://localhost:3000/scan |
| **Schermo Sala d'Attesa** | http://localhost:3000/display |
| API backend (facoltativo) | http://localhost:8001/api |

**Non serve alcun login**: l'accesso è diretto.

### Comandi utili

```bash
docker compose logs -f            # vedere i log in tempo reale
docker compose ps                 # stato dei container
docker compose down               # fermare l'applicazione
docker compose down -v            # fermare ED ELIMINARE i dati (DB + file caricati)
docker compose up -d --build      # ricostruire dopo modifiche al codice
```

I dati sono persistenti grazie ai volumi Docker:
- `mongo_data` → database MongoDB
- `uploads_data` → file caricati (logo, foto, video)

### Usare su un altro PC della rete / server

Se apri l'app da un altro computer, sostituisci `localhost` con l'IP del server (es. `http://192.168.1.50:3000`).
Poiché Nginx inoltra automaticamente le chiamate `/api` al backend, **non è necessaria alcuna configurazione aggiuntiva**.

---

## 3. Esecuzione manuale (senza Docker)

### 3.1 MongoDB
Assicurati che MongoDB sia in esecuzione (default `mongodb://localhost:27017`).

### 3.2 Backend

```bash
cd backend
python -m venv venv
source venv/bin/activate            # su Windows: venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env                # poi modifica i valori se necessario

uvicorn server:app --host 0.0.0.0 --port 8001
```

Il backend risponde su `http://localhost:8001/api`.

### 3.3 Frontend (in un altro terminale)

```bash
cd frontend
cp .env.example .env                # imposta REACT_APP_BACKEND_URL=http://localhost:8001
yarn install
yarn start                          # sviluppo su http://localhost:3000
```

Per una **build di produzione** statica:

```bash
yarn build
# il contenuto della cartella build/ può essere servito da qualsiasi web server
```

---

## 4. Configurazione (variabili d'ambiente)

### Backend (`backend/.env`)
| Variabile | Descrizione | Default |
|-----------|-------------|---------|
| `MONGO_URL` | Stringa di connessione MongoDB | `mongodb://localhost:27017` |
| `DB_NAME` | Nome del database | `avanzamento_produzione` |
| `CORS_ORIGINS` | Origini consentite (separate da virgola, `*` per tutte) | `*` |
| `STORAGE_DIR` | Cartella dei file caricati | `./uploads` |

### Frontend (`frontend/.env`)
| Variabile | Descrizione |
|-----------|-------------|
| `REACT_APP_BACKEND_URL` | URL del backend. Vuoto = stesso dominio (setup Docker con Nginx). Per esecuzione manuale: `http://localhost:8001` |

---

## 5. Primo utilizzo

1. Vai su **Anagrafiche** → aggiungi clienti e merce.
2. **Ordini** → crea un ordine (genera la scheda con codice a barre) → scarica/stampa il PDF.
3. **Dipendenti** → crea i dipendenti (genera i badge) → stampa i badge PDF.
4. **Schermo Sala** → carica **logo**, foto e video, imposta nome azienda, banner e orari.
5. **Postazione Officina** (`/scan`) → i dipendenti scansionano badge e schede per START/STOP e timbrano entrata/uscita.
6. **Dashboard**, **Report** e **Presenze** → monitoraggio ed export Excel/CSV.

> Suggerimento: collega un lettore di codici a barre USB (funziona come una tastiera) al PC della postazione officina.

---

## 6. Backup

- **Database:** `docker compose exec mongo mongodump --db avanzamento_produzione --out /data/db/backup`
- **File caricati:** si trovano nel volume Docker `uploads_data` (o nella cartella `STORAGE_DIR` in esecuzione manuale).

---

## 7. Risoluzione problemi

- **La build del frontend è lenta la prima volta:** è normale (scarica le dipendenze). Le esecuzioni successive usano la cache.
- **Porta già in uso (3000 o 8001):** modifica le porte nella sezione `ports` di `docker-compose.yml`.
- **Le immagini/video non si vedono:** verifica che il volume `uploads_data` esista e che il backend sia attivo (`docker compose logs backend`).
- **Reset completo:** `docker compose down -v` elimina database e file caricati per ripartire da zero.

---

## 8. Installer Windows (.exe) con avvio automatico

Se vuoi installare l'app su un PC Windows come **servizio che parte da solo all'accensione** (senza Docker), è disponibile un progetto installer completo nella cartella [`windows-installer/`](windows-installer/).

Include: MongoDB, backend e frontend impacchettati insieme, registrati come servizi Windows (avvio automatico), app raggiungibile su `http://localhost:8080`.

👉 Segui le istruzioni passo-passo in **[`windows-installer/BUILD_GUIDE.md`](windows-installer/BUILD_GUIDE.md)**: in sintesi, su un PC Windows con Python, Node/Yarn e Inno Setup basta eseguire `windows-installer\build.bat` per generare `FamilyPreziosi-Setup.exe`.

