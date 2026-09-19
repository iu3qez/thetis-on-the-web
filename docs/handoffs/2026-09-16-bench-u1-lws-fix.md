---
artifact_contract: "ce-handoff/v1"
created_at: "2026-09-16T19:31:41Z"
title: "TOTW — banco di riferimento di U1 a meta', crash di libwebsockets risolto sulla stazione"
summary: "Piano modulare scritto e U1 implementata con quattro correzioni emerse al banco; deskHPSDR ricompilato contro libwebsockets main per un crash nel permessage-deflate; checklist di riferimento eseguita solo in parte prima delle rimozioni di U2."
keywords: ["totw", "deskhpsdr", "banco", "u1", "u2", "checklist", "libwebsockets", "permessage-deflate", "split", "rx2", "piano-modulare"]
cwd: "/Users/sf/Developer/thetis-on-the-web/.claude/worktrees/handoff-riprendere-48722d"
resume_focus: "Riprendere il banco di riferimento di U1 (checklist parti 2-4, ritest VFO B e slider AF) e poi le rimozioni di U2."
repository: "iu3qez/thetis-on-the-web"
repo_root_sha: "a14fbec33d24418421b76761f4bbfe888d78bbb0"
branch: "feat/totw-modular-operable-client"
head: "c17806e"
worktree_path: "/Users/sf/Developer/thetis-on-the-web/.claude/worktrees/handoff-riprendere-48722d"
---

# Handoff — TOTW, banco di U1 a meta' e crash di libwebsockets risolto

**Sostituisce `docs/handoffs/2026-09-15-bench-c1-c2-main-trunk.md`** per lo stato del lavoro. Quello resta valido per la storia di C1 e C2.

Il campo `head` indica l'ultimo commit di codice. Il commit che aggiunge questo handoff, il piano, l'appunto e la modifica a `CONCEPTS.md` sta sopra di esso sullo stesso branch.

## Intento dell'operatore

- Criterio del triage, detto dall'operatore: tutto cio' che impedisce di usare la radio e' un problema, in qualunque repository stia la correzione.
- Scelte dell'operatore durante la pianificazione: sorgente modulare con `totw.html` generato; rimozione prima della migrazione; via DX cluster, greyline, marker WSPR/JS8, palette alternative, memorie e ripiego audio nello spettro; restano i temi UI; audio TCI e mobile a costo zero; due sorgenti audio TX selezionabili; keepalive TCI sul modello CWNet; spettro a bin dentro il lavoro sullo spettro e MIDI dopo i controlli.
- Durante il banco: checklist di riferimento prima delle rimozioni; libwebsockets `main` subito sulla stazione; correggere solo gli slider AF e aprire issue per split e RX2; push del branch, senza PR e senza toccare `main`.

## Stato

| Pezzo | Stato |
|---|---|
| `main` | invariato a `c154e66`, nessun push |
| Branch `feat/totw-modular-operable-client` | pushato su `origin`, nessuna PR |
| Piano `docs/plans/2026-09-16-0316-feat-totw-modular-operable-client-plan.md` | `implementation-ready`, 19 unita'; decisioni della revisione ancora aperte (vedi sotto) |
| U1 | implementata: `7ae9b74`, piu' tre correzioni emerse al banco `732203f`, `7ac1092`, `c17806e`; banco di U1 solo in parte |
| U2 | iniziata solo con la checklist `aab0f4f` (`docs/bench/reference-checklist.md`); nessuna rimozione fatta |
| U3-U19 | non iniziate |
| Appunto | `docs/solutions/integration-issues/libwebsockets-permessage-deflate-assert-crashes-deskhpsdr-ptt.md` |
| `CONCEPTS.md` | aggiunte "Finestra di vista" e "Costo zero" durante la pianificazione |

### Commit di U1 e cosa verificare

