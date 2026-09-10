---
title: TOTW fork per deskHPSDR - Plan
type: feat
date: 2026-09-09
origin: iu3qez/deskhpsdr docs/brainstorms/2026-09-08-remote-station-requirements.md
contract: iu3qez/deskhpsdr documentation/deskHPSDR_TCI_Remote_Extensions.md
artifact_contract: ce-unified-plan/v1
artifact_readiness: implementation-ready
execution: code
---

# TOTW fork per deskHPSDR - Plan

## Goal Capsule

- **Objective:** l'operatore remoto apre TOTW in un browser dietro WireGuard, vede lo spettro della radio HPSDR con banda sotto i 50 kbit/s, sente audio pulito, e pilota la radio con una console MIDI fisica.
- **Means:** fork di `n9bc/thetis-on-the-web` che consuma le estensioni TCI gia' implementate e congelate nel server deskHPSDR: stream a bin `type=4`, `spectrum_span`, `rx_att_ex`, `band_ex`. Piu' Web MIDI, che non tocca il protocollo.
- **Authority hierarchy:** il documento dei requisiti in `origin` decide il prodotto (CLI-ID); la specifica in `contract` decide il formato dei frame e non e' negoziabile da qui; questo piano decide il meccanismo lato client (CTD); l'implementatore decide i nomi e i dettagli locali entro le unita'.
- **Stop conditions:** fermarsi e chiedere se una modifica richiede di cambiare il formato del frame `type=4`, se una unita' richiede di abbandonare il singolo file vanilla JS, o se il consumo dello spettro a bin richiede modifiche al codice di disegno esistente.
- **Execution profile:** un fork personale, un branch, unita' in ordine di dipendenza. Nessun build step: il file si apre nel browser e si ricarica.
- **Tail ownership:** la verifica con la radio ANAN e la simulazione WAN spettano all'operatore (Simo); la ricognizione contro TOTW stock e i test in browser spettano all'implementatore.

---

## Product Contract

### Summary

Il piano porta TOTW da client di Thetis a client di deskHPSDR. Tre unita' obbligatorie rendono il client capace di parlare con deskHPSDR senza rumore ne' frame persi: un dispatcher rigoroso sul campo `type`, il sample rate IQ configurabile, e il consumo dello stream a bin. Due unita' consigliate sfruttano lo stream a bin per davvero, agganciandolo allo zoom e rendendo la sorgente selezionabile. Due unita' consigliate aggiungono il controllo da console MIDI. Una unita' di configurazione mette il tutto dietro https.

### Problem Frame

TOTW e' scritto per Thetis e riconosce i frame binari per euristica, non per contratto. Il risultato con deskHPSDR e' che l'audio si sente sporco e l'IQ non arriva affatto, per due cause distinte verificate nel sorgente:

- **L'audio.** Il dispatcher a `totw.html:4871` guarda il tipo a offset 24 solo per riconoscere `3` (TX_CHRONO) e `0` (IQ). Ogni altro tipo cade nel ramo audio, che assume un header non standard da 8 byte perche' Thetis usa quello. deskHPSDR manda l'header TCI regolare da 64 byte con `type=1`, quindi i 56 byte di header residuo vengono riprodotti come 14 float, cioe' 7 frame stereo di rumore per buffer. A 512 campioni su 48 kHz sono 93,75 clic al secondo.
- **L'IQ.** La condizione a `totw.html:4878` e' `srAt4 > 48000`, stretta. deskHPSDR a 48 kHz non la soddisfa, quindi anche i frame IQ finiscono nel ramo audio e vengono suonati come rumore.

Sopra a questo, lo stream IQ float32 costa almeno 3 Mbit/s ed e' inutilizzabile su 4G. Il server deskHPSDR espone gia' lo stream a bin che risolve il problema di banda, ma nessun client lo consuma: questo fork e' il primo.

### Requirements

