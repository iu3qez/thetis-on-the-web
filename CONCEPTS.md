# Concepts

Shared domain vocabulary for this project — entities, named processes, and status concepts with project-specific meaning. Seeded with core domain vocabulary, then accretes as ce-compound and ce-compound-refresh process learnings; direct edits are fine. Glossary only, not a spec or catch-all.

## Lavoro e verifica

### Unita' d'implementazione
La piu' piccola porzione di lavoro che il piano assegna e che si integra da sola, identificata da una sigla e tracciata da una issue.

Le unita' dichiarano le proprie dipendenze, e l'ordine fra due di esse puo' essere vincolante e non soltanto consigliato: quando i difetti che due unita' correggono si mascherano a vicenda, correggerne uno da solo produce un peggioramento visibile che si legge come regressione. Un'unita' implementata e integrata non e' ancora un'unita' accettata: le manca la verifica al banco.

### Ricognizione
La passata di accertamento condotta sul client non modificato, prima di scrivere codice, per confermare che i difetti previsti dall'analisi si manifestino davvero.

Ha condizioni di stop dichiarate in anticipo: se il sintomo atteso non compare, l'analisi va rivista prima dell'implementazione invece di procedere. Una condizione di stop vale solo se il suo esito cambia a seconda che l'analisi sia giusta o sbagliata; le condizioni formulate su cio' che si vede sullo schermo spesso non hanno questa proprieta'.

### Verifica al banco
La verifica condotta dall'operatore con la radio accesa, distinta dai controlli che girano senza hardware.

E' la condizione che chiude l'unita', e non e' surrogabile: i controlli senza hardware possono dire che il codice fa cio' che il codice dice, non che la stazione funziona. Ne segue che un'unita' puo' restare a lungo integrata e non accettata.

## Artefatti del client

### Variante portabile
L'artefatto autocontenuto del client: un singolo documento con stile e codice incorporati, che si copia su una chiavetta e si apre in un browser senza server ne' compilazione. E' la proprieta' piu' utile del progetto e vincola ogni unita' a stare dentro quel file.

### Variante scomposta
La stessa applicazione con stile e codice in file separati, pensata per essere servita da un sito.

E' derivata dalla variante portabile, non parallela a essa: si rigenera e non si modifica a mano. Lo strumento di generazione funziona nei due versi, quindi una modifica fatta sulla variante scomposta viene cancellata senza avviso alla rigenerazione successiva.

## Spettro

### Sorgente spettro
Il flusso da cui la traccia dello spettro viene alimentata. Sono due: i campioni IQ, su cui il client calcola la trasformata localmente, e i bin gia' decimati calcolati dal server.

La traccia disegnata e' una sola e non sa da quale delle due arriva; la scelta e' esplicita e non dedotta, perche' l'operatore sa meglio di qualunque euristica se si trova in rete locale o su un collegamento stretto. Le due sorgenti differiscono per banda occupata di oltre un ordine di grandezza, ed e' questa la ragione per cui la seconda esiste.

### Liveness della traccia
La proprieta' che distingue una traccia che viene aggiornata da una che e' soltanto disegnata.

Non coincide con la disponibilita' dei dati: una traccia puo' restare visibile indefinitamente mostrando l'ultimo contenuto ricevuto, se il flag che ne autorizza il disegno non viene invalidato dalla stessa condizione che lo produce. Il discriminante affidabile e' un contatore di frame che avanza fra due letture, mai lo stato del disegno.
