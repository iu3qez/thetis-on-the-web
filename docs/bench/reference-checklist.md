# Checklist di riferimento del client

Base di confronto per le rimozioni (U2) e per il primo spostamento nella build (U3) del piano `docs/plans/2026-09-16-0316-feat-totw-modular-operable-client-plan.md`.

Si esegue al banco sulla build indicata, con la radio accesa e il TX su carico fittizio. Si compila la colonna **Esito** con `OK`, `KO` o una nota. Dopo ogni rimozione o spostamento la stessa checklist deve dare lo stesso esito, con una sola eccezione: le voci della sezione N possono passare a "assente".

## Intestazione dell'esecuzione

| Campo | Valore |
|---|---|
| Commit del client | `fd0a96f`; B1, C1, C4, C6 e F6 riprovate su `941ee93`; E1-E4, F9 e F10 su `4fbac98`; C5, C11-C14, D1-D3, E5, E6, F7, F8, F11-F13, K1, K2 e L1-L4 su `eaab31d` |
| Browser e versione | Firefox, ultima versione per macOS al 2026-09-18 e al 2026-09-19 |
| Apertura | file:// (A1), poi server locale `http://127.0.0.1:8765/totw.html`; C5, C11-C14, D1-D3, E5, E6, F7, F8, F11-F13, K1, K2 e L1-L4 da file:// |
| Stazione | `ubuntu.lan`, TCI 50001 |
| Build di deskHPSDR | `master` a `3379cb8` (merge di upstream del 2026-09-18), compilata il 2026-09-18 alle 20:04, libwebsockets statica `d7f7fdeaf`; senza il branch locale `fix/tci-trx-owner-race`. E1-E4, F9 e F10 su `master` a `283cf6d` (merge della PR #25, con le PR #21, #22 e #23), compilata il 2026-09-18 alle 23:30. C5, C11-C14, D1-D3, E5, E6, F7, F8, F11-F13, K1, K2 e L1-L4 su `master` a `31250e8` (merge della PR #32, con le PR #26, #29 e #30), compilata il 2026-09-19 alle 04:43 |
| Data | 2026-09-18 e 2026-09-19 |

## A. Apertura e connessione

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| A1 | Aprire `totw.html` da file:// | Pagina completa, console senza errori | OK |
| A2 | Aprire `totw.html` da server locale | Pagina completa, console senza errori | OK |
| A3 | Scrivere host e porta della stazione, CONNECT | Stato connesso; VFO, modo e filtro mostrano i valori della radio | OK |
| A4 | Attendere 10 s connessi | Spettro e waterfall IQ disegnano; il log TCI non si riempie di righe periodiche | OK |
| A5 | DISCONNECT, poi CONNECT | Riconnessione riuscita, stessi valori della radio | OK |

## B. Bande

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| B1 | Click su 40m, poi 20m | VFO e modo cambiano anche nella GUI di deskHPSDR | OK dopo la correzione: le bande passano per `band_ex` e tornano all'ultima frequenza, modo e filtro. Prima: la frequenza precedente sulla banda si perdeva |
| B2 | Click su ⚙, chiudere; click su DIAG, chiudere; click su ? , chiudere | Il VFO non cambia | OK |

## C. Sintonia

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| C1 | Rotella del mouse sulle cifre del VFO, uno scatto | Un passo, radio allineata | OK dopo la correzione: `7.123 4 MHz`. Prima: quattro decimali attaccati, e arrotondamento sbagliato vicino al cambio di MHz |
| C2 | Dopo B2, rotella sulle cifre del VFO | Un passo per scatto, console senza errori | OK |
| C3 | Click e click destro su una cifra del VFO | Incremento e decremento di quella cifra | OK |
| C4 | Doppio click sul VFO, scrivere una frequenza, Invio | Radio sulla frequenza scritta | OK dopo la correzione: doppio click ovunque sul VFO; il click singolo agisce dopo 250 ms. Prima: si apriva solo sulla scritta MHz |
| C5 | VFO A e VFO B su bande e modi diversi; A→B, B→A, A⇌B | Si sposta tutto il VFO, anche in GUI: frequenza, banda, modo e filtro. I pulsanti di banda e di modo seguono VFO A; accanto a VFO B compare il suo modo | OK |
| C6 | Rotella sullo spettro | Sintonia a passi, lo spettro segue | OK, spettro e waterfall; sul waterfall aggiunti dopo la prima esecuzione sintonia e zoom con la rotella |
| C7 | Click e drag sullo spettro | Sintonia alla frequenza sotto il cursore | OK |
| C8 | Click sul waterfall | Sintonia alla frequenza sotto il cursore | OK |
| C9 | Frecce su e giu' da tastiera | Un passo per pressione | OK |
| C10 | Trackpad: swipe a due dita sul VFO | Annotare quanti passi produce uno swipe (difetto noto, riferimento per U7) | Diversi passi per swipe, non contati; issue #21 |
| C11 | Drag di VFO A dentro la sua banda passante, circa 50 px, poi rilasciare | VFO A si ferma sotto il cursore, anche in GUI, senza andare oltre il punto di rilascio | OK |
| C12 | Shift+click sullo spettro e sul waterfall | VFO B va alla frequenza sotto il cursore, anche in GUI; VFO A non si muove | OK |
| C13 | Shift+drag sullo spettro | VFO B segue il cursore, anche in GUI; VFO A e la finestra IQ non si muovono | OK |
| C14 | Shift+rotella sullo spettro e sul waterfall (su macOS arriva come scroll orizzontale) | VFO B a passi nei due versi, anche in GUI; VFO A non si muove | OK |

## D. Zoom

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| D1 | Ctrl+rotella sullo spettro | Zoom in e out centrato sul cursore | KO, come D2: ogni evento rotella raddoppia o dimezza lo zoom qualunque sia `deltaY` (`totw.html:8393`); sul trackpad lo zoom va da 1x a 32x in un istante. Lo corregge U7 (KTD9), issue #20 |
| D2 | Pinch sul trackpad sullo spettro | Annotare il comportamento (difetto noto, riferimento per U7) | Uguale a D1: su macOS il pinch arriva come rotella con Ctrl, e ogni evento raddoppia o dimezza lo zoom; ingestibile. Riferimento per U7, issue #20 |
| D3 | Doppio click sullo spettro | Zoom azzerato | OK |

## E. Modo e filtro

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| E1 | Selezionare LSB, USB, CWL, CWU, AM | Modo cambiato anche in GUI | OK |
| E2 | Selezionare tre larghezze di filtro | Filtro cambiato anche in GUI | OK |
| E3 | Impostare LO e HI a mano, SET | Filtro personalizzato applicato | OK |
| E4 | Dalla GUI di deskHPSDR: CWL, CWU, FM, LSB | Il pulsante di modo di TOTW segue, CWL e CWU distinti | OK |
| E5 | Una sola RX, VFO B in un modo diverso da VFO A; DISCONNECT e CONNECT, poi A⇌B con VFO A in CWL | Accanto a VFO B il modo di VFO B della GUI, subito dopo la connessione; dopo A⇌B, CWL e non CWU | OK |
| E6 | RX2 accesa, cambiare il modo di RX2 dalla GUI di deskHPSDR | Il modo accanto a VFO B segue | OK |

## F. Controlli RX

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| F1 | NR1, poi NR spento | NR acceso e spento in GUI (il tipo di NR puo' non seguire: difetto noto) | OK |
| F2 | ANF acceso e spento | ANF segue in GUI | OK |
| F3 | Slider AF | Volume RX1 segue in GUI | OK |
| F4 | Slider AF di RX2, se RX2 esiste | Volume RX2 segue in GUI | OK |
| F5 | Segnale forte e banda silenziosa | S-meter segue il segnale | OK |
| F6 | Slider CAL dell'S-meter | Lettura spostata dell'offset | Assente: slider CAL rimosso il 2026-09-18, non agiva piu' su nessuna lettura |
| F7 | Split acceso e spento dalla GUI di deskHPSDR | Il pulsante SPLIT di TOTW segue | OK |
| F8 | SPLIT da TOTW | Split segue in GUI | OK |
| F9 | RX2 ON da TOTW, acceso e spento | RX2 acceso e spento in GUI; il pulsante cambia solo dopo la risposta della radio | OK |
| F10 | RX2 acceso e spento dalla GUI di deskHPSDR | Il pulsante RX2 ON di TOTW segue | OK |
| F11 | Split acceso, VFO B diverso da VFO A | Linea rossa TX su VFO B nello spettro, con la frequenza di TX, e nel waterfall | OK |
| F12 | Split acceso, Shift+rotella su VFO B | La linea TX segue VFO B | OK |
| F13 | Split spento, poi sintonizzare VFO A con rotella veloce e con drag sullo spettro | Linea TX sparita, anche durante la sintonia; VFO B resta tratteggiato grigio su spettro e waterfall | OK |

## J. Trasmissione, su carico fittizio

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| J1 | Impostazioni: timeout PTT a 1 minuto, salvare | Valore salvato | OK |
| J2 | PTT momentaneo: tenere premuta la barra spaziatrice 3 s, rilasciare | TX per 3 s, poi RX; un solo comando `trx` di accensione nel log | OK |
| J3 | PTT con il pulsante a video | TX e RX seguono il pulsante | OK |
| J4 | TUNE acceso, attendere | TX di accordo, spento automaticamente dopo 1 minuto con riga di log | OK |
| J5 | TUNE acceso e spento a mano | Segue subito | OK |
| J6 | Slider DRIVE | Potenza di pilotaggio segue in GUI | OK |
| J7 | Riportare il timeout PTT a 3 minuti | Valore salvato | OK |

## K. Audio TCI

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| K1 | RX AUDIO acceso | Audio della radio nel browser | OK |
| K2 | RX AUDIO spento | Audio fermo | OK |
| K3 | RX AUDIO acceso su un segnale forte, poi RX AUDIO spento | Audio fermo senza clic | |
| K4 | RX AUDIO acceso su un segnale forte; attendere che la connessione si chiuda da sola (#22, ogni 1,5-9 min) | Nessun clic; nel log TCI una riga rossa `Disconnected: code …` e poi `Reconnecting in 1.0s…`; in DIAG, Reconnects e Last Close aggiornati. Copiare il JSON di DIAG nel #22 | |

## L. Interfaccia

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| L1 | Cambiare tema UI | Tema applicato | OK |
| L2 | Spostare e chiudere un pannello dock, ricaricare | Disposizione ricordata | OK |
| L3 | Applicare un preset di layout | Layout applicato | OK |
| L4 | Guadagno spettro, velocita' waterfall, smoothing, peak hold | Effetto visibile | OK |

## M. Persistenza

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| M1 | Salvare le impostazioni, ricaricare la pagina | Valori mantenuti | |
| M2 | Esportare il backup completo | File scaricato | |
| M3 | Reimportare il backup | Import riuscito, valori mantenuti | |

## N. Funzioni che U2 rimuove

In questa esecuzione di riferimento si annota se funzionano. Dopo U2 possono risultare assenti senza che sia una regressione.

| ID | Funzione | Stato prima di U2 | Esito dopo U2 |
|---|---|---|---|
| N1 | Pannello DX cluster e tasto X | | assente |
| N2 | Greyline sullo spettro | | assente |
| N3 | Marker WSPR/JS8 e tasto D | | assente |
| N4 | Memorie: pannello, tasto M, F1-F4, export e import | | assente |
| N5 | Palette alternative del waterfall | | assente, ne resta una |
| N6 | Badge di versione e controllo aggiornamenti | | assente |
| N7 | Spettro audio quando manca l'IQ e RX AUDIO e' acceso | | assente, il pannello dichiara l'assenza di IQ |
