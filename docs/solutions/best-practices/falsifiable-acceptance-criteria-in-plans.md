---
title: Criteri di accettazione che non possono falsificare
date: 2026-09-12
category: best-practices
module: deskhpsdr-client
problem_type: best_practice
component: development_workflow
severity: high
applies_when:
  - Un criterio di accettazione legge un flag booleano, un campo di stato o un elemento disegnato
  - Uno scenario di test chiede che il comportamento resti invariato in un'unita' che quel comportamento deve cambiare
  - Un criterio nomina un sintomo udibile o visibile senza aver accertato quale meccanismo lo produce
  - Due difetti toccano la stessa grandezza con segni opposti e si mascherano a vicenda
tags: [criteri-accettazione, falsificabilita, piani, verifica, totw, tci]
---

# Criteri di accettazione che non possono falsificare

## Context

Durante l'implementazione di C1 (issue #2, integrata con la PR #12) tre criteri di accettazione del piano `docs/plans/2026-09-09-feat-deskhpsdr-client-plan.md` si sono rivelati incapaci di falsificare quello che dichiaravano di verificare. Tutti e tre sono stati scoperti leggendo il sorgente o misurando, non osservando il comportamento del client. Il piano e' stato emendato in due passaggi sul branch d'integrazione `plan/deskhpsdr-client`: la condizione di stop rivista alla riga 303 viene dal commit `9c064c6`, mentre le righe 71 e 149 vengono da `51f9ade`. In questo fork `main` resta allineato a monte e non e' il ramo d'integrazione, quindi i commit citati qui si risolvono contro `plan/deskhpsdr-client`.

Il contesto che rende il comportamento un oracolo inaffidabile in questa unita' e' documentato alla riga 161 del piano: **due difetti si mascheravano a vicenda**. Il client cablava `iq_samplerate:192000` alla connessione, cosa ancora vera nel tree a `totw.html:4154` e a `totw.html:6008`, dentro `startIQ()`. Quel comando alza il sample rate della radio a 192 kHz, e 192 kHz soddisfa esattamente il gate rotto `srAt4 > 48000` che C1 doveva rimuovere. Il sintomo osservabile, "l'IQ funziona", era vero mentre il meccanismo era sbagliato: correggendo uno solo dei due difetti il pannello sparisce e sembra una regressione introdotta da noi.

In questo regime qualunque criterio formulato come "guarda se il pannello disegna" nasce difettoso, perche' la variabile osservata non e' collegata all'ipotesi sotto esame.

Una sessione precedente, del 10 e 11 settembre 2026 sullo stesso branch, aveva gia' stressato i primi due criteri leggendo il sorgente del server e costruendo simulazioni offline dell'artefatto audio, e si era fermata in attesa della radio senza arrivare a emendare il piano (session history). Il terzo caso viene da quelle simulazioni, riprodotte e confermate qui.

## Guidance

### 1. Un flag di validita' deve essere invalidato dalla stessa condizione che lo produce

Prima di scrivere un criterio che legge un flag booleano, elenca i suoi **siti di scrittura**. Se il flag ha un produttore e un invalidatore che rispondono a cause diverse, il flag non descrive lo stato corrente del sistema e il criterio che lo legge non prova nulla.

In `totw.html` il flag ha esattamente due scritture dopo l'inizializzazione a `totw.html:5886`:

```js
  IQ.fftReady = true;    // totw.html:6002, unica riga che lo pone vero, dentro runIQFFT()
  IQ.fftReady = false;   // totw.html:6016, unica riga che lo pone falso, dentro stopIQ()
```

Il produttore e' l'arrivo dei dati, l'invalidatore e' l'azione dell'utente. Sono cause indipendenti, quindi quando il flusso si interrompe per qualsiasi altra ragione il flag resta vero e il codice di disegno continua a leggere l'ultimo contenuto di `IQ.fftResult`, congelato (`totw.html:6269`, `totw.html:6285`, `totw.html:6424`). Lo stesso flag governa anche il marker di tuning, il meter IQ e i gesti sul pannello (`totw.html:5277`, `totw.html:6087`, `totw.html:6138`, `totw.html:7907`): un flag non invalidato propaga l'illusione a tutto cio' che lo legge.

### 2. Il discriminante corretto e' la liveness, non la readiness