| ID | Requisito | Priorita' |
|---|---|---|
| CLI-01 | Dispatch dei frame binari solo su `type` a offset 24. Audio con header da 64 byte e `type=1`. IQ riconosciuto anche a `sample_rate == 48000`. | M |
| CLI-02 | `iq_samplerate` configurabile; default di connessione adeguato a deskHPSDR. | M |
| CLI-03 | Handler `type=4`: dequantizza, espande i K bin su `IQ.fftResult`, imposta `S.iqSR`, `S.iqCentre`, `IQ.fftReady`. Nessuna modifica al codice di disegno. | M |
| CLI-04 | Invio di `spectrum_span` al cambio zoom. | S |
| CLI-05 | Sorgente spettro selezionabile: bin (default su WAN) o IQ+FFT (LAN). | S |
| CLI-06 | Web MIDI: accesso, dispatcher, learn mode, persistenza, encoder relativi e assoluti, coalescenza, feedback LED. | S |
| CLI-07 | Tabella azioni MIDI sui comandi TCI stock piu' `rx_att_ex` e `band_ex`. CW escluso. | S |
| CLI-09 | Servito via https dietro WireGuard, per microfono e Web MIDI in secure context. | M |

### Key Decisions

- **CLI-08 e' fuori.** DEC-01 del documento dei requisiti sceglie l'opzione A, Mumble su PipeWire, per l'audio su WAN. L'audio Opus dentro TCI non si fa in questa fase. Riaprirlo richiede di riaprire DEC-01.
- **Il CW non passa da qui.** La tabella azioni MIDI esclude il CW per scelta: il tool CW remoto e' un progetto separato con un percorso MIDI proprio, e mescolarli creerebbe un conflitto con il lock `trx` dei client TCI.
- **Il formato del frame e' congelato.** Lo definisce la specifica in `contract`, gia' implementata e testata lato server. Questo piano lo consuma e non lo discute.

### Scope Boundaries

Dentro: il consumo delle estensioni TCI di deskHPSDR, il controllo MIDI, il deployment https.

Fuori: qualsiasi modifica al server deskHPSDR; l'audio Opus in TCI; il tool CW; l'integrazione N1MM; il supporto Safari e iOS, che non implementano Web MIDI.

#### Deferred to Follow-Up Work

- Delta frame sullo spettro, cioe' inviare solo i bin cambiati. Il requisito CLI-04 lo cita come opzionale. Ha senso solo dopo aver misurato la banda reale con deflate attivo: se 512 bin a 10 fps stanno gia' sotto i 15 kbit/s, il delta non si ripaga.
- Un secondo receiver visualizzato in contemporanea.
- Lo scheduler audio. La issue #9 di origin descrive un ritardo che peggiora progressivamente e si azzera solo spegnendo e riaccendendo l'audio RX: e' lo scheduler naive gia' noto. Il fork lo eredita e questo piano non lo tocca, perche' riscriverlo e' CLI-08, fuori scopo con DEC-01 = A.

### Acceptance Examples

1. **Audio pulito.** Connesso a deskHPSDR, con l'audio TCI attivo, dieci minuti di ascolto senza il clic periodico. Il contatore dei frame audio sale, quello dei frame di tipo sconosciuto resta a zero.
2. **IQ a 48 k.** Con la sorgente spettro su IQ, il pannello disegna la traccia e `S.iqSR` vale 48000. Prima di questo piano il pannello resta piatto.
3. **Spettro a bin.** Con la sorgente su bin e `spectrum_start:0,512,10;`, la traccia e' sovrapponibile a quella IQ sulla stessa porzione di banda, con lo stesso floor entro 1 dB.
4. **Zoom che chiede.** Portando lo zoom da 1 a 8 il client invia `spectrum_span` con lo span ristretto, e la risoluzione visibile migliora davvero invece di ingrandire gli stessi bin.
5. **MIDI.** Ruotando l'encoder mappato sul VFO la frequenza segue senza scatti; il learn mode associa un controllo in meno di cinque secondi; la mappatura sopravvive al ricaricamento della pagina.

---

## Planning Contract

### Key Technical Decisions

