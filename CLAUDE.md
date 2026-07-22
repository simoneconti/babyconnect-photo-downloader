# Scarica Foto Scuola — contesto tecnico

Strumento gratuito per genitori de "Il Casello dei Piccoli": scarica in automatico
le foto dei figli dal portale scolastico **BabyConnect** (`app.babyconnect.it`),
organizzate e con data corretta nei metadati EXIF.

Deliverable: **un unico file HTML autonomo** (`index.html`) — landing
page + bookmarklet. Nessun backend, nessun account, nessuna dipendenza esterna
tranne due librerie da CDN (piexifjs, JSZip) e il file `status.json` per il kill
switch (vedi sotto).

Autore/manutentore: Simone Conti (papà di Sole, iscritta al Casello dei Piccoli).

---

## Regola su git: nessuna operazione automatica

**Non eseguire mai comandi git in autonomia** — niente `git add`, `git commit`,
`git push`, niente di niente. Quella parte è **manuale**, la fa sempre e solo
l'utente. Puoi leggere lo stato del repo (`git status`, `git diff`, `git log`)
per capire cosa è cambiato, ma non modificare mai la history o il remote senza
che sia l'utente stesso a farlo.

---

## Come funziona il portale (reverse-engineered)

BabyConnect non ha un'API JSON pubblica leggera per il portale web. Ci sono due
endpoint rilevanti, con caratteristiche di velocità opposte:

### 1. Pagina-giorno — LENTA, usata solo per scoprire gli ID
```
GET /Parent/ActivityTimeline/{childId}/{giorno}/{mese}/{anno}
```
- Restituisce ~150 KB di HTML server-rendered (Razor).
- **Impiega 27-33 secondi**, misurato ripetutamente e confermato: è il server di
  BabyConnect a essere lento, non un limite nostro. Verificato che la concorrenza
  aiuta molto (6-12 richieste in parallelo impiegano quasi lo stesso tempo di
  una sola, es. 6 fetch paralleli ≈ 27.5s totali vs 26.7s per uno solo).
- Contiene gli ID delle attività del giorno, individuabili con la regex
  `/(?:showDetails|openGallery)\((\d+)\)/g` (stesso ID per entrambe le chiamate
  JS, deduplicato).
- **Non esiste una versione "partial" più leggera**: testato inviando
  `X-Requested-With: XMLHttpRequest`, risposta identica byte per byte.
- **Non esiste un endpoint mensile/bulk utilizzabile**: esiste una rotta
  `/Parent/ActivityTimeline/Month/{childId}/{mese}/{anno}` ma risponde sempre
  `500 MissingRequestBodyRequiredValueAccessor` — è un endpoint Kendo che il
  sito stesso non chiama mai via AJAX (il calendario reale cambia solo giorno,
  non mese), quindi non c'è una vera "firma corretta" da imitare. Directory
  listing sui path statici delle foto (`/resources/Routines/...`) è disattivato
  (404 immediati). Non perdere tempo a ritentare queste due strade.

### 2. Dettaglio attività — VELOCE, usato per il download vero
```
GET /Parent/ActivityTimeline/DettaglioAttivitaEducativa/{id}
Header: x-requested-with: XMLHttpRequest
```
- **~75ms** di risposta (misurato). Restituisce HTML con:
  - Titolo vero dell'attività in `<span class="titleDet">` (es. "Attività
    Grafiche pittoriche", "Festa dei Nonni", "Musica in fasce").
  - Le foto full-res in tag `<img src="/resources/Routines/...">` dentro un
    carosello owl-carousel — **niente miniature**, solo originali.
- Il token antiforgery, se servisse, si legge dal cookie `XSRF-TOKEN`
  (leggibile da JS, non httpOnly) e si manda come header `x-xsrf-token`.

