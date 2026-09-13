---
name: Thetis On The Web
last_updated: 2026-09-13
---

# Thetis On The Web Strategy

Thetis On The Web è l'interfaccia browser della stazione remota descritta nei requisiti di deskHPSDR; questo documento copre solo il client.

## Purpose

L'operatore vuole usare la propria stazione HPSDR da remoto, da un portatile, con un'interfaccia radio che sta tutta nel browser. Il nodo è il collegamento: i client esistenti presumono la LAN, e spettro e audio a piena risoluzione costano megabit che su un 4G non ci sono.

## Positioning

Possediamo entrambi i capi del collegamento: il server calcola quello che il client deve mostrare e manda solo quello, così il client resta un file da browser che funziona anche su un 4G. I percorsi critici, audio e morse, non passano da qui; il resto è best effort.

## Users

**Primary:** Il radioamatore proprietario della stazione, da solo, lontano dallo shack - Affida a Thetis On The Web il compito di condurre un QSO come se fosse davanti alla radio: vedere la banda, sintonizzare, pilotare la stazione.

## Boundaries

- Compatibilità con Thetis o con altri server TCI: si lavora solo con deskHPSDR.
- iPad, telefoni e iOS: il bersaglio è un portatile con browser desktop.
- Audio e morse nel browser: passano per trasporti propri, fuori da questo client.
- Multi-operatore: non in questa fase.
- Funzioni ereditate che chiamano servizi esterni: nessun investimento, si tolgono se costano banda o danno problemi.
- Il progetto originale: non si segue e non si cita.

_Resist a change when:_ porta nel client un percorso critico, lo lega a un server che non è deskHPSDR, o spende banda del 4G per qualcosa che la stazione non usa.

## Key metrics

- **QSO completi da remoto** - contatti conclusi senza tornare allo shack; dal log di stazione.
- **Sessioni perse per il collegamento** - sessioni remote abbandonate perché spettro, audio o controllo erano inutilizzabili; annotate dall'operatore.
- **Banda per sessione** - kbit/s medi di una sessione tipica con spettro a bin, tetto 50; dal monitor di rete del client.
- **Ritardo di comando su 4G** - tempo fra un'azione dalla console e la conferma del server; misura da introdurre.

## Tracks

### Spettro su banda stretta

Lo spettro calcolato dal server arriva come bin e alimenta la traccia: stream a bin, span che segue lo zoom, scelta esplicita fra bin e IQ.

_Why it serves the approach:_ è il lavoro spostato sul server, ed è ciò che porta lo spettro sotto il tetto di banda.

### Pannello fisico

La stazione si pilota da una console MIDI come davanti alla radio: jog sul VFO, comandi a due stati, LED di stato.

_Why it serves the approach:_ banda e attenuatore passano per i comandi estesi che esistono solo perché il server è vostro.

## Brand

**One-liner:** Fast and lean
