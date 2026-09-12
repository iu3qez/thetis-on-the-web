---
artifact_contract: "ce-handoff/v1"
created_at: "2026-09-12T01:11:01Z"
title: "TOTW per deskHPSDR — C1 in merge, si riprende da C2"
summary: "C1 e' implementata e mergiata su plan/deskhpsdr-client, la ricognizione e' chiusa, il passo successivo e' C2 piu' la verifica al banco di C1 che attende la radio."
keywords: ["totw", "deskhpsdr", "tci", "c1", "c2", "dispatch-frame-binari", "iq-samplerate"]
cwd: "/Users/sf/Developer/thetis-on-the-web/.claude/worktrees/handoff-riprendere-48722d"
resume_focus: "C2 (issue #3), e la verifica al banco di C1 quando la radio e' libera"
repository: "iu3qez/thetis-on-the-web"
repo_root_sha: "a14fbec33d24418421b76761f4bbfe888d78bbb0"
branch: "plan/deskhpsdr-client"
head: "51f9adee8887e1c1baadf990d87fbc4cc6e62786"
worktree_path: "/Users/sf/Developer/thetis-on-the-web/.claude/worktrees/handoff-riprendere-48722d"
---

# Handoff — TOTW per deskHPSDR, dopo C1

**Sostituisce `docs/handoffs/2026-09-12-totw-handoff.md`**, che resta in archivio ma e' superato su un punto sostanziale: diceva nessuna riga di codice scritta, e non e' piu' vero. Se entrambi compaiono in una ricerca, questo e' il piu' recente.

Riguarda solo il client. Il lavoro sul server deskHPSDR ha un handoff proprio nel suo repository e non va mescolato con questo.

## Stato

Branch autorevole **`plan/deskhpsdr-client`**, punta `51f9ade`. `main` resta allineato a `upstream` n9bc a `c147edf`, senza nostre modifiche: e' un invariante del fork, non un caso.