- **CTD1 — Dispatch a tabella, non a euristica.** Il riconoscimento del frame diventa: se la lunghezza e' almeno 64 byte, leggi `type` a offset 24 e instrada su una tabella `type -> handler`. Nessun ramo di caduta che suona quello che non ha riconosciuto. Un tipo ignoto viene contato e scartato, con un log una tantum. Questo e' il cuore di CLI-01 e la ragione per cui le altre unita' possono assumere frame ben formati.

  L'alternativa scartata e' tenere il ramo euristico per compatibilita' con Thetis. Non serve: Thetis manda l'header da 8 byte e quello resta riconoscibile per esclusione sulla lunghezza, che e' un test esplicito e non un fallback silenzioso.

- **CTD2 — La modalita' a bin scrive in `IQ.fftResult` e nient'altro.** Il codice di disegno legge `IQ.fftResult[bin]` e mappa i pixel via `pxToBin`, che usa `S.iqCentre` e `S.iqSR`. Scrivendo i bin dequantizzati nell'array e impostando quei due valori, il disegno funziona senza toccarlo. E' il requisito CLI-03 e vincola tutte le scelte a valle.

- **CTD3 — Nessuno smoothing in modalita' a bin.** `IQ.smooth` vale 0,95 perche' la FFT locale e' rumorosa. I bin che arrivano dal server sono gia' il massimo su un gruppo di pixel, calcolati su una traccia che WDSP ha gia' mediato. Applicarci sopra un secondo filtro esponenziale ritarderebbe i segnali brevi senza ridurre nulla. Va anche tenuto allineato `IQ._prev`, altrimenti il ritorno alla modalita' IQ parte da uno stato vecchio.

- **CTD4 — Espansione a bin piu' vicino, non interpolazione lineare.** I K bin vanno portati sui 4096 di `IQ.fftResult`. Il bin sorgente e' gia' un massimo su un gruppo: interpolando linearmente si abbassano i picchi, cioe' si perde esattamente l'informazione che la decimazione aveva conservato. Il costo e' una traccia a gradini quando K e' molto minore della larghezza del canvas. E' un giudizio visivo da confermare al banco: se i gradini danno fastidio, la risposta giusta e' chiedere piu' bin, non interpolare.

- **CTD5 — La sorgente spettro e' una modalita' esplicita.** Niente autodetect. L'operatore sa se e' in LAN o su 4G meglio di qualsiasi euristica, e una modalita' che cambia da sola mentre si opera e' peggio di una scelta sbagliata ma stabile.

- **CTD6 — La mappatura MIDI vive in `localStorage`.** Chiave singola, oggetto JSON, versionata. Il learn mode scrive, il dispatcher legge. Nessun server coinvolto.

- **CTD7 — Il file resta uno.** TOTW e' un singolo `totw.html` da 454 kB, vanilla JS, senza build step, e questa e' la sua proprieta' piu' utile: si copia su una chiavetta e funziona. Ogni unita' aggiunge codice li' dentro. Se servisse WASM va incorporato in base64.

### High-Level Technical Design

Il client ha tre percorsi che questo piano tocca, e restano separati.

**Percorso dei frame binari.** Oggi `rxAudio()` a riga 4865 fa da dispatcher e da handler audio insieme. Le due responsabilita' si separano: un `dispatchBinaryFrame()` che legge il tipo e instrada, e handler distinti per audio, IQ, TX_CHRONO e spettro. L'handler audio impara a leggere l'header da 64 byte, distinguendolo da quello Thetis da 8 sulla lunghezza del frame contro il conteggio dei campioni.

**Percorso dello spettro.** In modalita' IQ nulla cambia: i frame IQ alimentano `runIQFFT()` che scrive `IQ.fftResult` con smoothing. In modalita' a bin il nuovo handler scrive lo stesso array direttamente, imposta lo span e alza `IQ.fftReady`. Il disegno non sa quale dei due percorsi lo ha alimentato, ed e' il punto: una sola traccia, due sorgenti.