### Struttura del path delle foto
```
/resources/Routines/{n}/{mese}/{GG-MM-AAAA}/{CATEGORIA}/{guid}/{IMG_AAAAMMGG_HHMMSS_xxx.jpg}
```
**Il GUID è per-GALLERIA, non per-foto**: più foto della stessa attività
condividono lo stesso GUID. Verificato coi dati reali (galleria 3806759, 8 foto,
stesso GUID `3c9bb260-...` per tutte). Per questo il nome file finale usa il
nome file *originale* (col suo timestamp `IMG_AAAAMMGG_HHMMSS`) come parte
univoca, mai il GUID — altrimenti foto diverse si sovrascriverebbero a vicenda
in una cartella piatta.

### Anni scolastici multipli per lo stesso bambino
Il widget Kendo del selettore bambino (`kendoDropDownList` su `#Figli`) ha un
`dataSource` con **un oggetto per ogni combinazione (bambino, anno scolastico)**.
Lo stesso bambino compare più volte con **id diversi** per ogni anno (es. Sole:
id `5583` per l'anno 2025/2026, id `4495` per il 2024/2025 — **entrambi gli id
funzionano davvero** per scaricare i giorni di quell'anno, verificato scaricando
attività reali con l'id dell'anno passato).

Ogni oggetto ha anche un `Genitore` annidato con Id/Nome/Cognome del genitore —
**va escluso esplicitamente** quando si estraggono i campi, altrimenti si
scambiano i genitori per fratelli. Vedi `parseChildren()`/`firstLevelFields()`
nel bookmarklet: tronca il testo dell'oggetto PRIMA di `"Genitore":{`,
`"Frequenza":{`, `"Sezione":{`, `"TipoUtente":{` prima di leggere Id/Nome/Cognome.

Regola di raggruppamento: **raggruppare per nome, mai per id** — altrimenti lo
stesso bambino con più anni sembra un fratello diverso nella UI.

Formato "Anno" nel dataSource: stringa tipo `"2025/2026"`. Da qui si derivano i
limiti del date-picker: dal 1° settembre dell'anno più basso al 31 luglio
dell'anno più alto (`yearBounds()`).

### Multi-attività nello stesso giorno
Un giorno può avere più attività distinte (verificato con dati reali:
21/5/2026 → "Festa dei Nonni" 12 foto + "DoReMi_Music" 4 foto). La regex di
discovery estrae tutti gli ID trovati nella pagina-giorno, non solo il primo:
già gestito correttamente, nessuna azione necessaria se si tocca questo codice.

---

## Decisioni di design (perché, non solo cosa)

- **Niente vera API bulk** → la fase discovery resta per-giorno, con
  concorrenza (default 5 richieste in parallelo — validato: il server la
  regge bene anche a 12 senza errori né rallentamenti sproporzionati).
- **Sync incrementale**: file `_stato-{childId}.json` nella cartella scelta
  (solo in modalità cartella, vedi sotto) con `scannedDays` (giorno → id
  attività trovate) e `savedPhotos` (nomi file già scaricati). Ricontrolla
  sempre gli ultimi `RECHECK_DAYS` giorni (upload in ritardo da parte della
  scuola).
