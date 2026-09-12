# Handoff — fork TOTW per deskHPSDR

Data: 2026-09-12. Riguarda **solo** il client. Il lavoro sul server deskHPSDR ha un handoff proprio nel suo repository e non va mescolato con questo.

## Stato

Fork di `n9bc/thetis-on-the-web` creato il 2026-09-09. Remote `upstream` verso n9bc, `main` allineato a lui (`c147edf`), nessuna modifica.

Tutto il nostro lavoro sta sul branch **`plan/deskhpsdr-client`**, e per ora e' solo documentazione: **nessuna riga di codice scritta**.

- Piano: `docs/plans/2026-09-09-feat-deskhpsdr-client-plan.md`
- Issue aperte: #1 ricognizione, #2..#8 le unita' C1..C7.

Il piano e' la fonte autorevole per il meccanismo; il formato dei frame e' congelato dalla specifica del server e non si discute da qui.

## Cosa ha trovato la ricognizione del 2026-09-12

Eseguita in parte, con TOTW stock servito su `http://localhost:8000/totw.html` in Firefox 155, connesso a `ws://127.0.0.1:40001`.

### 1. TOTW riconfigura la radio collegandosi

Il `iq_samplerate:192000` cablato nel client non e' un default scomodo: e' un comando che **cambia il sample rate della radio dell'operatore**. Il gestore lato server schedula il cambio quando nessun client riceve ancora IQ e il rate richiesto differisce da quello del receiver attivo, che e' esattamente la situazione alla connessione. La radio passa da 48 a 192 kHz senza che nessuno lo chieda.

Su WAN sarebbe disastroso: IQ a 192 kHz float32 stereo sono circa 12 Mbit/s.

Conseguenza sul piano: **C2 (issue #3) e' piu' grande, non piu' piccola, di come la descrivevano i requisiti.** La porta invece resta un dettaglio: non era cablata, era solo il valore iniziale del campo di connessione.

### 2. I difetti di C1 e C2 si mascherano a vicenda

La condizione `sample_rate > 48000` di TOTW fa sparire l'IQ quando la radio e' a 48 kHz. Ma il difetto di C2 porta la radio a 192 kHz, quindi la condizione viene soddisfatta e l'IQ funziona: nella ricognizione il client conta decine di migliaia di frame e disegna.

**Vincolo d'ordine: C1 (#2) va prima di C2 (#3), o insieme.** Correggendo C2 da solo l'IQ sparisce e sembra una regressione introdotta da noi.

Per questo la condizione di stop della ricognizione e' stata emendata invece che applicata: il pannello IQ non e' piatto come previsto, ma la causa e' accertata sul sorgente. La condizione rivista e': fermarsi se, **con la radio riportata a 48 kHz**, il pannello IQ disegna lo stesso.

### 3. L'header da 8 byte non esiste

Il commento a `totw.html:4883` dichiara che l'header di Thetis e' non standard, da 8 byte, e che il layout e' stato ricavato per via sperimentale.

E' falso, verificato leggendo il sorgente di Thetis: `buildStreamPayload()` in `Project Files/Source/Console/TCIServer.cs` di `ramdor/Thetis` alloca 64 byte piu' payload, scrive receiver, sample rate, tipo di campione, due zeri, lunghezza, tipo di stream e canali come `uint32`, poi otto parole di riserva, e copia i campioni a offset 64. E' l'header TCI standard, campo per campo.

I presunti otto byte sono i primi otto di quello standard, letti con un layout inventato che combacia solo perche' il receiver 0 riempie di zeri i posti attesi. Il campanello e' la terza osservazione del commento, che esprime sorpresa nel trovare un sample rate dove si aspettava un conteggio: all'offset 4 dell'header standard c'e' esattamente un sample rate.

**Quindi TOTW suona da offset 8 anche i frame di Thetis**, portandosi 56 byte di header nel flusso audio. Quei byte sono interi piccoli e zeri: come `float32` sono denormali, cioe' silenzio. L'artefatto non e' rumore ma un buco periodico, che la dissolvenza di 64 campioni applicata nella stessa funzione trasforma in modulazione di ampiezza a 93,75 Hz. Su una portante CW stabile si sente come ronzio; sul parlato e' mascherato.

Questa e' con ogni probabilita' la issue #12 aperta su n9bc, il ronzio nelle portanti CW marcato come problema di vecchia data.

**Nessuna segnalazione a monte.** Decisione dell'operatore: non si disturba un collega per una svista del suo assistente. Il materiale resta qui se un giorno si vorra' contribuire.

## Decisioni prese (non riaprire senza motivo)

- **CLI-09 non e' un'unita'.** Servire il client in https e' un vincolo di deployment e appartiene al runbook della stazione, insieme a WireGuard e Mumble. L'unica parte che e' codice e' il degrado, assorbita in C6 (#7).
- **Il degrado e' incondizionato e deve essere forzabile.** Non dipende da come il client e' servito. Va provato con parametri di query (`?no-midi`, `?no-mic`, `?insecure`), altrimenti resta codice che nessuno esegue mai. La simulazione reale, LAN in chiaro e Safari, serve a confermare i parametri.
- **CLI-08, l'audio Opus in TCI, resta fuori.** L'audio su WAN va su Mumble. Riaprirlo significa riaprire DEC-01 nei requisiti.
- **Il file resta uno**, vanilla JS, senza build step.
- **Il CW non entra nella tabella MIDI**: ha un percorso proprio nel tool separato.

## Cosa manca

- **Il clic all'ascolto.** E' corroborazione, non prova: la questione dell'header e' gia' chiusa sul sorgente. Non blocca C1.
- Tutto il codice. Le tre unita' obbligatorie sono #2, #3 e #4, in quest'ordine.

Un'anomalia osservata e scartata: `vfoA: null` nel dump diagnostico. Nel browser il VFO funziona, `S.vfoA` nasce con un valore non nullo, quindi era stato transitorio. Non e' una issue.

## Come riprendere

```bash
cd ~/Developer/thetis-on-the-web && git fetch upstream && git status
python3 -m http.server 8000
```

Poi `http://localhost:8000/totw.html`, con la porta **40001** scritta a mano nel campo di connessione: il default di TOTW stock e' 50001, che e' quella di Thetis.

Serve deskHPSDR in esecuzione con la radio e TCI abilitato. Dopo la connessione, controllare che il sample rate della radio non sia rimasto a 192 kHz.
