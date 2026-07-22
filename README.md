# Scarica Foto Scuola — BabyConnect

Strumento gratuito per genitori de **"Il Casello dei Piccoli"**: scarica in
automatico le foto dei figli dal portale scolastico [BabyConnect](https://app.babyconnect.it),
organizzate per data e con **data corretta nei metadati EXIF**.

Sito live: https://simoneconti.github.io/babyconnect-photo-downloader/

## Cos'è

BabyConnect mostra le foto dei bambini una attività alla volta, senza modo di
scaricarle in blocco. Questo strumento risolve il problema con un
**bookmarklet**: un segnalibro del browser che, una volta trascinato nella
barra dei preferiti, scarica tutte le foto di un intervallo di date scelto,
rinominandole con data e nome attività e scrivendo la data corretta nei
metadati EXIF di ogni immagine.

## Come si usa

1. Si apre la landing page e si trascina il pulsante nella barra dei
   preferiti del browser (nessuna installazione, nessun account, nessuna
   estensione).
2. Si va sul portale BabyConnect, si effettua il login normale.
3. Si clicca il segnalibro: si apre un pannello che chiede bambino e
   intervallo di date.
4. Lo strumento scarica le foto direttamente su disco (Chrome/Edge, cartella
   scelta dall'utente) o in uno ZIP (altri browser), con sync incrementale
   per non riscaricare foto già salvate.

## Architettura

- **Un unico file HTML autonomo** (`index.html`): landing page + codice del
  bookmarklet. Nessun backend, nessun account, nessuna dipendenza esterna a
  parte due librerie da CDN (`piexifjs` per gli EXIF, `JSZip` per lo ZIP) e
  `status.json` per il kill switch.
- **Nessuna credenziale passa da un server terzo**: il bookmarklet gira
  interamente nel browser dell'utente, usando la sessione già autenticata sul
  portale BabyConnect.
- **Kill switch remoto**: prima di ogni uso, il bookmarklet controlla
  `status.json` sul sito — permette di disattivare lo strumento per tutti se
  BabyConnect avesse problemi di carico, senza intervento lato utente.

I dettagli tecnici completi (reverse engineering degli endpoint del portale,
decisioni di design, infrastruttura di deploy, cose provate e scartate) sono
documentati in [CLAUDE.md](CLAUDE.md).

## Deploy

GitHub Pages, pubblicazione automatica ad ogni push su `master` (vedi sezione
Infrastruttura in [CLAUDE.md](CLAUDE.md)).

## Autore

Simone Conti — papà di Sole, iscritta a Il Casello dei Piccoli.