**Percorso MIDI.** Completamente laterale. `requestMIDIAccess()` produce gli ingressi, un dispatcher normalizza i messaggi in eventi astratti, una tabella li mappa su azioni, le azioni chiamano le stesse funzioni che chiamano i controlli su schermo. Non tocca ne' il protocollo ne' il disegno.

### Sequencing

C1 e' il prerequisito di tutto il resto sui frame: senza dispatch pulito ogni altra unita' lavora su dati che potrebbero essere stati dirottati. C2 e' indipendente e piccolo. C3 dipende da C1. C4 e C5 dipendono da C3. C6 e C7 sono una catena a se' e possono procedere in parallelo. C8 e' configurazione e non dipende da niente.

Ordine consigliato: C1, C2, C3, poi in parallelo C4 piu' C5 da un lato e C6 piu' C7 dall'altro, C8 quando serve provare il microfono o il MIDI.

### Deferred Implementation Notes

La ricognizione contro TOTW stock descritta nel piano di verifica dei requisiti va fatta **prima** di C1, non dopo: serve a confermare che il clic a 94 Hz e l'IQ assente si manifestano davvero come previsto. Se non si manifestano, l'analisi del Problem Frame e' sbagliata da qualche parte e il piano va rivisto prima di scrivere codice.

---

## Implementation Units

### C1. Dispatch rigoroso dei frame binari

- **Goal:** sostituire il riconoscimento euristico con una tabella sul campo `type`, e insegnare all'handler audio l'header TCI da 64 byte.
- **Requirements:** CLI-01.
- **Dependencies:** nessuna.
- **Files:** `totw.html`, funzione `rxAudio()` e dintorni (righe 4865-4925 circa).
- **Approach:**
  1. Estrarre da `rxAudio()` un `dispatchBinaryFrame(buf)`: se `byteLength < 64` e' un frame Thetis legacy e va all'handler audio a 8 byte; altrimenti leggi `type` a offset 24 e instrada.
  2. Tabella dei tipi: `0` IQ, `1` audio, `3` TX_CHRONO, `4` spettro (aggancio per C3). Ogni altro valore incrementa un contatore e produce un log una tantum. Nessuna riproduzione.
  3. L'handler IQ perde la condizione `srAt4 > 48000`. Il sample rate resta utile come dato ma non come discriminante: il tipo lo ha gia' deciso.
  4. L'handler audio riceve l'offset del payload dal chiamante, 64 o 8, invece di assumere `MIN_HEADER = 8`.
  5. Il conteggio diagnostico esistente in `audioDiag` guadagna un campo per i frame scartati per tipo ignoto.
- **Patterns to follow:** lo stile del file, funzioni globali senza moduli, `DataView` con little-endian esplicito, log via `log('sys', ...)`.
- **Test scenarios:**
  - Frame audio deskHPSDR da 64 byte con `type=1`: riprodotto dall'offset 64, nessun clic, il conteggio dei frame audio sale.
  - Frame audio Thetis da 8 byte: riprodotto dall'offset 8, comportamento invariato rispetto a oggi.
  - Frame IQ deskHPSDR con `sample_rate == 48000` e `type=0`: riconosciuto come IQ, non suonato.
  - Frame IQ Thetis a 192000: riconosciuto come prima.
  - Frame con `type=4` prima che C3 esista: scartato e contato, nessun rumore.
  - Frame con tipo arbitrario, per esempio 7: scartato, contato, un solo log.
  - Frame di 40 byte: trattato come legacy Thetis, non come TCI troncato.