- **Fallimento vs "zero attività" (bug corretto in v10)**: `idsForDay()`
  restituisce `{ok, ids}` esplicitamente. Solo `ok:true` viene scritto nel
  promemoria. Prima del fix, un fetch fallito (rete, sessione scaduta) veniva
  trattato come "giorno senza foto" e MAI PIÙ ritentato — bug subdolo e
  silenzioso, l'utente vedeva "fatto" con foto mancanti senza nessun errore
  visibile. Se si tocca questa parte, mantenere la distinzione.
  **Corollario verificato in v19 (Network tab, sessione scaduta reale)**: senza
  l'header `x-requested-with: XMLHttpRequest`, la pagina-giorno con sessione
  scaduta risponde `302 → /Account/Login`; `fetch()` segue il redirect da solo
  e ottiene `200 OK` con l'HTML del login, che `getText()` considera una
  risposta riuscita (`r.ok===true`) — il giorno veniva quindi marcato
  silenziosamente come "analizzato, zero attività" **per sempre**, aggirando
  del tutto il fix del bug sopra. `DettaglioAttivitaEducativa` non ha questo
  problema perché manda già quell'header (il server risponde 401 invece di
  reindirizzare). Fix: `idsForDay()` ora manda lo stesso header — verificato
  nel CLAUDE.md (sezione "Come funziona il portale") che da loggati la risposta
  resta identica byte per byte, quindi il fix è sicuro anche nel caso normale.
  **Non esteso a `detectChildId()`** (chiama `/Parent/ActivityTimeline` senza
  parametri, una rotta diversa mai testata con questo header): lì il
  fallimento è già visibile subito (schermata "non riconosco il bambino" con
  inserimento manuale dell'id), niente promemoria da corrompere silenziosamente
  — rischio di rompere il riconoscimento normale non giustificato senza prima
  verificare il comportamento di quella rotta specifica.
- **Naming file piatto**: `AAAA-MM-GG - NomeAttività - nomefileoriginale.ext`,
  tutti nella stessa cartella, senza sottocartelle (richiesta esplicita).
- **Modalità cartella vs ZIP**: `window.showDirectoryPicker` (Chrome/Edge)
  scrive su disco mano a mano, nessun limite di memoria, promemoria
  incrementale disponibile. Fallback ZIP (altri browser): tutto in RAM fino
  alla fine → `ZIP_MAX_WEEKDAYS = 40` (~8 settimane) come limite di sicurezza,
  attivo SOLO in questa modalità. Non serve in modalità cartella.
- **Calendario italiano scritto da zero** (non `<input type="date">` nativo):
  il formato/lingua del picker nativo segue le impostazioni di sistema
  dell'utente, non la pagina — impossibile forzare gg/mm/aaaa altrimenti
  (testato: `lang="it"` sull'input non basta). Il calendario custom **non ha
  ancora navigazione da tastiera** (nota per miglioramento futuro: se si
  ripristina il nativo per l'accessibilità, si perde la garanzia del formato —
  meglio aggiungere keyboard nav al custom che tornare al nativo).
- **Selettore bambino+anno unificato**: un'unica select con voci tipo
  "Sole Conti Sideri — 2025/2026 (in corso)", mai due select dipendenti
  (bambino → poi anno filtrato) — evita di dover gestire correttamente casi
  limite tipo "fratello non ancora iscritto in un anno passato".
- **Kill switch remoto (v9+, URL calcolato per-deploy dalla v12)**: prima di
  ogni uso, fetch al proprio `status.json` (fail-closed: se il controllo
  fallisce per qualunque motivo — rete, CORS, JSON non valido — lo strumento
  si blocca, non procede). Serve per poter "spegnere" lo strumento per tutti
  i genitori se BabyConnect dovesse avere problemi di carico, senza dover
  chiedere a nessuno di fare nulla.
  **URL non fisso**: `STATUS_URL` dentro `BOOKMARKLET()` è solo un fallback.
  Il valore vero viene calcolato nell'IIFE finale della landing page con
  `new URL("status.json", location.href).href` e iniettato nel codice
  serializzato prima di generare l'`href` del bookmarklet — così il
  bookmarklet controlla sempre il `status.json` pubblicato insieme alla
  pagina da cui è stato trascinato, senza un URL fisso da tenere allineato a
  mano.
  **Perché non un URL relativo dentro `BOOKMARKLET()`**: quando il
  bookmarklet gira, il contesto è la pagina di BabyConnect
  (`app.babyconnect.it`), non la landing page — un URL relativo lì
  risolverebbe contro il dominio sbagliato. La sostituzione va fatta prima,
  lato landing page, dove `location.href` è ancora quello giusto (`new URL()`
  gestisce da sola anche il sottopercorso del "project site" di GitHub
  Pages).
  **CORS**: GitHub Pages manda già `Access-Control-Allow-Origin: *` di
  default su tutti i file, nessuna configurazione manuale necessaria.
  **Compromesso noto**: GitHub Pages è servita dietro CDN Fastly con
  `Cache-Control: max-age=600` fisso, non derogabile per singolo file — lo
  spegnimento può impiegare fino a ~10 minuti a propagarsi a tutti gli
  utenti. Accettabile per il caso d'uso (non è un kill switch di sicurezza in
  tempo reale).
  **Limite noto**: copre solo chi ha trascinato il segnalibro dalla v9 in poi;
  versioni precedenti non hanno il controllo.
- **Numero di versione (`BM_VERSION`)**: un bookmarklet già trascinato nei
  preferiti NON si aggiorna mai da solo (il browser copia il codice al momento
  del trascinamento). Il numero, definito una sola volta dentro
  `BOOKMARKLET()`, viene letto anche dalla landing page tramite regex sul
  codice serializzato (`code.match(/BM_VERSION\s*=\s*"(\d+\.\d+\.\d+)"/)`),
  così i due punti non possono disallinearsi per un refuso.
  **Formato semver (`major.minor.patch`)**, adottato dalla v0.21.0 (le
  versioni precedenti erano un contatore intero semplice, v1-v21, ora solo
  storia nei commit). Il confronto per l'avviso "nuova versione disponibile"
  (`checkLatestVersion`/`compareSemver` nel bookmarklet) è un confronto
  semver numerico campo per campo, non lessicografico sulla stringa — non
  tornare a `parseInt` su tutto il numero se si tocca questa parte.
  **Alzare questo numero a ogni modifica funzionale reale del bookmarklet**
  (non serve per modifiche solo alla landing page/CSS che non toccano
  `BOOKMARKLET()`): patch per bugfix, minor per nuove funzionalità
  retrocompatibili, major per cambi che rompono qualcosa (es. formato del
  promemoria `_stato-{childId}.json`, naming dei file già scaricati).

---

## Infrastruttura

- **Sito**: **GitHub Pages**, unico metodo di pubblicazione — repo
  `simoneconti/babyconnect-photo-downloader` su GitHub, pubblicato su
  `https://simoneconti.github.io/babyconnect-photo-downloader/` (Settings →
  Pages → Deploy from branch `master`, root; richiede repo pubblico sul piano
  free). File principale `index.html` (questo stesso file, copiato così
  com'è), più `status.json` per il kill switch.
- **Deploy**: nessuna pipeline manuale — pubblicazione automatica ad ogni
  `git push` su `master`. Aggiornare `status.json` per attivare il kill
  switch è quindi anch'esso un semplice push, non richiede accesso a nessun
  server.
- **CORS**: GitHub Pages manda di default `Access-Control-Allow-Origin: *` su
  tutti i file, nessuna configurazione manuale necessaria.

---

## Cose provate e scartate (per non riprovarle)

- Endpoint `Month` Kendo → sempre 500, non è la firma giusta, il sito non lo
  usa mai via AJAX.
- Directory listing su `/resources/Routines/` → disattivato, 404 sempre.
- API `/api/...` cookie-based sul portale web → 404, l'API veloce esiste solo
  lato app mobile (probabilmente token-based, altro host) e comunque non
  richiamabile da un bookmarklet cookie-based, né condivisibile con genitori
  non tecnici.
- Versione "partial" della pagina-giorno via `X-Requested-With` → identica
  alla versione piena, nessun guadagno.

## Miglioramenti noti, non ancora fatti

1. Navigazione da tastiera sul calendario italiano custom (accessibilità).
2. Avviso se si sceglie una cartella diversa da quella usata l'ultima volta
   per lo stesso bambino (oggi funziona comunque, solo riparte da zero lì).
3. Considerata e scartata: un'estensione di browser al posto del bookmarklet
   (risolverebbe l'aggiornamento automatico e il fastidio del drag-and-drop,
   stesso modello di sicurezza — nessuna credenziale passa da un server terzo)
   — **non voluta, non perseguire senza nuova richiesta esplicita**.
4. Considerato e scartato: portale centralizzato dove i genitori inseriscono
   le credenziali BabyConnect e un server scarica per loro — rischio troppo
   alto (credenziali di terzi transitano da un server che dovresti gestire tu,
   pattern di accesso centralizzato più visibile e più a rischio ban lato
   BabyConnect) — **scartato consapevolmente, non riproporre senza nuovo
   contesto che cambi la valutazione del rischio**.