Al posto del flag, usa un contatore monotono incrementato dall'evento che ti interessa:

```js
  IQ.frameCount++;   // totw.html:5976, unica riga che lo incrementa, in fondo a handleIQFrame()
```

`IQ.frameCount` e' dichiarato a `totw.html:5888` e non viene mai azzerato (il reset diagnostico a `totw.html:4800` tocca solo `audioDiag`). Quindi il criterio non e' "il contatore e' diverso da zero" ma **"il contatore avanza fra due letture"**. E' esposto nella diagnostica come `iqFrames` a `totw.html:7689`, accanto a `fftReady`: il dato giusto e quello fuorviante stanno nello stesso blocco.

### 3. Prima di scrivere il criterio, risali la catena a produttore singolo

Il ragionamento sopra e' solido solo perche' la catena ha un produttore per anello. Verificalo nel tree, non a memoria:

- l'unico chiamante di `handleIQFrame()` (definita a `totw.html:5949`) e' il dispatcher, a `totw.html:4897`;
- l'unico chiamante di `runIQFFT()` (definita a `totw.html:5979`) e' `handleIQFrame`, a `totw.html:5973`;
- `runIQFFT` e' l'unico scrittore di `IQ.fftResult` dopo l'allocazione in `initIQ()` (`totw.html:5897`), alla riga `totw.html:5999`.

Una catena a produttore singolo ti permette di concludere con certezza quale osservabile cambia se l'ipotesi e' falsa. La domanda da porsi e' sempre quella: **quale grandezza osservabile assume un valore diverso se la mia analisi e' sbagliata?** Se la risposta e' "nessuna", il criterio e' decorativo e va riscritto.

### 4. Uno scenario di test non puo' contraddire un'altra parte dello stesso documento

Quando aggiungi uno scenario a un piano, rileggi le decisioni tecniche gia' registrate nello stesso documento e verifica la coerenza. Uno scenario che chiede di preservare "il comportamento di oggi" e' sospetto per costruzione in un'unita' il cui scopo e' cambiare quel comportamento.

Il modo piu' economico per scoprirlo e' **scrivere lo scenario come test eseguibile e guardarlo fallire**. Se fallisce per la ragione sbagliata, cioe' non perche' il codice e' sbagliato ma perche' lo scenario e' insoddisfacibile, hai trovato un difetto nel piano invece che nel codice.

### 5. Accerta che il sintomo nominato dal criterio sia prodotto dal meccanismo che l'unita' cambia

E' la regola piu' costosa da ignorare, perche' produce il falso negativo: un criterio che **boccia un'implementazione corretta**. Un sintomo udibile o visibile puo' avere una causa diversa dal difetto che stai correggendo, e un criterio che li identifica misura la causa sbagliata.

Il controllo e' sempre lo stesso e costa poco: isola il meccanismo candidato e misuralo da solo, con il difetto presente e con il difetto assente. Se il sintomo non cambia, il criterio non parla dell'unita'.

### 6. Quando due difetti si mascherano, ordina le unita' e non fidarti dell'osservazione

Se due difetti si cancellano a vicenda, la correzione isolata di uno dei due produce un peggioramento visibile. Le conseguenze operative: l'ordine delle unita' diventa vincolante e va scritto nel piano, e il criterio di verifica dell'unita' va ancorato a una grandezza interna che non dipende dall'altro difetto.

## Why This Matters

Un criterio che passa in entrambi i mondi non e' debole: e' **nullo**, e costa piu' di nessun criterio. Nel caso A la condizione di stop serviva a decidere se fermarsi e riesaminare l'analisi. Letta alla lettera avrebbe imposto di fermarsi anche con l'analisi corretta, oppure, con lo stesso diritto, di procedere con l'analisi sbagliata: la scelta fra le due sarebbe stata presa dal giudizio dell'operatore, non dal criterio. Il piano conserva la lezione alla riga 71: il discriminante e' `IQ.frameCount` che avanza, non `IQ.fftReady`.

Il costo cresce con il valore dell'osservazione. Qui le osservazioni residue si fanno al banco con la radio accesa (sezione *Verification Contract* del piano, riga 311 e seguenti). Una sessione al banco che valida un criterio nullo e' tempo e hardware spesi per non apprendere niente, e produce la peggiore delle uscite: fiducia in una conclusione non verificata.

