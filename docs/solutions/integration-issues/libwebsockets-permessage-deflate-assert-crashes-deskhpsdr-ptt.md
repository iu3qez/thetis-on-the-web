---
title: Il permessage-deflate di libwebsockets 4.3.5 fa abortire deskHPSDR quando TOTW trasmette il microfono
date: 2026-09-16
category: integration-issues
module: deskhpsdr-client
problem_type: integration_issue
component: messaging
symptoms:
  - "deskHPSDR va in SIGABRT circa 3 s dopo che il PTT di TOTW inizia a trasmettere il microfono del browser"
  - "Core dump ripetibili in systemd-coredump, due occorrenze il 2026-09-16"
  - "Il thread che abortisce e' tci_lws_server, dentro lws_service, con frame di libwebsockets senza simboli"
  - "Messaggio di abort nel core, extension-permessage-deflate.c:350, Assertion 'priv->rx.avail_out' failed"
root_cause: logic_error
resolution_type: dependency_update
severity: critical
framework_version: libwebsockets 4.3.5 (Ubuntu libwebsockets19t64 4.3.5-3ubuntu1)
related_components:
  - tci_server
  - tx_audio_capture
  - libwebsockets
  - build_tooling
tags: [tci, websocket, permessage-deflate, libwebsockets, deskhpsdr, ptt, sigabrt, crash]
---

# Il permessage-deflate di libwebsockets 4.3.5 fa abortire deskHPSDR quando TOTW trasmette il microfono

## Problem

deskHPSDR va in abort (SIGABRT) quando un client TCI che ha negoziato permessage-deflate invia frame binari grandi. Il caso reale e' il PTT di TOTW (Firefox) che trasmette il microfono del browser a deskHPSDR su `ubuntu.lan:50001`. Il difetto non e' in deskHPSDR ne' in TOTW: e' nel percorso RX di permessage-deflate di libwebsockets 4.3.5 (pacchetto Ubuntu `libwebsockets19t64 4.3.5-3ubuntu1`), contro cui era linkato il binario della stazione.

Riferimenti, in tre repository diversi: `iu3qez/deskhpsdr` al commit `eaec707` (percorsi `src/`, `Makefile`, `build-libwebsockets.sh`); `warmcat/libwebsockets` al tag `v4.3.5` (file `lib/roles/ws/ext/extension-permessage-deflate.c`, di seguito `pmd.c`) e fix upstream `2fa375756a`; questo repository, `totw.html` sul branch `feat/totw-modular-operable-client` (commit `c17806e`, non ancora in `main` al 2026-09-16).

Catena causale:

- deskHPSDR offre permessage-deflate a ogni client (`src/tci.c:7215-7230`, stringa `"permessage-deflate; client_no_context_takeover; client_max_window_bits"` a `src/tci.c:7225`). Il fork l'ha aggiunto per comprimere lo spettro sul link 4G. I browser lo negoziano sempre.
- TOTW invia un frame TCI type 2 (`TX_AUDIO_STREAM`) per ogni blocco del worklet: header di 64 byte + 2048 campioni duplicati L=R in float32, cioe' 16448 byte ogni 42,7 ms (`totw.html:5142`, `totw.html:5222-5250`). Firefox li comprime.
- In 4.3.5 il buffer d'uscita dell'inflate vale `1 << PMD_RX_BUF_PWR2` = 1024 byte (default a `pmd.c:166`, applicato a `pmd.c:262`). Il `rx_buffer_size` 8192 dei protocolli TCI (`src/tci.c:7198-7200`) fa solo da tetto verso il basso (`pmd.c:53-66`). Ogni messaggio si gonfia quindi in almeno 17 passate.
- Nel ramo "RX trailer apply 2" (`pmd.c:322-351`) l'input del messaggio e' finito ma l'uscita puo' essere piena. lws aggiunge 5 byte di margine (`pmd.c:331`), passa il trailer sintetico `00 00 ff ff` con `Z_SYNC_FLUSH` (`pmd.c:335-337`) e poi esegue `assert(priv->rx.avail_out)` (`pmd.c:350`). Se zlib deve ancora emettere piu' di 5 byte, per esempio la coda di un match lungo fino a 258 byte, riempie il margine e l'assert scatta. Il contenuto compresso lo sceglie il peer, quindi la condizione e' raggiungibile dalla rete.
- Stesso percorso, secondo difetto: `pmd.c:255-258` usa il buffer del chiamante come `next_in`, e `pmd.c:374-375` ritorna `PMDR_HAS_PENDING` mentre zlib punta ancora li'. Il ws role considera il buffer consumato e lo libera o lo riusa. E' un use-after-free, e puo' gonfiare byte di un'altra connessione.

## Symptoms

