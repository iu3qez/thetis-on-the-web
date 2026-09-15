---
artifact_contract: "ce-handoff/v1"
created_at: "2026-09-15T19:11:17Z"
title: "Thetis On The Web — dopo il banco di C1 e C2, main tronco"
summary: "C1 e C2 passano il banco contro deskHPSDR su ubuntu.lan e le issue #1-#3 sono chiuse; main e' il tronco con STRATEGY.md; l'operatore giudica il client lontano dall'essere completo, con molti problemi aperti elencati qui."
keywords: ["totw", "deskhpsdr", "banco", "c1", "c2", "c3", "strategy", "triage", "ubuntu-lan"]
cwd: "/Users/sf/Developer/thetis-on-the-web/.claude/worktrees/handoff-riprendere-48722d"
resume_focus: "Ripresa con contesto pulito: decidere fra triage dei problemi del client e prosecuzione del piano con C3"
repository: "iu3qez/thetis-on-the-web"
repo_root_sha: "a14fbec33d24418421b76761f4bbfe888d78bbb0"
branch: "main"
head: "691dde8"
worktree_path: "/Users/sf/Developer/thetis-on-the-web/.claude/worktrees/handoff-riprendere-48722d"
---

# Handoff — Thetis On The Web, dopo il banco di C1 e C2

**Sostituisce `docs/handoffs/2026-09-12-c1-merged-c2-next.md`.** Quello descriveva C2 come non iniziata e `main` come copia dell'originale: entrambe le cose non sono piu' vere.

## Valutazione dell'operatore

Parole dell'operatore, 2026-09-15: «ci sono un mare di problemi su TOTW. E' tutto tranne che un progetto completo.» Ha chiesto di chiudere le issue chiudibili e di riprendere con contesto pulito. Non ha indicato il prossimo lavoro.

## Stato

| Pezzo | Stato |
|---|---|
| `main` | tronco, `691dde8`, contiene tutto il lavoro e `STRATEGY.md` |
| Issue #1 ricognizione, #2 C1, #3 C2 | chiuse il 2026-09-15, con le misure nei commenti di chiusura |
| Issue #4..#8, cioe' C3..C7 | aperte, non iniziate |
| `plan/deskhpsdr-client` | a riposo, contenuto identico a `main`, non cancellato |

C1 sta nella PR #12, C2 e la correzione della dissolvenza audio nella PR #13, l'integrazione in `main` nella PR #14.

## Riferimenti autorevoli

- `STRATEGY.md` — decide cosa e' in strategia. Copre solo il client, dentro la stazione remota. Posizionamento: si possiedono entrambi i capi del collegamento, e il client e' best effort perche' audio e morse passano altrove. I confini escludono Thetis e altri server TCI, iOS e tablet, audio e morse nel browser, il multi-operatore in questa fase, investimenti sulle funzioni ereditate che chiamano servizi esterni, e qualsiasi riferimento al progetto originale. Due tracce: spettro su banda stretta e pannello fisico MIDI.
- `docs/plans/2026-09-09-feat-deskhpsdr-client-plan.md` — le unita' da C3 a C7 con approccio e scenari. **I numeri di riga su `totw.html` sono superati** dalle modifiche di C1 e C2. La verifica al banco numero 5 chiede ancora la regressione con Thetis, in contrasto con la strategia.
- Requisiti della stazione, **fuori da questo repository**: `docs/brainstorms/2026-09-08-remote-station-requirements.md` nel repository `iu3qez/deskhpsdr`, checkout locale in `/Users/sf/Developer/deskhpsdr` (machine-local). E' la fonte sul fine ultimo; il requisito SRV-10 fissa la porta TCI a 40001.
- `docs/solutions/best-practices/falsifiable-acceptance-criteria-in-plans.md` — tre criteri del piano che non potevano falsificare, e perche'. Utile prima di scrivere i criteri di C3.
- `CONCEPTS.md` — vocabolario del dominio: unita' d'implementazione, ricognizione, verifica al banco, varianti portabile e scomposta, sorgente spettro, liveness della traccia.

## Banco del 2026-09-15

Client servito da `main`, collegato a deskHPSDR su `ubuntu.lan`. Tutti i criteri di C1 passati; per C2 il default del rate funziona, perche' la radio e' passata da 192 a 48 kHz senza toccare nulla. Su 30 s nessuna discontinuita' audio nuova, audio a 93,7 buffer al secondo. Misure complete nei commenti di chiusura di #2 e #3. **Non fatta**: la prova a orecchio del ronzio su una portante CW.