Nel caso B il costo e' diverso e piu' insidioso. Uno scenario che chiede "comportamento invariato rispetto a oggi" su un difetto che l'unita' corregge, se preso per buono, si trasforma in un vincolo di regressione: blocca la correzione in nome della compatibilita' con un bug. Ed essendo insoddisfacibile sopra i 64 byte, un tentativo di soddisfarlo avrebbe prodotto un ramo di codice morto, esattamente il ramo euristico che CTD1 ha deciso di togliere.

Il caso C costa di piu' di entrambi, perche' il criterio non e' nullo ma **falso**: avrebbe bocciato C1, che e' corretta. Un'unita' bocciata da un criterio sbagliato e' un'unita' che si rimette in discussione, e lo storico del repo mostra che su questo progetto i revert avvengono (`0137bf7`, `ab8614f`). Ha anche prodotto un danno gia' materializzato: il corpo della PR #12 afferma che l'artefatto periodico non c'e' piu', e quell'affermazione e' falsa.

Infine il mascheramento reciproco: ignorarlo significa che la prima correzione corretta viene letta come regressione.

## When to Apply

Applica le regole 1, 2 e 3 **ogni volta che un criterio di accettazione legge un flag booleano, un campo di stato o un elemento di interfaccia disegnato**, invece di leggere un contatore di eventi o un valore ricavato dai dati. Il caso sospetto per eccellenza e' "il pannello X si aggiorna / si svuota / mostra Y".

Applica la regola 4 quando aggiungi o modifichi scenari in un documento che contiene gia' decisioni tecniche registrate, e in particolare quando uno scenario contiene la formula "come oggi" o "comportamento invariato".

Applica la regola 5 ogni volta che un criterio nomina un sintomo percepito, un clic, un ronzio, uno sfarfallio, un ritardo, invece di una grandezza interna. Il sintomo va attribuito a un meccanismo prima di entrare in un criterio.

Applica la regola 6 quando due difetti toccano la stessa grandezza con segni opposti, in qualunque forma: un default cablato che soddisfa un gate rotto, un retry che nasconde un timeout, una cache che nasconde una query sbagliata.

Per le unita' rimanenti del piano, in concreto:

- **C2** toglie il `192000` cablato. I due siti di invio nel tree attuale sono `totw.html:4154` e `totw.html:6008`; la riga 168 del piano li indica come "righe 4154 e 5992", e il secondo riferimento e' ormai stale perche' il file e' cresciuto. Poiche' C1 e' gia' integrata il mascheramento non c'e' piu', ma il criterio di C2 va ancorato al comando effettivamente inviato e a `S.iqSR` (letto dall'header a `totw.html:5956`, esposto come `iqSampleRate` a `totw.html:7688`), non al fatto che il pannello disegni. Attenzione anche al proprietario dello stream lato server: un secondo client viene coerciato al rate esistente senza errore, quindi il valore da leggere e' la risposta del server.
- **C3** aggiunge l'handler dello spettro a bin. Oggi il ramo `STREAM_SPECTRUM` conta e scarta (`totw.html:4900`), e il contatore `audioDiag.spectrumFrames` (`totw.html:4757`) e' gia' il discriminante di liveness giusto. Se C3 introduce un proprio flag di readiness, va invalidato dall'assenza di frame e non solo dallo stop esplicito.
- **C5** tocca avvio e arresto dei due stream. E' il punto in cui decidere l'invalidazione di `IQ.fftReady`, che la riga 71 del piano lascia aperta fra C5 e un'unita' propria.
- **C4, C6, C7** ereditano la regola generale: ogni criterio che nomina il pannello, il marker o il meter sta leggendo `IQ.fftReady` per interposta persona.

## Examples

### Caso A: la condizione di stop che passa in entrambi i mondi

Ipotesi sotto esame: il dispatcher binario scartava i frame IQ quando la radio girava a 48 kHz, perche' li filtrava dietro `sample_rate > 48000`.

**Prima** (condizione originale, riga 301 del piano): fermarsi se non si osservano il clic a 93,75 Hz e il pannello IQ piatto. Il pannello non era piatto, il client contava decine di migliaia di frame e disegnava.