- **Verification:** dieci minuti di audio deskHPSDR senza clic udibile e senza frame ignoti; il pannello IQ disegna a 48 kHz.
- **Questa unita' ripara anche il caso Thetis, non solo deskHPSDR.** Il commento a `totw.html:4883` dichiara che l'header da 8 byte di Thetis e' stato determinato per via sperimentale. Non lo e': `buildStreamPayload()` in `Project Files/Source/Console/TCIServer.cs` di `ramdor/Thetis` alloca 64 byte piu' payload, scrive receiver, sample rate, tipo di campione, due zeri, lunghezza, tipo di stream e canali come `uint32`, poi otto parole di riserva, e copia i campioni a offset 64. E' lo stesso layout dell'header TCI standard, campo per campo. I presunti otto byte sono i primi otto di quello standard, letti con un layout inventato che combacia solo perche' il receiver 0 riempie di zeri i posti giusti.

  Ne segue che `n9bc/thetis-on-the-web` #12, il ronzio nelle portanti CW aperto come problema di vecchia data, e' quasi certamente questo difetto: 56 byte di header suonati in testa a ogni buffer. Sono interi piccoli e zeri, quindi come `float32` sono denormali, cioe' silenzio: l'artefatto e' un buco periodico, e la dissolvenza di 64 campioni che questa stessa funzione applica ai bordi lo trasforma in una modulazione di ampiezza a 93,75 Hz. Su una portante CW stabile si sente come ronzio; sul parlato e' mascherato.

  **Portare questa diagnosi al manutentore di origin prima di divergere.** Non e' un sospetto: ha i riferimenti al suo sorgente e a quello di Thetis.

### C2. Sample rate IQ e default di connessione

- **Goal:** togliere il 192000 cablato e dare un default di connessione sensato per deskHPSDR.
- **Requirements:** CLI-02.
- **Dependencies:** nessuna.
- **Files:** `totw.html`, righe 4154 e 5992 per `iq_samplerate`, riga 3309 per il campo host, riga 8163 per il messaggio di aiuto.
- **Approach:**
  1. Un campo nella UI di connessione per il sample rate IQ, con i valori che la radio puo' offrire, default 48000.
  2. Le due `send('iq_samplerate:192000;')` leggono quel valore.
  3. Il default del campo host passa a `ws://127.0.0.1:40001`, che e' la porta TCI di deskHPSDR. La porta non era cablata nel codice: era solo il valore iniziale del campo, e resta modificabile dall'utente come oggi.
  4. Il messaggio di aiuto alla riga 8163 cita la porta nuova.
- **Test scenarios:**
  - Connessione a deskHPSDR con i default: la porta e' giusta e la radio risponde.
  - Sample rate IQ impostato a 96000 e poi a 48000: il comando inviato segue il campo.
  - Connessione a un Thetis su 50001 cambiando la porta a mano: funziona come prima.
- **Verification:** connessione riuscita a deskHPSDR senza toccare nessun campo.

### C3. Handler dello stream a bin

- **Goal:** consumare il frame `type=4` e alimentare con esso la traccia esistente.
- **Requirements:** CLI-03.
- **Dependencies:** C1.
- **Files:** `totw.html`, nuovo handler accanto a `handleIQFrame`, piu' l'oggetto `IQ` a riga 5865.
- **Approach:**
  1. Parsare l'header da 64 byte per `receiver` e `length`, poi il prefisso da 32 byte a offset 64 secondo la specifica in `contract`: versione, flag, sequenza, `low_hz`, `high_hz`, `floor_db`, `scale_db`.
  2. Verificare la versione del prefisso. Una versione ignota va scartata con un log, non interpretata.
  3. Dequantizzare: `db = floor_db + bin * scale_db`.
  4. Espandere i K bin sui 4096 di `IQ.fftResult` a bin piu' vicino, secondo CTD4. Allineare `IQ._prev` allo stesso contenuto, cosi' il ritorno alla modalita' IQ non parte da uno stato vecchio.
  5. Impostare `S.iqSR = high_hz - low_hz`, `S.iqCentre = (low_hz + high_hz) / 2`, `IQ.fftReady = true`.
  6. Monitorare la sequenza: un salto significa frame persi e va contato, perche' e' il segnale che il link e' saturo.
