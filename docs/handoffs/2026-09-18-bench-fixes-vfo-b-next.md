---
artifact_contract: "ce-handoff/v1"
created_at: "2026-09-18T20:38:54Z"
title: "TOTW — correzioni dal banco di riferimento, VFO B e split progettati"
summary: "Banco ridotto sulla build di deskHPSDR del 2026-09-18, sei correzioni del client, quattro issue lato server, PR #17 in bozza; VFO B e split decisi ma non implementati, correzione del modo da riprovare al banco."
keywords: ["totw", "deskhpsdr", "banco", "checklist", "band_ex", "modo", "filtro", "cw", "vfo-b", "split", "pr-17"]
cwd: "/Users/sf/Developer/thetis-on-the-web/.claude/worktrees/handoff-riprendere-48722d"
resume_focus: "Esito al banco della correzione modo e filtro, poi implementazione di VFO B e split (#15) con le decisioni gia' prese"
repository: "iu3qez/thetis-on-the-web"
repo_root_sha: "a14fbec33d24418421b76761f4bbfe888d78bbb0"
branch: "feat/totw-modular-operable-client"
head: "c69f464"
worktree_path: "/Users/sf/Developer/thetis-on-the-web/.claude/worktrees/handoff-riprendere-48722d"
---

# Handoff — TOTW, correzioni dal banco e VFO B in progettazione

**Sostituisce `docs/handoffs/2026-09-16-bench-u1-lws-fix.md`** per lo stato del lavoro. Quello resta valido per U1, il crash di libwebsockets e le voci aperte della revisione del piano.

Il campo `head` e' l'ultimo commit di codice; il commit che aggiunge questo handoff sta sopra di esso sullo stesso branch.

## Intento e decisioni dell'operatore

Dette dall'operatore in questa sessione:

- **Nessun test su storia passata.** Il banco si fa solo sulla release piu' recente di deskHPSDR; gli esiti del 2026-09-16 sono stati tolti dalla checklist.
- **Banco ridotto, poi correzioni.** Eseguite A-C, F e J; D, E, K, L, M e N saltate per scelta dell'operatore ("passiamo a cose serie"). Poi correzione dei difetti trovati, prima di U2.
- **Calibrazione dell'S-meter eliminata del tutto.**
- **Inserimento della frequenza con doppio click ovunque sul VFO**, accettando il ritardo sul click singolo.
- **VFO B e split subito dopo, come lavoro a se'**; gesto scelto: **entrambi**, Shift piu' selezione A/B.
- **Lato server solo issue** nel fork `iu3qez/deskhpsdr`, nessuna correzione diretta.
- Handoff e PR in bozza mentre l'operatore prova al banco.
- Preferenza di comunicazione: niente hash git nelle spiegazioni in chat (salvata in memoria); nei documenti del repository restano.

Mie scelte, non chieste: bande via `band_ex` invece di una memoria nel client; lettura del VFO troncata sotto i 100 Hz; Shift muove sempre **l'altro** VFO rispetto a quello attivo; marker TX mostrato con split acceso invece di confrontare frequenze; lettura di ripiego dell'S-meter da IQ con l'offset di default come costante; una issue per difetto; questo handoff in `docs/handoffs/` come i precedenti.

## Stato