**Dopo l'emendamento** (riga 303): fermarsi se, con la radio riportata a 48 kHz, il pannello IQ disegna lo stesso.

Anche la versione emendata non falsifica niente, ed e' questo il punto. Con la radio a 48 kHz e l'ipotesi **vera**, i frame vengono scartati, `IQ.fftReady` resta vero dalla fase a 192 kHz, e il pannello disegna lo stesso il contenuto congelato di `IQ.fftResult`. Con l'ipotesi **falsa**, i frame arrivano e il pannello disegna dal vivo. "Il pannello disegna lo stesso" e' vero nei due casi.

**Forma sana del criterio**, costruita sulla catena verificata sopra:

```
Leggi iqFrames nella diagnostica (totw.html:7689), attendi qualche secondo, rileggilo.
  - il valore avanza   -> i frame IQ raggiungono handleIQFrame: l'ipotesi e' falsa
  - il valore e' fermo -> i frame non arrivano al handler: l'ipotesi regge
In nessun caso guardare il pannello: fftReady non si invalida da se'.
```

### Caso B: lo scenario che contraddiceva il proprio documento

**Prima**, nell'elenco dei test di C1:

> Frame audio Thetis da 8 byte: riprodotto dall'offset 8, comportamento invariato rispetto a oggi.

**Dopo** (riga 149 del piano, emendata): il frame legacy sotto i 64 byte, riprodotto dall'offset 8, con la nota che lo scenario precedente era insoddisfacibile.

Due cose non tornavano. La prima: la nota CTD1 dello stesso piano (righe 88 e 90, dettagliata alla riga 156) registra, come risultato verificato su `buildStreamPayload()` in `Project Files/Source/Console/TCIServer.cs` di `ramdor/Thetis`, che Thetis scrive l'header TCI standard da 64 byte campo per campo, non uno da 8. Questo repo non contiene un clone di `ramdor/Thetis`, quindi la verifica **non e' ripetibile qui**: il fatto e' riportato come conclusione registrata nel piano, non come controllo eseguito in questa sede. Cio' che **e'** verificabile in locale e' l'header da 64 byte del server deskHPSDR, in un checkout gemello fuori da questo repo, quindi stato machine-local: `/Users/sf/Developer/deskhpsdr/src/tci_audio.c:612`, che valida la lunghezza del frame audio come `64 + length * sizeof(float)`, e `/Users/sf/Developer/deskhpsdr/src/tci_spectrum.c:242`, che documenta lo stesso header da 64 byte davanti a un payload a bin `uint8`, dove quella formula non si applica.

La seconda: "comportamento invariato rispetto a oggi" chiedeva di preservare proprio il difetto che C1 corregge, cioe' leggere un frame da 64 byte a partire dall'offset 8 e suonare 56 byte di header come campioni.

Lo scenario e' inoltre insoddisfacibile per costruzione, perche' la nuova regola di dispatch tratta qualunque frame di almeno 64 byte come frame TCI standard:

```js
  if (buf.byteLength < TCI_HEADER_BYTES) {   // totw.html:4887
    rxAudioFrame(buf, LEGACY_HEADER_BYTES);
    return;
  }
```

L'unica forma rivendicabile e' quella sotto i 64 byte, che coincide con il caso da 40 byte gia' elencato separatamente. Il difetto e' emerso scrivendo lo scenario come test eseguibile e guardandolo fallire per la ragione sbagliata: non perche' il codice fosse sbagliato, ma perche' lo scenario lo era.

### Caso C: il criterio che avrebbe bocciato un'implementazione corretta

Criterio di verifica di C1, nel piano: dieci minuti di audio deskHPSDR senza clic udibile e senza frame di tipo ignoto, piu' il pannello IQ che disegna a 48 kHz.

La prima meta' non puo' passare, **ne' prima ne' dopo C1**, perche' l'artefatto udibile non e' prodotto da cio' che C1 cambia. `playFloat32Stereo` applica una dissolvenza di `Math.min(64, frames >> 2)` campioni a entrambi i bordi di **ogni** buffer (`totw.html:4969` e seguenti), e C1 non l'ha toccata. Con i frame audio da 512 campioni di deskHPSDR questo attenua un quarto di ogni buffer, 93,75 volte al secondo, che e' semplicemente 48000/512, la cadenza dei buffer.