- Con il PTT di TOTW premuto, deskHPSDR termina circa 3 s dopo, cioe' dopo una settantina di frame audio. SIGABRT, core dump presenti.
- Due occorrenze il 2026-09-16, alle 20:20:32 e alle 20:22:39 CEST. Poco prima lo stesso PTT aveva funzionato una volta: il crash dipende dai byte compressi, quindi dal segnale del microfono.
- Nello stesso momento anche la pagina TOTW si bloccava. Era un problema separato del client (vedi sotto).
- `coredumpctl info` mostra il thread che abortisce: `tci_lws_server` -> `lws_service` -> frame `n/a` -> `abort`.
- Messaggio di abort letto dal core: `./lib/roles/ws/ext/extension-permessage-deflate.c:350: lws_extension_callback_pm_deflate: Assertion 'priv->rx.avail_out' failed.`

## What Didn't Work

1. **Cercare il bug nel pulsante PTT del client.** Il blocco della pagina era reale ma separato: un `alert()` causato da una race nell'armamento del microfono, corretto a parte (`armTxMic`, `totw.html:5184`). Correggerlo non toccava il crash del server.
2. **Leggere lo stack del main thread.** Era fermo in `XGetGeometry` (X11) e sembrava un'interfaccia bloccata. Non era il thread che abortiva: in un core multi-thread va cercato il thread che contiene `raise`/`abort`.
3. **Fidarsi del solo stack del thread che abortisce.** La libwebsockets di sistema non aveva simboli, quindi i frame erano `n/a` e lo stack diceva solo "dentro `lws_service`". Il file e la riga sono arrivati solo dal messaggio di abort salvato nel core.
4. **Allargare `rx_buf_size`, scartato in analisi.** Il buffer piu' grande sposta solo il punto in cui l'uscita si riempie. Il peer controlla ancora il contenuto compresso, e l'use-after-free resta.

## Solution

### Diagnosi (sulla stazione)

```sh
coredumpctl list deskhpsdr
coredumpctl info <PID>        # cercare il blocco "Stack trace of thread" che contiene raise/abort
coredumpctl debug <PID> --debugger-arguments="-batch -nx \
  -ex 'set debuginfod enabled off' \
  -ex 'printf \"%s\n\", ((char*)__abort_msg)+4'"
```

`__abort_msg` punta a `struct abort_msg_s { unsigned int size; char msg[0]; }` (glibc-2.39 `include/stdlib.h:356-360`). `__assert_fail_base` ci copia il testo prima di `abort()` (`assert/assert.c:69-79`), e `+4` salta `size`. Il testo porta file e riga esatti anche senza simboli. Da li' si confronta `pmd.c` di 4.3.5 con `main` e si arriva al fix upstream.

### Opzioni valutate

- (A) Disattivare permessage-deflate (`src/tci.c:7215-7233`). Elimina il crash ma perde la compressione dello spettro sul WAN.
- (B) Linkare staticamente libwebsockets `main` con il fix. **Scelta dall'operatore.**
- (C) Lasciare il server com'e' ed evitare il microfono dal browser. Scartata: qualunque client TCI su LAN/WireGuard puo' abbattere la stazione, e l'use-after-free resta.

### Fix upstream

Commit `2fa375756a` di libwebsockets ("ws: pmd: C-066: own unconsumed rx input, make trailer application a state", 2026-09-08). Il GitHub compare API lo da' `diverged`/ahead rispetto a `v4.5.8` e a `v5.0.0`: **non e' in nessuna release al 2026-09-16**.

### Applicazione (in `~/deskhpsdr` sulla stazione)

```sh
./build-libwebsockets.sh
git -C libwebsockets-5 merge-base --is-ancestor 2fa375756a HEAD && echo fix-presente
grep LWS_LIBRARY_VERSION libwebsockets-5/build/include/lws_config.h   # "5.0.99-sai-d7f7fdeaf"
cp deskhpsdr deskhpsdr.lws-4.3.5
make clean && make -j4
ldd deskhpsdr | grep websockets                                        # vuoto: statico
strings deskhpsdr | grep -e 5.0.99-sai-d7f7fdeaf -e 'permessage-deflate available'
```

- `build-libwebsockets.sh` clona il branch di default in `libwebsockets-5/` (`build-libwebsockets.sh:7,22`). Compila solo la libreria statica, con extension e zlib (`build-libwebsockets.sh:47-51`), e installa in `libwebsockets-5/build`.
- Il Makefile preferisce quella libreria quando ci sono sia l'header sia `libwebsockets.a` (`Makefile:510-518`). La riga di link diventa `./libwebsockets-5/build/lib/libwebsockets.a -lz`.
- `make clean` e' obbligatorio perche' il Makefile non traccia le dipendenze dagli header (nessun `-MMD`). Gli oggetti compilati con gli header 4.3.5 non verrebbero ricompilati. Questo vale anche per il gate `#if !defined(LWS_WITHOUT_EXTENSIONS)` di `src/tci.c:7215`, che dipende da `lws_config.h`.
- Dopo il riavvio di deskHPSDR, pressioni brevi e lunghe del PTT al banco non hanno piu' causato crash. Commit compilato: `d7f7fdeaf`, che contiene `2fa375756a`.