- **Patterns to follow:** la lettura del prefisso rispecchia la tabella della specifica campo per campo, con `getInt32`/`getFloat32` e i due `low_hz`/`high_hz` a 64 bit letti come `BigInt64` e convertiti.
- **Test scenarios:**
  - Frame con 512 bin: `IQ.fftResult` interamente popolato, nessun `NaN`, nessun indice fuori range.
  - Frame con 16 bin, il minimo: l'espansione non divide per zero.
  - Frame con 4096 bin: nessuna espansione, copia diretta.
  - Prefisso con versione 2: scartato, un log, la traccia precedente resta.
  - `floor_db` negativo e `scale_db` a 0,5: i valori dequantizzati stanno nel range che il disegno si aspetta.
  - Sequenza che salta da 10 a 14: il contatore dei persi sale di 3.
  - Frame con span disgiunto dal precedente, cioe' dopo un cambio banda: `S.iqCentre` segue senza artefatti.
- **Verification:** con la stessa radio e la stessa porzione di banda, la traccia a bin e quella IQ hanno lo stesso floor entro 1 dB e i picchi coincidono in frequenza.

### C4. spectrum_span agganciato allo zoom

- **Goal:** far chiedere al server lo span che serve davvero, invece di ingrandire i bin che si hanno.
- **Requirements:** CLI-04.
- **Dependencies:** C3.
- **Files:** `totw.html`, `specZoom` e `specZoomCentre` a riga 6097, piu' i tre punti che li modificano: pinch a 8014, Ctrl piu' rotella a 8096, doppio clic a 8145.
- **Approach:**
  1. Una funzione sola che, dato lo stato di zoom, calcola lo span visibile e invia `spectrum_span:<rx>,<low>,<high>;`. I tre punti che cambiano lo zoom la chiamano.
  2. Coalescenza: durante un pinch lo zoom cambia in continuo, e mandare un comando per evento inonda il socket. Un ritardo di 150-200 ms con vittoria dell'ultimo valore.
  3. Il flag di ritaglio nel prefisso dice se il server ha ristretto la richiesta. Se ristretta, lo zoom locale va riportato allo span davvero servito, altrimenti la scala in frequenza mente.
  4. In modalita' IQ questa funzione non fa nulla: lo zoom resta un ritaglio locale sull'array completo.
- **Test scenarios:**
  - Zoom da 1 a 8: un solo comando inviato, con lo span corretto.
  - Pinch continuo per due secondi: al piu' una decina di comandi, non uno per frame di animazione.
  - Doppio clic di reset: lo span torna a quello pieno.
  - Server che ritaglia perche' lo span richiesto sborda: il flag e' letto e la scala si corregge.
  - Modalita' IQ: nessun comando inviato.
- **Verification:** a zoom 8 su 512 bin la risoluzione visibile e' otto volte migliore che a zoom 1, non la stessa ingrandita.

### C5. Selettore della sorgente spettro

- **Goal:** rendere esplicita e persistente la scelta fra bin e IQ.
- **Requirements:** CLI-05.
- **Dependencies:** C3.
- **Files:** `totw.html`, UI accanto ai controlli dello spettro, piu' i punti che avviano e fermano i due stream.
- **Approach:**
  1. Un selettore a due valori. Il passaggio a bin invia `spectrum_start` e ferma l'IQ con `iq_stop`; il passaggio a IQ fa l'inverso con `spectrum_stop`.
  2. Numero di bin e fps sono campi accanto al selettore, con i default della specifica, 512 e 10.
  3. La scelta e il resto della configurazione vivono in `localStorage`.
  4. Alla riconnessione lo stream si riavvia da solo nella modalita' scelta.
  5. L'evento `spectrum_state` con valore 0 significa che il display remoto e' in pausa, tipicamente in TX. La UI lo mostra invece di far sembrare il client bloccato.
- **Test scenarios:**
  - Passaggio bin verso IQ e ritorno: nessuno stream resta acceso in sottofondo.
  - Ricaricamento della pagina: la modalita' e i parametri sono quelli di prima.
  - Riconnessione dopo caduta del link: lo stream riparte nella modalita' scelta.
  - `spectrum_state:0,0;` durante il TX: la UI segnala la pausa e riprende da sola.
  - `spectrum_fps` che scende a 5 sotto saturazione: la UI mostra il valore servito, non quello richiesto.
