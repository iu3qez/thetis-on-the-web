# Checklist di riferimento del client

Base di confronto per le rimozioni (U2) e per il primo spostamento nella build (U3) del piano `docs/plans/2026-09-16-0316-feat-totw-modular-operable-client-plan.md`.

Si esegue al banco sulla build indicata, con la radio accesa e il TX su carico fittizio. Si compila la colonna **Esito** con `OK`, `KO` o una nota. Dopo ogni rimozione o spostamento la stessa checklist deve dare lo stesso esito, con una sola eccezione: le voci della sezione N possono passare a "assente".

## Intestazione dell'esecuzione

| Campo | Valore |
|---|---|
| Commit del client | `fd0a96f` |
| Browser e versione | Firefox, ultima versione per macOS al 2026-09-18 |
| Apertura | file:// (A1), poi server locale `http://127.0.0.1:8765/totw.html` |
| Stazione | `ubuntu.lan`, TCI 50001 |
| Build di deskHPSDR | `master` a `3379cb8` (merge di upstream del 2026-09-18), compilata il 2026-09-18 alle 20:04, libwebsockets statica `d7f7fdeaf`; senza il branch locale `fix/tci-trx-owner-race` |
| Data | 2026-09-18 |

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
| B1 | Click su 40m, poi 20m | VFO e modo cambiano anche nella GUI di deskHPSDR | Parziale: VFO e modo seguono; tornando su 40m (40, 20, di nuovo 40) la frequenza usata prima su 40m non viene ripristinata, il client non ha memoria per banda |
| B2 | Click su ⚙, chiudere; click su DIAG, chiudere; click su ? , chiudere | Il VFO non cambia | OK |

## C. Sintonia

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| C1 | Rotella del mouse sulle cifre del VFO, uno scatto | Un passo, radio allineata | OK; formato della frequenza poco leggibile: MHz con quattro decimali attaccati (7.1234), senza stacco dopo la cifra dei kHz; atteso per esempio 7.123 4 |
| C2 | Dopo B2, rotella sulle cifre del VFO | Un passo per scatto, console senza errori | OK |
| C3 | Click e click destro su una cifra del VFO | Incremento e decremento di quella cifra | OK |
| C4 | Doppio click sul VFO, scrivere una frequenza, Invio | Radio sulla frequenza scritta | Parziale: l'inserimento si apre solo con doppio click sulla scritta MHz; sulle cifre il doppio click le incrementa o decrementa |
| C5 | A→B, B→A, A⇌B | VFO scambiati come indicato, anche in GUI | KO di usabilita': l'operatore non trova come selezionare VFO B. Il client non ha un VFO attivo; VFO B si sintonizza solo dalle sue cifre grigie sotto A→B (issue #15) |
| C6 | Rotella sullo spettro | Sintonia a passi, lo spettro segue | OK sullo spettro; sul waterfall la rotella non sintonizza, da aggiungere per coerenza |
| C7 | Click e drag sullo spettro | Sintonia alla frequenza sotto il cursore | OK |
| C8 | Click sul waterfall | Sintonia alla frequenza sotto il cursore | OK |
| C9 | Frecce su e giu' da tastiera | Un passo per pressione | OK |
| C10 | Trackpad: swipe a due dita sul VFO | Annotare quanti passi produce uno swipe (difetto noto, riferimento per U7) | Diversi passi per swipe, non contati |

## D. Zoom

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| D1 | Ctrl+rotella sullo spettro | Zoom in e out centrato sul cursore | |
| D2 | Pinch sul trackpad sullo spettro | Annotare il comportamento (difetto noto, riferimento per U7) | |
| D3 | Doppio click sullo spettro | Zoom azzerato | |

## E. Modo e filtro

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| E1 | Selezionare LSB, USB, CWL, CWU, AM | Modo cambiato anche in GUI | |
| E2 | Selezionare tre larghezze di filtro | Filtro cambiato anche in GUI | |
| E3 | Impostare LO e HI a mano, SET | Filtro personalizzato applicato | |

## F. Controlli RX

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| F1 | NR1, poi NR spento | NR acceso e spento in GUI (il tipo di NR puo' non seguire: difetto noto) | |
| F2 | ANF acceso e spento | ANF segue in GUI | |
| F3 | Slider AF | Volume RX1 segue in GUI | |
| F4 | Slider AF di RX2, se RX2 esiste | Volume RX2 segue in GUI | |
| F5 | Segnale forte e banda silenziosa | S-meter segue il segnale | |
| F6 | Slider CAL dell'S-meter | Lettura spostata dell'offset | |
| F7 | Split acceso e spento dalla GUI di deskHPSDR | Il pulsante SPLIT di TOTW segue | |
| F8 | SPLIT da TOTW | Split segue in GUI | |

## J. Trasmissione, su carico fittizio

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| J1 | Impostazioni: timeout PTT a 1 minuto, salvare | Valore salvato | |
| J2 | PTT momentaneo: tenere premuta la barra spaziatrice 3 s, rilasciare | TX per 3 s, poi RX; un solo comando `trx` di accensione nel log | |
| J3 | PTT con il pulsante a video | TX e RX seguono il pulsante | |
| J4 | TUNE acceso, attendere | TX di accordo, spento automaticamente dopo 1 minuto con riga di log | |
| J5 | TUNE acceso e spento a mano | Segue subito | |
| J6 | Slider DRIVE | Potenza di pilotaggio segue in GUI | |
| J7 | Riportare il timeout PTT a 3 minuti | Valore salvato | |

## K. Audio TCI

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| K1 | RX AUDIO acceso | Audio della radio nel browser | |
| K2 | RX AUDIO spento | Audio fermo | |

## L. Interfaccia

| ID | Azione | Atteso | Esito |
|---|---|---|---|
| L1 | Cambiare tema UI | Tema applicato | |
| L2 | Spostare e chiudere un pannello dock, ricaricare | Disposizione ricordata | |
| L3 | Applicare un preset di layout | Layout applicato | |
| L4 | Guadagno spettro, velocita' waterfall, smoothing, peak hold | Effetto visibile | |

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