| Pezzo | Stato |
|---|---|
| `main` | invariato a `c154e66` |
| `feat/totw-modular-operable-client` | pushato; in locale lavorato come `claude/bench-u1-lws-fix-90073c` con upstream sul branch remoto |
| PR #17 | bozza verso `main`, descrive tutto il branch |
| Checklist `docs/bench/reference-checklist.md` | A-C, F, J compilate sulla build del 2026-09-18; B1, C1, C4, C6, F6 riprovate dopo le correzioni |
| Correzioni bande, lettura VFO, inserimento, rotella sul waterfall, CAL | fatte e riprovate al banco: OK |
| Correzione modo e filtro (`c69f464`) | fatta, provata con server finto; **da riprovare al banco** (E1, E2, e il "glitch" segnalato dall'operatore, forse il pulsante che si spegneva) |
| VFO B e split (#15) | decisioni prese, **niente codice** |
| U2 e seguenti | non iniziate |

## Difetti lato server aperti in questa sessione

Tutti in `iu3qez/deskhpsdr`, con righe a `3379cb8`:

- #17 CWL e CWU riportati come `CW`: proposta `modulation_ex`. Finche' manca, un cambio CWL/CWU dalla GUI puo' mostrare la banda laterale sbagliata in TOTW.
- #18 il broadcast di `rx_filter_band` inviato subito dopo `modulation` e' normalizzato col modo vecchio. Il filtro applicato e' giusto, perche' `ext_rx_filter_update` rinormalizza sul main loop.
- #19 il cambio di modo non manda ai client il filtro del nuovo modo: il client lo interroga dopo l'eco di `modulation`.
- #20 `nfm` rifiutato: il client manda `FM`.

Gia' esistenti: #11 split, chiusa e presente nella build; #12 `rx_enable`, aperta.

## VFO B e split: cosa si sa e cosa e' deciso

Fatti verificati nel sorgente di deskHPSDR (`src/tci.c`):

- `split_enable:0,true` = TX su VFO B, assoluto anche con RX2 attivo (correzione della #11 in `tci_apply_split_update`).
- `tx_frequency:<Hz>` e' inviato a ogni `tci_vfos_changed` ed e' la frequenza del VFO di TX (`vfo_get_tx_freq` in `src/vfo.c`), senza XIT ne' offset CW: senza split coincide con A.
- VFO B si imposta con `vfo:0,1,<Hz>`; con due ricevitori e' anche la frequenza di RX2.

Nel client (`totw.html`, righe a `c69f464`):

- SPLIT e' un chip nel pannello Options (riga 3627) e manda solo `split_enable`.
- `tx_frequency` e' nella lista dei messaggi silenziati (riga 4253) e non e' gestito.
- Il marker di VFO B esiste solo sullo spettro, tratteggiato grigio (riga 6500); il waterfall disegna le linee di VFO A dopo `putImageData` (riga 6630), il posto dove aggiungere B e TX.
- Click e drag sullo spettro (intorno a riga 8098), click sul waterfall (riga 8277), rotella su spettro e waterfall (`specWheel`, riga 8289) e frecce da tastiera (riga 7904) sintonizzano solo VFO A e mandano `vfo:0,0`.

Deciso:

- Shift+click, drag o rotella su spettro e waterfall muove l'altro VFO; click su un display del VFO lo rende attivo, evidenziato; senza Shift si muove il VFO attivo.
- SPLIT spostato nel pannello VFO accanto ad A→B.
- Marker rosso "TX" a `tx_frequency` su spettro e waterfall quando lo split e' acceso; il marker di B anche sul waterfall, col colore del TX quando e' la frequenza di TX.

Aperto, mia proposta non confermata: il VFO attivo vale solo per la sintonia; bande, modo e filtro restano su A. La rotella con Shift su macOS arriva come `deltaX` con `deltaY` a zero, va gestita. Il ricentraggio dell'IQ e lo zoom seguono solo A.

## La stazione (stato machine-local)

- deskHPSDR su `ubuntu.lan`: `master` a `3379cb8`, compilato il 2026-09-18 alle 20:04, libwebsockets statica `d7f7fdeaf`, TCI 50001.
- Il branch `fix/tci-trx-owner-race` (`d2de1c9`, proprieta' del TX con MOX o TUNE in coda) esiste **solo sulla stazione**: non e' su GitHub e non e' nella build.
- Binari di backup non tracciati in `~/deskhpsdr/`: `deskhpsdr.4752ddc-lws5`, `deskhpsdr.4ba2797-lws5`, `deskhpsdr.eaec707-lws5`.

## Come si e' verificato

- Server statico: `.claude/launch.json` del worktree (non tracciato), `python3 -m http.server 8765`, avviato con il pannello browser dell'app.
- Nel browser integrato una classe finta sostituisce `WebSocket` e risponde come deskHPSDR secondo il sorgente: `band_ex` da una band stack, cambio di modo in differita con filtro per modo, `CW` e `FM` come nomi di risposta, nessun filtro dopo il cambio di modo. Lo script stava nello scratchpad della sessione, che non sopravvive: va riscritto. Il `wsUrl` salvato dal browser integrato puo' puntare all'indirizzo finto.
- Dopo ogni modifica a `totw.html`: `node scripts/build-variants.js split` e `node scripts/smoke-test.js`.

## Trappole verificate

- **Il browser integrato non va collegato alla stazione vera** mentre l'operatore e' al banco.
- **Il branch di lavoro e' aperto anche nel worktree `aerokeyer-rp2040-analysis-f77c92`**, nome fuorviante, ora indietro rispetto al remoto: li' serve un `git pull` prima di usarlo.
- **Push**: con nomi diversi fra locale e remoto serve `git push origin HEAD:feat/totw-modular-operable-client`.
- **zsh**: `$R:src/...` viene letto come modificatore `:s`; scrivere `${R}:src/...`.
- **Checklist**: le righe da compilare finiscono con `| |`; le modifiche fatte con uno script Python e un'espressione regolare per ID sono state le piu' sicure.
- Restano valide le trappole degli handoff precedenti: `gh` sempre con `-R`, niente riferimenti in chiaro al progetto originale.

## Passi successivi plausibili

Un percorso, in ordine:

1. Esito al banco della correzione modo e filtro, da registrare nella checklist (sezione E).
2. VFO B e split con le decisioni sopra, un commit per parte, provati col server finto e poi al banco (C5, F7, F8 della checklist).
3. U2, le rimozioni, confrontando con la checklist.

Indipendenti: le issue #17-#20 lato server; il destino del branch `fix/tci-trx-owner-race` sulla stazione.
