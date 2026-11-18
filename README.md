# Query SQL per l'analisi forense di Foto iOS e WhatsApp

Raccolta di query SQLite commentate per l'analisi forense di database di dispositivi mobili, con particolare attenzione a **contenuti eliminati**, **cestinamenti** e **salti di ID**.

## Contenuto del repository

| File | Database | Piattaforma | Contenuto principale |
|---|---|---|---|
| [`query_photos_sqlite.txt`](query_photos_sqlite.txt) | `Photos.sqlite` | iOS | Asset nel cestino, eliminazioni definitive dalla persistent history di Core Data (`ACHANGE`), ricostruzione dei cestinamenti, salti di `Z_PK` |
| [`query_whatsapp_ios_ChatStorage.txt`](query_whatsapp_ios_ChatStorage.txt) | `ChatStorage.sqlite` | iOS | Chat, messaggi, messaggi eliminati per tutti, salti di `Z_PK` e di `ZSORT`, media, posizioni, gruppi |
| [`query_whatsapp_android_msgstore.txt`](query_whatsapp_android_msgstore.txt) | `msgstore.db` | Android | Chat, messaggi, revoche, messaggi modificati, salti di `_id`, stati ed effimeri, media, chiamate |

## Percorsi tipici dei database

| Database | Percorso |
|---|---|
| `Photos.sqlite` | `/private/var/mobile/Media/PhotoData/Photos.sqlite` |
| `ChatStorage.sqlite` | `/private/var/mobile/Containers/Shared/AppGroup/<UUID>/ChatStorage.sqlite` (gruppo `group.net.whatsapp.WhatsApp.shared`) |
| `msgstore.db` | `/data/data/com.whatsapp/databases/msgstore.db` |
| `wa.db` (rubrica WhatsApp Android) | `/data/data/com.whatsapp/databases/wa.db` |

I percorsi possono variare in base alla versione del sistema operativo e al tipo di acquisizione.

## Contenuto dei file di query

### Photos.sqlite (iOS)

1. Verifiche preliminari: schema, entità Core Data, copertura temporale della persistent history
2. Asset e album attualmente nel cestino ("Eliminati di recente")
3. Eliminazioni definitive (`ACHANGE`, `ZCHANGETYPE = 2`) con motivazione e contesto della transazione
4. Ricostruzione dei cestinamenti per giorno e per ora, selezioni multiple, origine locale o sincronizzata
5. Supporto a ricerche in altre fonti: UUID degli asset eliminati, salti di `Z_PK`
6. Riepilogo delle transazioni per componente

### WhatsApp iOS - ChatStorage.sqlite

1. Verifiche preliminari e distribuzione dei tipi di messaggio
2. Chat e messaggi
3. Messaggi eliminati per tutti, con contesto
4. Salti di ID: `Z_PK` globale, `ZSORT` per chat, contatori, chat eliminate, record orfani
5. Media, posizioni e contatti condivisi
6. Gruppi, membri e push name
7. Messaggi stellati, ricerca per parola chiave, incongruenze temporali, periodi di inattività

### WhatsApp Android - msgstore.db

1. Verifiche preliminari, `sqlite_sequence` e tipi di messaggio
2. Chat e messaggi
3. Messaggi eliminati per tutti (`message_revoked`) e messaggi modificati (`message_edit_info`)
4. Salti di `_id`, chat con ultimo messaggio mancante, record orfani, stati e messaggi effimeri
5. Media, posizioni e contatti condivisi
6. Registro chiamate e relativi salti di ID
7. Messaggi di sistema, stellati, ricerca per parola chiave, incongruenze temporali, periodi di inattività

## Come usare le query

1. **Lavorare sempre su una copia** del database, insieme ai file `-wal` e `-shm`, mai sull'originale acquisito.
2. Aprire la copia con un client SQLite, ad esempio DB Browser for SQLite, DBeaver o la CLI `sqlite3`. Le query che usano funzioni finestra (`LAG`, `LEAD`) richiedono **SQLite 3.25 o successivo**.
3. Eseguire per prime le query della **sezione 1** di ogni file, per verificare tabelle e colonne effettivamente presenti.
4. Adattare gli **intervalli di date** di esempio (26–28/05/2026) al caso in esame.

### Conversione dei timestamp

| Database | Formato | Conversione SQLite |
|---|---|---|
| `Photos.sqlite`, `ChatStorage.sqlite` | Cocoa time (secondi dal 01/01/2001 UTC) | `datetime(campo + 978307200, 'unixepoch')` |
| `msgstore.db` | Unix time in millisecondi | `datetime(campo / 1000, 'unixepoch')` |

Tutti gli orari restituiti sono in **UTC**. Per l'ora italiana si aggiunge il modificatore `'+2 hours'` (ora legale, CEST) o `'+1 hours'` (ora solare, CET). Si sconsiglia `'localtime'`, che usa il fuso orario della macchina di analisi e non quello del dispositivo.

## Avvertenze

- **Gli schemi cambiano tra versioni** di iOS e di WhatsApp. Nomi di colonne e tabelle vanno verificati su ogni dataset.
- **I codici dei tipi di messaggio** (ad esempio `ZMESSAGETYPE = 14` su iOS e `message_type = 15` su Android per i messaggi eliminati per tutti) sono quelli comunemente documentati e vanno confermati sul dispositivo in esame.
- **Il significato dei campi `ZTOMBSTONE`** in `Photos.sqlite` dipende dall'entità e dalla versione di iOS. La mappatura indicata nei commenti va validata caso per caso, ad esempio con l'intervallo di 30 giorni degli asset scaduti dal cestino.
- **Un salto di ID non prova da solo un'eliminazione manuale.** Può derivare da aggiornamenti di stato scaduti, messaggi effimeri, svuotamento o eliminazione di chat, ripristino da backup o record tecnici rimossi dall'applicazione. Va sempre contestualizzato.
- **La persistent history di Core Data viene potata periodicamente.** L'assenza di eventi in `ACHANGE` non dimostra l'assenza di eliminazioni anteriori alla prima transazione disponibile.
- Le query sono uno **strumento di supporto all'analisi**. I risultati vanno validati con altri strumenti e fonti (WAL, freelist, file system, backup).

## Contributi

Segnalazioni di errori, differenze di schema tra versioni e nuove query sono benvenute tramite issue o pull request. Nel contributo è utile indicare:

- versione del sistema operativo e dell'applicazione;
- tabella e colonne coinvolte;
- descrizione del comportamento osservato.

## Esclusione di responsabilità

Il materiale è fornito "così com'è", senza garanzie di completezza. 