- `7ae9b74`: i pulsanti ⚙/DIAG/? con classe `band-btn` portavano il VFO a `NaN`; `split_enable` letto dall'ultimo argomento; `rx_sensors` come S-meter e silenziato nel log; volume RX2 a tre argomenti; watchdog su TUNE con il timeout del PTT; autorepeat ignorato.
- `732203f`: doppio clic su CONNECT apriva due WebSocket, entrambi con `onMsg`, e l'audio arrivava due volte (crackling). Il pulsante e' ignorato durante la connessione e `connect()` chiude il socket precedente.
- `7ac1092`: pressioni corte del PTT chiamavano due volte l'armamento del microfono; il secondo `AudioContext` rompeva il primo e un `alert()` bloccava la pagina. Armamento single-flight, errori nel log, PTT rilasciato se il microfono non parte (scelta mia, non chiesta dall'operatore).
- `c17806e`: slider AF RX1, mobile e RX2 portati a -40..0 dB, il limite di `tci_clamp_volume` nel server; `rx_volume` dal server muove gli slider.

Tutte provate nel browser integrato con mock e WebSocket finto, poi al banco dove indicato sotto. Dopo ogni modifica a `totw.html` sono stati eseguiti `node scripts/build-variants.js split` e `node scripts/smoke-test.js`.

## Banco del 2026-09-16

Client aperto da `http://127.0.0.1:8765/totw.html`, stazione `ws://ubuntu.lan:50001`.

| Voce della checklist | Esito |
|---|---|
| B2, C2 (VFO dopo ⚙/DIAG/?) | OK |
| F5 (S-meter) | OK |
| J1-J4 (PTT breve e lungo, timeout di TUNE) | OK dopo `7ac1092` e il fix di libwebsockets |
| K1 (audio RX) | OK dopo `732203f` |
| F7 (split) | KO: difetto del server, issue `iu3qez/deskhpsdr` #11 |
| F4, F3 (slider AF) | corretti in `c17806e`, **non ritestati** |
| VFO B e A->B | **non verificato**: nel test A e B erano gia' uguali; il server applica `vfo:0,1,<freq>` (provato con un client TCI minimale) |
| Parti 2, 3 e 4 della checklist (A, B1, C3-C10, D, E, F1-F2, L, M, N) | **non eseguite** |

## La stazione (stato machine-local)

- deskHPSDR su `ubuntu.lan` al commit `eaec707`, ricompilato alle 20:31 contro libwebsockets statica `d7f7fdeaf` (contiene il fix `2fa375756a`), in `~/deskhpsdr/libwebsockets-5/` (ignorata da git). Diagnosi e procedura nell'appunto.
- Backup del binario vecchio in `~/deskhpsdr/deskhpsdr.lws-4.3.5`: **non e' ignorato da git** in quel repository.
- Porta TCI 50001; due ricevitori attivi, RX2 come ricevitore attivo al momento del test dello split.
- Il checkout sul Mac `/Users/sf/Developer/deskhpsdr` e' allo stesso commit `eaec707`.
- Server statico locale per il banco: `.claude/launch.json` nel worktree (non tracciato, `python3 -m http.server 8765`); l'app lo ferma quando la sessione e' inattiva.

## Issue aperte in questa sessione

- `iu3qez/deskhpsdr` #11: `split_enable` scritto relativo e letto assoluto, stato invertito con RX2 attivo.
- `iu3qez/deskhpsdr` #12: `rx_enable` e' solo query, RX2 ON da TCI non fa nulla.
- `iu3qez/thetis-on-the-web` #15: split usabile, marker e sintonia di VFO B dallo spettro.
- `iu3qez/thetis-on-the-web` #16: supporto RX2 completo, con le decisioni di scopo ancora da prendere.

## Revisione del piano: voci aperte

La revisione del documento ha applicato quattro correzioni meccaniche. Le voci sotto **non sono nel piano** e vanno decise prima delle unita' che toccano. Raccomandazioni della revisione, non decisioni dell'operatore.

Decisioni:

1. Sorgente audio TX di default "Stazione" invece di "Browser" (tocca R26, KTD14, U16). Il default Browser e' una mia assunzione.
2. Sorgente spettro di default "bin" senza scelta salvata (R17, U11).
3. Nuovo esempio di accettazione: QSO completo su WAN simulata prima della fase MIDI.
4. Link-lost del PTT dopo 5 s senza messaggi dal server, non dopo tre sonde a 1 Hz; esclusa la fase di richiesta del microfono (KTD14, U16).
5. Via d'uscita dallo stato di allarme "stato TX ignoto" (KTD14).
6. Togliere il buffer del microfono dalla pressione, che supera il costo zero di R6 (KTD14). Con `7ac1092` il microfono non e' ancora bufferizzato: resta coerente.
7. Server senza `capabilities_ex`: query stock sentinella (`trx_count`) per distinguere "assente" da "lento" (KTD12, AE6).
8. `capabilities_ex` in U12 e modulo delle capacita' in U6 (sequenza delle unita').
9. Nuovo KTD sul collegamento fra codice legacy e moduli ESM (iniezione, niente import dal legacy).
10. Issue a monte su `dl1bz/deskhpsdr` per il TX bloccato su socket half-open, e U12 preparata come patch proponibile.

Correzioni proposte, coerenti con decisioni gia' nel piano: rilascio su `blur` solo per il PTT da tastiera; transizione `requested -> owned` sull'eco `trx` senza `tx_owner_ex` e con radio in RX; PTT e TUNE mobili attraverso la macchina a stati; regola DROP in entrambi i versi per la prova half-open; `,error` dei comandi `_ex` che riporta subito il valore; `unsupported -> unknown` alla chiusura; esiti `aborted` e `refused,rx` dello zero-beat con timeout nel client; diff di U3 contro `src/legacy/`; checklist prima delle rimozioni (**gia' applicata**, decisione dell'operatore).

Osservazione di sicurezza a basso punteggio nella revisione: il PTT desktop rilascia su `onmouseleave` (`totw.html`, pulsante `pttBtn`), che non risale; convertito a `data-action` smetterebbe di funzionare in silenzio.

## Trappole verificate in questa sessione

- **Cache del browser**: dopo una modifica Firefox e il browser integrato servono la versione vecchia; `Cmd+Shift+R` o un parametro `?v=`.
- **Browser integrato**: non apre file:// senza un server; non va collegato alla stazione reale mentre l'operatore e' al banco, usare un WebSocket finto.
- **Due connessioni dallo stesso client**: `ss -tni '( sport = :50001 )'` sulla stazione e `lsof -nP -iTCP:<porta>` sul Mac identificano il processo.
- **Build lunghe via SSH**: il tool SSH va in timeout; lanciare con `nohup ... > /tmp/<log> &` e interrogare il log.
- **Numeri di riga del piano**: U1 ha spostato le righe di `totw.html` di alcune decine; l'Appendix del piano e' a `c154e66`.
- **SIGABRT di deskHPSDR**: la ricetta `coredumpctl` + `__abort_msg` e' nell'appunto.
- Restano valide le trappole del handoff precedente: `gh` sempre con `-R`, niente `owner/repo#N` in chiaro verso il progetto originale.

## Passi successivi plausibili

Un solo percorso, in sequenza:

1. Completare il banco di U1 con la checklist (`docs/bench/reference-checklist.md`): ritest di F3, F4 e VFO B con A diverso da B, poi parti 2, 3 e 4. Gli esiti vanno registrati, oggi esistono solo in questa sessione.
2. Rimozioni di U2, un blocco per commit (Appendix del piano, righe a `c154e66`), confrontando con l'esito della checklist.
3. Prima di U6, U11, U12 e U16 decidere le voci aperte della revisione che li riguardano.

Indipendenti: riallineare nel repository di deskHPSDR il piano del 2026-09-08 (KTD9), l'handoff del 2026-09-09 e §5 di `documentation/deskHPSDR_TCI_Remote_Extensions.md` con il vincolo su libwebsockets; issue #11 e #12 lato server.