Misura ottenuta replicando la funzione su una portante continua a 700 Hz (confermata indipendentemente da una simulazione della sessione precedente, session history):

| configurazione | riga di modulazione | quota di buffer attenuata |
|---|---|---|
| pre-C1, payload dall'offset 8, 519 campioni per buffer | 92,49 Hz | 24,7 % |
| post-C1, payload dall'offset 64, 512 campioni per buffer | 93,75 Hz | 25,0 % |
| dissolvenza rimossa | nessuna | 0 % |

I campioni nella tabella sono frame stereo: il frame audio del server porta 512 frame stereo, cioe' 1024 float, ed e' sempre esattamente quella dimensione perche' il server rifiuta i buffer parziali. La quota attenuata e' la frazione di buffer coperta dalle due rampe, `2 x FADE`; i campioni con fattore strettamente minore di uno sono 127 e non 128, perche' il primo campione della rampa di chiusura vale esattamente uno.

C1 sposta la riga di 1,26 Hz e toglie dal flusso i 56 byte di header, che sono difetti veri, ma **il ronzio sopravvive**. I 93,75 Hz che il piano attribuisce ai buchi dell'header modellati dalla dissolvenza sono la cadenza dei buffer: la dissolvenza li produce da sola, e la sua rimozione e' l'unica cosa che li fa sparire.

Due conseguenze. Il criterio avrebbe dichiarato fallita un'unita' corretta. E la issue #12 a monte su `n9bc/thetis-on-the-web`, il ronzio nelle portanti CW, e' con ogni probabilita' la dissolvenza e non l'header: l'attribuzione registrata nel piano va considerata non dimostrata.

**Forma sana del criterio**: misurare la profondita' di modulazione a 93,75 Hz sul flusso riprodotto, e verificare che C1 riduca a zero i frame di tipo ignoto e i campioni spuri in testa al buffer, senza promettere nulla sul ronzio finche' la dissolvenza resta dov'e'.

### Cosa ha cambiato C1, per riferimento

Il gate euristico rimosso dal commit `e41121a`, raggiungibile dal tree attraverso il merge della PR #12:

```js
  // prima
  if (typeAt24 === 0 && srAt4 > 48000 && srAt4 <= 1000000) { handleIQFrame(buf); return; }
```

sostituito dal dispatch sul tipo di stream a offset 24 in `dispatchBinaryFrame()` (`totw.html:4883`), con la tabella a `totw.html:4896`. L'handler audio riceve l'offset del payload dal chiamante invece di assumerlo (`rxAudioFrame(buf, offset)`, `totw.html:4920`), e il ramo di caduta che suonava i byte non riconosciuti e' stato rimosso anziche' lasciato inutilizzato: `tryRawAudio` non ha piu' nessuna occorrenza nel tree. Un tipo ignoto viene contato in `audioDiag.discardedFrames` e loggato una volta sola (`totw.html:4910`).

## Related

Nessun apprendimento precedente si sovrappone a questo: `docs/solutions/` non esisteva prima di questo documento, quindi la sovrapposizione con il corpus e' nulla su tutte e cinque le dimensioni.

- `docs/plans/2026-09-09-feat-deskhpsdr-client-plan.md` — il documento che conteneva i tre criteri. Righe 71, 149 e 303 portano le correzioni dei primi due; il criterio di verifica di C1 e la nota CTD1 sull'attribuzione del ronzio restano da correggere alla luce del caso C.
- `docs/handoffs/2026-09-12-c1-merged-c2-next.md` — stato dopo C1 e trappole verificate. La sua ricostruzione del meccanismo del ronzio e' superata dal caso C.
- `docs/handoffs/2026-09-12-totw-handoff.md` — handoff superato, conservato per tracciabilita'. Contiene l'attribuzione originale del ronzio all'header.
- `README.md` — da aggiornare quando C1 e C3 saranno verificate al banco, per la Definition of Done del piano.
- Issue `#1` (ricognizione) e `#2` (C1) di `iu3qez/thetis-on-the-web`: i criteri qui discussi provengono da queste due. `#3` (C2) porta il vincolo d'ordine del mascheramento reciproco.
- Issue `#12` di `n9bc/thetis-on-the-web`: ronzio nelle portanti CW, la cui attribuzione cambia per il caso C.