| Pezzo | Maturita' |
|---|---|
| Ricognizione (issue #1) | chiusa, vedi sotto |
| C1 dispatch dei frame binari (#2) | implementata e mergiata, **non verificata al banco** |
| C2 sample rate IQ (#3) | non iniziata, e' il prossimo passo |
| C3..C7 (#4..#8) | non iniziate |

C1 e' il commit `e41121a`, PR #12, merge `c29d5d2`. Gli emendamenti al piano sono `51f9ade`.

## Riferimenti autorevoli

- `docs/plans/2026-09-09-feat-deskhpsdr-client-plan.md` — la fonte sul meccanismo. Leggere CTD7 e CTD8 prima di toccare codice: il primo dice che `website/` e' generata e non si modifica a mano, il secondo che sul formato del payload audio si crede all'header binario e non al dump di testo del server. Lo stato di C1 e' in testa alla sua sezione.
- `iu3qez/deskhpsdr`, `documentation/deskHPSDR_TCI_Remote_Extensions.md` — contratto del frame `type=4`, congelato, non negoziabile dal client.
- `totw.html` — `dispatchBinaryFrame()` e `rxAudioFrame()` sono il risultato di C1. C2 deve toccare le tre `send('iq_samplerate:192000;')`, di cui una dentro `startIQ()`, piu' il campo host e il messaggio di aiuto; il piano elenca le righe.
- `iu3qez/deskhpsdr`, `src/tci.c` — `tci_cmd_iq_samplerate()` e `tci_iq_effective_sample_rate()` sono il cuore del comportamento che C2 deve smettere di innescare.

## Cosa ha prodotto la ricognizione, e come si e' chiusa

La condizione di stop emendata chiedeva di fermarsi se, con la radio a 48 kHz, il pannello IQ disegnava comunque. **E' stata chiusa sul sorgente invece che al banco**, perche' la catena e' a produttore unico: l'unico chiamante di `handleIQFrame` era la condizione `srAt4 > 48000`, l'unico chiamante di `runIQFFT` e' `handleIQFrame`, e solo `runIQFFT` scrive `IQ.fftResult`. Con la radio a 48 kHz la traccia non poteva aggiornarsi, per costruzione.

Attenzione a non ripetere l'errore che quella condizione conteneva: **`IQ.fftReady` non torna falso da se'**, quindi il pannello si congela invece di svuotarsi e disegna lo stesso. Il discriminante giusto e' `IQ.frameCount` che avanza. Il difetto e' registrato nel piano sotto Deferred, non assegnato a nessuna unita'.

La domanda su `vfoA: null` e' chiusa, non era una issue.

**Correzione del 2026-09-12, piu' tarda del resto di questo documento.** Dove sopra si leggeva che restava da raccogliere il clic all'ascolto: non resta niente, ed e' stato chiuso per misura invece che per ascolto. Il clic non dipende dall'offset del payload, cioe' da cio' che C1 corregge: lo produce la dissolvenza di 64 campioni che `playFloat32Stereo` applica ai bordi di ogni buffer, che attenua un quarto del buffer 93,75 volte al secondo, cioe' alla cadenza dei buffer. Replicando la funzione, la riga di modulazione resta identica con e senza la correzione di C1 e scompare solo togliendo la dissolvenza. Ne seguono tre cose: **C1 non elimina il ronzio** e il corpo della PR #12 afferma il contrario, quindi e' sbagliato; l'attribuzione della issue #12 a monte all'header e' da considerare non dimostrata; e la dissolvenza e' un difetto noto che nessuna unita' tocca, registrato fra i rinviati nel piano. Il ragionamento per esteso sta in `docs/solutions/best-practices/falsifiable-acceptance-criteria-in-plans.md`.

## Decisioni, e di chi sono

Dell'operatore, prese in sessione:

- Riportare la radio a 48 kHz via TCI dal client dell'assistente, invece di farlo dalla GUI.
- Saltare la misura del clic e passare direttamente a C1.
- Fare la PR, e mergiarla.
- Nessuna segnalazione a monte su n9bc, decisione precedente confermata: non si disturba un collega per una svista del suo assistente.

Mie, prese senza chiedere perche' reversibili o di routine:

- Lavorare su branch nuovi creati **a partire da** `plan/deskhpsdr-client` dentro il worktree, invece di fare reset del branch del worktree. Nulla e' andato perso.
- Rimuovere `tryRawAudio()` invece di lasciarla inerte: dopo C1 non aveva chiamanti, e tenerla definita invitava a rifare l'errore.
- Base della PR su `plan/deskhpsdr-client` e non su `main`, per non violare l'invariante su `main`.
- `Related: #2` invece di `Fixes #2`: le parole magiche chiudono solo puntando al branch predefinito. **La issue #2 e' ancora aperta, volutamente**: il criterio di accettazione di C1 e' la verifica al banco, che non e' stata fatta.
- Commit degli emendamenti al piano spinto diretto su `plan/deskhpsdr-client`, come i nove commit di documentazione precedenti, invece di aprire una seconda PR.

## Trappole verificate, da non ripetere

- **`gh` risolve la base verso monte.** `origin` e' il fork, `upstream` e' n9bc, e non c'e' un repository predefinito impostato. Senza `-R iu3qez/thetis-on-the-web` esplicito una PR puo' finire in casa di n9bc, che e' esattamente cio' che l'operatore ha deciso di non fare.
- **Abbassare il rate IQ via TCI non funziona se un altro client riceve IQ.** Il server coercia la richiesta al rate dello stream attivo e risponde con quello, senza errore: `iq_samplerate:48000;` ha ricevuto `192000`. `iq_stop` e' per client, quindi non si puo' fermare lo stream di qualcun altro. Serve che il proprietario dello stream smetta.
- **Il lock `TCI_SET_LOCK_IQ_RATE` non e' una mitigazione.** E' una mutua esclusione da 200 ms fra client concorrenti. Io l'avevo letto come difesa lato server e mi sono corretto: non esiste difesa lato server contro il difetto di C2.
- **Modificare `website/assets/totw.js` a mano si perde.** `scripts/build-variants.js` gira nei due versi e `split` considera il portabile la verita'.
- **Il commento a `totw.html` sull'header non standard da 8 byte era falso** e ora non c'e' piu'. Non reintrodurre euristiche sulla dimensione del frame.

## Verifiche fatte, e quella che manca

Fatte su `e41121a`: gli scenari di dispatch del piano su frame sintetici, tutti passati; `node scripts/smoke-test.js`; il parsing dello script inline con `node --check`; il caricamento della pagina in browser con console vuota e le funzioni nuove presenti nello scope.

**Manca la verifica al banco**, che e' l'accettazione vera, nella forma corretta il 2026-09-12: con la radio a 48 kHz il pannello IQ disegna e `IQ.frameCount` avanza, il contatore dei frame di tipo ignoto resta a zero, e il primo frame audio nel log mostra il payload letto dall'offset 64. Non aspettarsi la scomparsa del ronzio: non dipende da C1.

## Blocchi e stato locale fragile

- **La radio e' a 192 kHz** e lo resta finche' il TOTW aperto in Firefox non ferma l'IQ o non si disconnette, perche' ne e' il proprietario.
- **Il checkout principale era indietro di due commit** rispetto a `origin/plan/deskhpsdr-client`, quindi il server HTTP che serve da lui mostrava codice pre-C1. Da aggiornare prima di qualunque prova in Firefox.
- Stato **machine-local**, non riproducibile altrove: il worktree `/Users/sf/Developer/thetis-on-the-web/.claude/worktrees/handoff-riprendere-48722d`, un `python3 -m http.server` sulla 8000 che serve il checkout principale, un altro sulla 8899 avviato per il controllo in browser del worktree, e `deskhpsdr` in esecuzione con il TCI su `127.0.0.1:40001`. Nessuna di queste cose sopravvive a un riavvio.
- Branch locali ormai inutili nel worktree: `c1/dispatch-binary-frames`, mergiato, e `docs/plan-amendments-c1`, il cui commit e' gia' sulla punta.

## Passi successivi plausibili

Un percorso unico, in sequenza, non alternative: **C2 (#3)**, poi la **verifica al banco di C1 e C2 insieme** quando la radio e' libera, poi **C3 (#4)**. C2 va fatta subito perche' C1 da sola lascia in piedi il comando che porta la radio a 192 kHz alla connessione, e la verifica conviene unica perche' i due difetti si mascheravano a vicenda.

Due cose indipendenti, che non bloccano nulla:

- Decidere se e dove correggere la dissolvenza per buffer di `playFloat32Stereo`, che e' il vero ronzio. Confina con CLI-08, lo scheduler audio, che DEC-01 mette fuori scopo: per questo la decisione non e' automatica.
- Decidere dove vive la correzione di `IQ.fftReady`: dentro C5, che gia' tocca avvio e arresto degli stream, oppure in un'unita' propria.
- Portare nel repository di deskHPSDR la contraddizione sui canali audio, `audio_stream_channels:1` nel testo contro `channels = 2` nell'header. E' un difetto del server e appartiene al suo handoff.