## Why This Works

Il fix `2fa375756a` corregge entrambi i difetti dentro `lws_extension_callback_pm_deflate`:

- **Assert.** L'applicazione del trailer diventa uno stato (`rx_trailer_pending`). Se zlib non accetta tutti e 4 i byte perche' l'uscita e' piena, il resto di `trail[]` resta come input pendente. `trail[]` e' statico, quindi il puntatore resta valido. Il trailer si completa in una passata successiva con un buffer d'uscita nuovo. `was_fin` viene posto solo quando `avail_in == 0`, e l'`assert` e' rimosso.
- **Use-after-free.** Prima di tornare all'event loop con input non consumato, la parte residua viene copiata in `buf_rx_holding`, di proprieta' dell'extension, come faceva gia' il TX con `buf_tx_holding`. zlib non punta piu' a `pt->serv_buf`, condiviso da tutte le wsi del thread, ne' a segmenti di buflist gia' liberati.
- **Reset.** Al reset per no_context_takeover vengono azzerati anche `next_in`/`avail_in`, che `inflateInit2()` non tocca.

Il link statico garantisce che il binario esegua questo codice qualunque sia la libwebsockets installata dal sistema. `ldd` vuoto e la stringa di versione nel binario lo verificano.

## Prevention

- **Fallback silenzioso a 4.3.5.** `libwebsockets-5/` e' in `.gitignore:43`. Se sparisce (per esempio `git clean -xfd` o un clone nuovo), `Makefile:524-527` torna alla libreria di sistema via pkg-config e lo segnala solo con una riga `$(info)`. Dopo ogni build ricontrollare `ldd deskhpsdr | grep websockets` (deve essere vuoto) e `strings deskhpsdr | grep 5.0.99-sai`.
- **Commit non fissato.** `build-libwebsockets.sh` riusa la sorgente esistente senza aggiornarla (`build-libwebsockets.sh:24`). Se la directory manca, invece, clona la HEAD corrente (`build-libwebsockets.sh:22`). Registrare nel runbook della stazione il commit fissato (`d7f7fdeaf`) e il controllo `git merge-base --is-ancestor 2fa375756a HEAD`.
- **Aggiornamenti di sicurezza manuali.** Con il link statico `apt` non porta piu' fix di libwebsockets, ne' regressioni. Seguire upstream i commit su `lib/roles/ws/` e tornare a un tag appena una release contiene `2fa375756a`.
- **Superficie d'attacco.** Il commento a `src/tci.c:7217-7219` dice che i client TCI nativi non sono toccati. Ma ogni client che offre permessage-deflate, cioe' ogni browser, raggiunge il percorso di inflate. Per come libwebsockets struttura le estensioni (callback `LWS_EXT_CB_PAYLOAD_RX` dell'extension, eseguita prima della callback del protocollo), il crash avviene prima che il frame arrivi al codice di deskHPSDR. La porta TCI non va esposta oltre LAN/WireGuard.
- **Test di regressione.** Un client TCI che negozia permessage-deflate e invia ripetutamente frame type 2 da 16448 byte di float32 a zero (circa 23 al secondo, per minuti) deve lasciare deskHPSDR vivo: stesso PID, porta TCI che accetta ancora connessioni. Variare la lunghezza dei frame per spostare il confine del buffer da 1024 byte. Il test non richiede di andare in TX, perche' il crash avviene prima del parsing TCI.
- **La ricetta di diagnosi vale per qualunque SIGABRT di deskHPSDR:** core dump, poi thread che abortisce, poi `__abort_msg`. glibc stampa lo stesso messaggio anche su stderr prima di `abort()`. Se deskHPSDR gira con stderr catturato (journal o file), la riga e' disponibile anche senza core.

## Related Issues

- `iu3qez/deskhpsdr`, `docs/plans/2026-09-08-2215-feat-tci-remote-spectrum-extensions-plan.md` (KTD9): la decisione che ha introdotto permessage-deflate nel server.
- `iu3qez/deskhpsdr`, `documentation/deskHPSDR_TCI_Remote_Extensions.md` §5: specifica del permessage-deflate, senza vincolo sulla versione di libwebsockets.
- `docs/plans/2026-09-16-0316-feat-totw-modular-operable-client-plan.md` (KTD13, U12): il lavoro sul keepalive dipende dalla versione di libwebsockets sull'host di build della stazione.