- **Verification:** su LAN in modalita' IQ e su 4G in modalita' bin, entrambe utilizzabili senza toccare altro.

### C6. Infrastruttura Web MIDI

- **Goal:** trasformare i messaggi MIDI in eventi astratti, con learn mode e persistenza.
- **Requirements:** CLI-06.
- **Dependencies:** nessuna. Richiede C8 per il secure context.
- **Files:** `totw.html`, sezione nuova.
- **Approach:**
  1. `navigator.requestMIDIAccess({ sysex: false })`, elenco degli ingressi, gestione di `statechange` per la console collegata a caldo.
  2. Normalizzazione: nota on/off, control change, pitch bend diventano un evento con sorgente, canale, controllo e valore.
  3. Encoder relativi in due codifiche, complemento a due e segno-modulo, piu' assoluti a 7 bit. La codifica e' una proprieta' della mappatura, non indovinata: si sceglie nel learn mode.
  4. Learn mode: si attiva su un'azione, il primo controllo mosso viene catturato, si chiede la codifica se il controllo sembra un encoder.
  5. Coalescenza a 30-50 ms con vittoria dell'ultimo valore, altrimenti un encoder veloce genera decine di comandi TCI al secondo.
  6. Feedback LED via MIDI out per le azioni a due stati.
- **Test scenarios:**
  - Console collegata dopo il caricamento: compare senza ricaricare.
  - Encoder in complemento a due: giri orari e antiorari danno delta di segno opposto e modulo giusto.
  - Encoder in segno-modulo: idem con l'altra codifica.
  - Rotazione veloce per un secondo: i comandi inviati sono limitati dalla coalescenza, l'ultimo valore e' quello giusto.
  - Learn mode su un controllo gia' mappato: sostituisce, non duplica.
  - Ricaricamento: la mappatura e' quella di prima.
  - Browser senza Web MIDI, cioe' Safari: la sezione si disabilita con un messaggio, il resto del client funziona.
- **Verification:** una console DJ mappata da zero in meno di cinque minuti, funzionante dopo il ricaricamento.

### C7. Tabella azioni MIDI

- **Goal:** collegare gli eventi astratti ai comandi TCI, inclusi i due nuovi del server.
- **Requirements:** CLI-07.
- **Dependencies:** C6.
- **Files:** `totw.html`, accanto a C6.
- **Approach:**
  1. Le azioni chiamano le stesse funzioni dei controlli su schermo, mai `send()` diretto: cosi' la UI resta sincronizzata e la logica di lock non si duplica.
  2. Insieme delle azioni: VFO a jog, volume, drive, squelch, RIT, larghezza filtro, modo, banda, attenuazione, PTT, tune, mute, split, NR e NB.
  3. Banda e attenuazione usano `band_ex` e `rx_att_ex`, i due comandi che il server ha aggiunto. Sono gli unici che escono dal TCI stock.
  4. Il CW e' escluso per la decisione presa nel Product Contract.
  5. Le azioni continue accettano delta relativi e valori assoluti; le azioni a due stati accettano nota on/off.
- **Test scenarios:**
  - Jog sul VFO: la frequenza segue e il campo su schermo si aggiorna.
  - `band_ex` su una banda gia' attiva senza il flag: nessun avanzamento nel band stack.
  - `rx_att_ex` su radio a due ADC: agisce sull'ADC del ricevitore giusto.
  - PTT da nota on/off: il TX parte e si ferma; rilascio mancante per messaggio perso, da verificare il fail-safe.
  - Azione su un parametro bloccato da un altro client: l'errore e' mostrato, non ignorato.
- **Verification:** un contatto completo condotto dalla sola console, senza toccare mouse ne' tastiera.

### C8. Servizio https dietro WireGuard

