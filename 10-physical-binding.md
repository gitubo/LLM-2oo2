# Physical Binding

## Scopo

Il Physical Binding è il componente che traduce il piano concordato, con il suo binding logico già definito, in un piano concretamente eseguibile. Dove il Logical Binding aveva stabilito quale categoria di implementazione usare per ogni nodo, il Physical Binding stabilisce quale risorsa fisica specifica usare in questo momento: quale istanza del database, quale endpoint dell'API, quale nodo del cluster di cache.

Opera dopo il Comparatore, in un momento in cui il piano ha già ricevuto la validazione dell'intera pipeline. Questo posizionamento è una scelta progettuale deliberata: le condizioni runtime non devono influenzare le decisioni di pianificazione e validazione.

---

## Perché avviene dopo il Comparatore

La motivazione principale è che le condizioni runtime sono per natura non deterministiche e mutevoli. Includere decisioni infrastrutturali nella fase di validazione significherebbe che un timeout di rete, un failover, o una variazione di carico potrebbero produrre risoluzioni diverse sui due canali, creando disaccordi al Comparatore che non hanno nulla a che fare con la qualità del piano.

Il Logical Binding garantisce che la categoria di implementazione sia stata scelta in modo deterministico e comparabile. Il Physical Binding poi risolve, una sola volta, quale risorsa fisica di quella categoria usare in questo specifico momento.

---

## Cosa fa concretamente

Per ogni nodo del piano concordato, il Physical Binding legge la categoria di implementazione stabilita dal Logical Binding e risolve la risorsa fisica appropriata usando la conoscenza dello stato corrente del sistema.

Per una categoria database_primary, potrebbe risolvere al database primario se è disponibile, oppure a una replica in lettura se il primario è sotto manutenzione, con segnalazione esplicita del fallover nel log.

Per una categoria cache_layer, risolve all'istanza di cache appropriata per quella entità e verifica che il TTL residuo sia compatibile con i vincoli di freschezza dell'intent (se presenti).

Per una categoria service, risolve all'endpoint del servizio disponibile, gestendo eventuali meccanismi di load balancing e circuit breaker.

---

## Il fallover e la sua visibilità

Il Physical Binding è il luogo dove i fallover infrastrutturali si materializzano. Quando la risorsa preferita non è disponibile e viene usata un'alternativa, questo evento deve essere loggato in modo esplicito e strutturato, non gestito silenziosamente.

Un fallover silenzioso è pericoloso perché il piano viene eseguito su una risorsa diversa da quella nominale, il che può avere implicazioni sulla correttezza o sulla freschezza dei dati, e nessun componente a valle ne è consapevole.

Il log del Physical Binding deve includere per ogni nodo la categoria richiesta, la risorsa nominale, la risorsa effettivamente usata, e in caso di fallover il motivo. Queste informazioni permettono di correlare eventuali anomalie nell'output con condizioni infrastrutturali al momento dell'esecuzione.

---

## La finestra temporale tra validazione e esecuzione

Il Physical Binding opera dopo una pipeline che ha una latenza cumulativa dell'ordine di 1-2 secondi. Questo significa che esiste sempre una finestra temporale tra il momento in cui il piano è stato validato e il momento in cui viene risolto fisicamente e poi eseguito.

In questa finestra, lo stato del sistema può cambiare: una risorsa può diventare non disponibile, un dato può essere aggiornato, un lock può essere acquisito da un altro processo. Il Physical Binding gestisce i cambiamenti infrastrutturali (la risorsa è disponibile o no), ma non può gestire i cambiamenti di stato dei dati che il piano andrà a leggere o modificare.

Questo è un limite intrinseco dell'architettura che va riconosciuto e documentato. Per operazioni che richiedono consistenza assoluta, serve una strategia a livello di esecuzione (transazioni, lock espliciti) che non è gestita dalla pipeline di pianificazione.

---

## La relazione con l'esecuzione

Il Physical Binding produce come output un piano eseguibile: una struttura che contiene per ogni nodo non solo cosa fare e con quale categoria di implementazione, ma i dettagli concreti della risorsa (connection string, endpoint URL, parametri di connessione) necessari per avviare l'esecuzione.

Questo output è il confine tra la pipeline di pianificazione e il sistema di esecuzione. Il sistema di esecuzione assume che il piano sia corretto e completo, e lo esegue. Errori che emergono durante l'esecuzione (timeout, errori di business logic, dati non trovati) sono gestiti dal sistema di esecuzione con le proprie logiche di retry e error handling, non dalla pipeline di pianificazione.

---

## Esempio pratico

Il piano concordato contiene un nodo fetch(orders) con binding logico category: database_primary. Il Physical Binding interroga il service discovery per trovare l'istanza corrente del database primario, verifica che sia raggiungibile, e risolve la connessione specifica.

In questo momento, il database primario è sotto manutenzione. Il Physical Binding rileva la condizione, registra nel log: "risorsa nominale db-primary non disponibile, fallover a db-replica-2", e risolve la connessione alla replica in lettura.

Il log include la nota che i dati potrebbero avere un ritardo di replica dell'ordine di pochi secondi. Se l'intent aveva specificato requires_realtime con tolleranza zero, questo fallover dovrebbe produrre un errore invece di un fallover silenzioso. Il Physical Binding deve essere in grado di verificare questa compatibilità.

---

## Pro dell'approccio

Separare il Physical Binding dalla fase di validazione mantiene la pipeline deterministica e riproducibile. Lo stesso input produce sempre lo stesso piano canonico e lo stesso binding logico, indipendentemente dalle condizioni infrastrutturali del momento.

Il Physical Binding come componente esplicito rende i fallover infrastrutturali visibili e loggati, invece di gestirli silenziosamente nei punti di connessione sparsi nel codice di esecuzione.

---

## Contro, dubbi e punti aperti

Il Physical Binding è il componente della pipeline con meno supervisione. Opera dopo il Comparatore, che è l'ultimo checkpoint di qualità. I suoi log sono l'unico strumento di visibilità su quello che fa.

Un fallover silenzioso che produce dati leggermente stale può essere difficile da correlare con un'anomalia nell'output, specialmente se l'anomalia si manifesta molto dopo l'esecuzione. Investire in logging strutturato e in correlazione automatica tra log del Physical Binding e anomalie dell'output è essenziale.

Un punto aperto significativo riguarda la gestione del caso in cui nessuna risorsa fisica sia disponibile per una categoria di implementazione. Il piano è valido, il binding logico è corretto, ma l'infrastruttura non è in grado di servirlo in questo momento. Il sistema deve avere un comportamento definito per questo scenario: tipicamente un errore esplicito con dettaglio sulla risorsa non disponibile, che permette al chiamante di ritentare in un momento successivo. Gestire questo come eccezione invece che come errore strutturato è un pattern da evitare.