## La stazione

- **Host**: deskHPSDR gira su `ubuntu.lan`, 192.168.1.35, non sul Mac. Accesso SSH disponibile tramite l'alias `ubuntu` del server SSH configurato nell'harness.
- **Porta TCI 50001, non 40001.** La configurazione in `~/.config/deskhpsdr/68-27-19-99-35-57.props` sull'host imposta `tci_port=50001`, mentre il sorgente di deskHPSDR, il client e SRV-10 usano 40001. Il TCI ascolta su tutte le interfacce, `tci_bind_addr` vuoto; il rigctl sulla 19090. Allinearla e' una decisione dell'operatore, da fare dal menu CAT/TCI a server spento, perche' deskHPSDR riscrive il file props all'uscita.
- **Radio**: ultima osservazione a 48 kHz in CW su 7,022 MHz, dopo il banco. Prima era a 192 kHz. Stato attuale non verificato.

## Problemi noti del client, osservati e non toccati

Materiale per un eventuale triage, non un piano:

- **S-meter**: deskHPSDR manda `rx_sensors`, che il client non riconosce; il log si riempie di «unknown TCI» e l'S-meter ripiega sullo spettro IQ.
- **Host di default** `127.0.0.1`, mentre la stazione e' remota: l'indirizzo va scritto a ogni connessione e non viene ricordato.
- **Testo del pannello spettro** fermo a «Connect to Thetis to activate. 192 kHz wideband IQ spectrum».
- **Riferimenti al progetto originale** nel client, cioe' controllo aggiornamenti e badge di release, e nel README, badge e link per donazioni. Il clone locale ha ancora il remote `upstream`.
- **Resti di Thetis** in contrasto con la strategia: il ramo per l'header legacy da 8 byte e il messaggio di aiuto che cita la porta 50001.
- **`IQ.fftReady` non si invalida**: se il flusso IQ si ferma, spettro, waterfall e S-meter restano congelati sull'ultimo frame. Registrato fra i rinviati del piano, senza unita'.
- **Autoconnessione** attiva piu' rate di default: il caricamento della pagina riconfigura il rate della radio senza azioni dell'operatore.
- **Ritardo audio che cresce nel tempo**: lo scheduler accumula latenza; fuori scopo, perche' l'audio remoto passa da Mumble.
- **Lato server**, da portare nel repository di deskHPSDR: `audio_stream_channels:1` nel dump di testo contro `channels = 2` nell'header binario.

## Decisioni, e di chi sono

Dell'operatore: le risposte a `STRATEGY.md`; `main` come tronco, dopo il riallineamento; il default del rate a 48 kHz; la correzione della dissolvenza fuori dalle unita'; nessun riferimento al progetto originale; la chiusura delle issue chiudibili.

Mie: chiudere la #3 nonostante la porta della stazione, perche' il client fa cio' che la issue chiede e la porta e' configurazione; spingere questo handoff direttamente su `main`, come i commit di documentazione precedenti sul ramo d'integrazione.

## Trappole verificate

- **`gh` senza `-R iu3qez/thetis-on-the-web`** puo' risolvere la base verso il repository originale, perche' il clone ha due remote e nessun default.
- **Un riferimento `owner/repo#N` in chiaro** in un messaggio di commit o nel corpo di una PR crea un evento visibile sulla issue di quel repository. Scriverlo in codice inline, o non scriverlo.
- **La sandbox dei comandi blocca la LAN**: da li' `ubuntu.lan` da' «No route to host». Le prove di rete locale vanno fatte fuori sandbox.
- **La ripresa di sessione ha riportato il worktree** sul branch iniziale al commit dell'originale, e ha svuotato lo scratchpad. Controllare branch e HEAD prima di servire il client.
- **`website/` e' generata**: dopo ogni modifica a `totw.html` eseguire `node scripts/build-variants.js split` e `node scripts/smoke-test.js`.

## Passi successivi plausibili

Due strade alternative, la scelta e' dell'operatore:

1. **Triage dei problemi del client**: partire dall'elenco sopra e dalla valutazione dell'operatore, confrontarlo con `STRATEGY.md`, e decidere cosa correggere, cosa togliere e cosa ignorare prima di aggiungere funzioni.
2. **Proseguire il piano con C3**, issue #4, lo stream dello spettro a bin, che apre la traccia principale della strategia.

Indipendente da entrambe: allineare la porta TCI della stazione a 40001.