- **Goal:** servire il client in secure context, che microfono e Web MIDI richiedono.
- **Requirements:** CLI-09.
- **Dependencies:** nessuna.
- **Files:** configurazione fuori dal repo, piu' una nota nel README.
- **Approach:**
  1. Il file e' statico: basta un server qualsiasi con TLS sull'indirizzo WireGuard dell'host radio.
  2. Il certificato puo' essere autofirmato, ma allora va installato come attendibile sul dispositivo, altrimenti il secure context non scatta.
  3. Il WebSocket TCI passa a `wss://` solo se c'e' un terminatore TLS davanti: deskHPSDR parla `ws://` in chiaro. In alternativa il tunnel WireGuard fa da cifratura e il WebSocket resta in chiaro dentro di esso, che e' la scelta piu' semplice.
  4. Il bind address del server TCI va messo sull'indirizzo WireGuard, non su tutte le interfacce.
- **Test scenarios:**
  - Pagina su https: il microfono chiede il permesso e Web MIDI e' disponibile.
  - Pagina su http da un indirizzo non locale: entrambi negati, il messaggio lo spiega.
- **Verification:** microfono e MIDI funzionanti da un dispositivo remoto dentro il tunnel.
- **Conferma dall'utenza di origin:** le issue #8 e #11 di `n9bc/thetis-on-the-web` sono entrambe questo problema, il PTT e il microfono che non funzionano perche' la pagina e' servita in chiaro. Non e' un requisito teorico nostro.

---

## Verification Contract

**Prima di C1**, ricognizione contro TOTW stock connesso a deskHPSDR: registrare la sequenza `audio_samplerate` e `audio_start`, i tipi dei frame ricevuti, e confermare a orecchio e allo spettrogramma il clic a 93,75 Hz e il pannello IQ piatto. Se non si osservano, fermarsi.

**Dopo ogni unita'**, il client si apre nel browser e si connette senza errori in console.

**Al banco con la radio**, con l'operatore:

1. Traccia a bin contro traccia IQ sulla stessa porzione: stesso floor entro 1 dB.
2. Banda misurata su 512 bin a 10 fps con deflate attivo: sotto i 15 kbit/s.
3. WAN simulata con `netem` a 60 ms di ritardo, 2 % di perdita e 32 kbit/s: la scala fps scende a 5 e risale, il client mostra il valore servito.
4. Console MIDI: tutte le azioni della tabella di C7.
5. Regressione: TOTW stock contro Thetis continua a funzionare, perche' il fork non deve rompere il caso originale.

## Definition of Done

- Le tre unita' obbligatorie sono implementate e verificate al banco.
- Il client si connette a deskHPSDR con i default, senza clic e con lo spettro a bin funzionante.
- Nessuna modifica al codice di disegno esistente.
- Il file resta uno, senza build step.
- Il README documenta la connessione a deskHPSDR e la differenza fra le due sorgenti spettro.
- Le unita' consigliate sono implementate o esplicitamente rinviate con una ragione.

## Sources / Research

- `n9bc/thetis-on-the-web` a `main`, `totw.html`, letto il 2026-09-09: dispatcher a 4865-4881, handler audio a 4884-4915, oggetto `IQ` a 5865, scrittura di `fftResult` a 5983, mappatura del disegno a 6155-6165, stato dello zoom a 6097, campo host a 3309.
- `iu3qez/deskhpsdr`, `documentation/deskHPSDR_TCI_Remote_Extensions.md`: contratto del frame `type=4`, comandi `spectrum_*`, `rx_att_ex`, `band_ex`.
- `iu3qez/deskhpsdr`, `docs/brainstorms/2026-09-08-remote-station-requirements.md`: requisiti CLI-01 e seguenti, decisione DEC-01.
- Issue aperte di `n9bc/thetis-on-the-web` consultate il 2026-09-10: #8 e #11 (secure context per PTT e microfono), #9 (ritardo audio progressivo), #12 (ronzio nelle portanti CW). Le altre cinque non toccano nessuna unita' di questo piano.
