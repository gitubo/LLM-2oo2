# Modello di Osservabilità

## Principio fondamentale

L'osservabilità non è un layer che si aggiunge al sistema dopo che è stato progettato. È una proprietà che deve essere progettata dentro ogni componente dall'inizio. Questa affermazione, spesso ripetuta come principio astratto, ha una conseguenza pratica precisa: le informazioni necessarie per capire il comportamento del sistema devono essere emesse dai componenti stessi nel momento in cui elaborano i dati, non ricostruite dopo.

Aggiungere osservabilità dopo significa riaprire ogni componente per aggiungere logging e metriche, scoprendo spesso che alcune informazioni critiche non sono più disponibili nel punto in cui servirebbero perché erano state scartate o trasformate da elaborazioni precedenti.

---

## I tre livelli

L'osservabilità del sistema si struttura su tre livelli distinti che rispondono a domande diverse.

Il logging risponde alla domanda "cosa è successo". Ogni componente emette eventi strutturati che descrivono le operazioni eseguite, le decisioni prese, e gli esiti. I log sono l'unica fonte di verità su cosa ha fatto il sistema su un input specifico.

Le metriche rispondono alla domanda "quanto spesso succede e quanto ci vuole". Sono aggregazioni statistiche dei log che permettono di capire il comportamento del sistema nel tempo, identificare trend, e definire alert.

Il tracing risponde alla domanda "come un singolo input ha attraversato il sistema". Permette di ricostruire il percorso completo di una singola richiesta attraverso tutti i componenti, correlando eventi che si sono verificati in momenti diversi e su componenti diversi.

---

## Il trace_id come elemento fondante

Ogni input che entra nel sistema riceve un trace_id nel momento in cui viene creato, prima ancora dell'Intent Parser. Questo identificatore viene propagato immutato attraverso tutti i componenti di tutta la pipeline, su entrambi i canali.

Il trace_id non deve essere generato internamente al sistema: deve essere fornito dal sistema chiamante. Questo permette di correlare il comportamento della pipeline di pianificazione con il comportamento del sistema che la chiama, ricostruendo scenari che attraversano confini di sistema.

Ogni evento emesso da ogni componente include il trace_id come campo primario. Questo permette di aggregare tutti gli eventi relativi a una singola richiesta e ricostruire la sua storia completa.

---

## Gli eventi per componente

Ogni componente emette eventi strutturati con un insieme minimo di campi comuni: trace_id, nome del componente, canale di appartenenza (A o B, dove applicabile), timestamp, esito, e durata in millisecondi. A questi si aggiungono campi specifici del componente.

L'Intent Parser emette la confidence della classificazione, il tipo di intent rilevato, il numero di ambiguità trovate, se è stato usato il path deterministico o il path LLM nel caso di approccio ibrido, e se è stata necessaria una richiesta di chiarimento.

Il Pre-Validator emette solo l'esito e, in caso di fallimento, la categoria dell'errore.

Il Sanitizer emette la lista completa delle correzioni applicate, ciascuna con il campo modificato, il valore originale, il valore applicato, la regola usata, e una stima della confidence della correzione.

Il Full Schema Validator emette l'esito e, in caso di fallimento, la lista degli errori di validazione con i path JSON precisi.

Il Semantic Validator emette l'esito, le regole verificate, e in caso di fallimento le regole violate con dettaglio.

L'Optimizer emette le trasformazioni applicate, ciascuna con la regola usata e i nodi coinvolti.

Il Logical Binding emette per ogni nodo la categoria selezionata, le categorie eliminate e i motivi dell'eliminazione.

Il Comparatore emette il verdetto, il livello al quale è stato trovato il disaccordo, il diff strutturato, e la categorizzazione del disaccordo.

Il Physical Binding emette per ogni nodo la risorsa nominale, la risorsa effettiva, e in caso di fallover il motivo.

---

## Le metriche che contano

Non tutte le metriche sono ugualmente utili. Il criterio di selezione è l'azionabilità: una metrica è utile se, quando mostra un'anomalia, suggerisce un'azione specifica.

Il tasso di correzione del Sanitizer per regola è la metrica più diagnostica per la qualità dei modelli LLM. Una regola che corregge frequentemente segnala un problema sistematico nel prompt che ha una soluzione diretta.

La distribuzione della confidence dell'Intent Parser, in particolare i percentili bassi, rivela la frequenza dei casi edge non coperti dalla tassonomia degli intent. Un peggioramento del p5 nel tempo indica che il dominio degli input sta evolvendo oltre le aspettative del design.

Il tasso di disaccordo del Comparatore per categoria è l'indicatore di salute della pipeline nel suo insieme. Un tasso stabile è rassicurante. Un aumento del tasso di disaccordo topologico può segnalare un aggiornamento di versione di uno dei modelli che ha cambiato il comportamento. Un aumento del tasso di disaccordo di binding può segnalare un'inconsistenza nel modo in cui i due modelli producono i binding constraints.

La latenza per stadio ai percentili p50, p95, e p99 permette di identificare i colli di bottiglia e di pianificare i timeout in modo realistico. Il p99 è più importante della media per questa analisi.

Il tasso di escalation per motivo mostra dove il sistema ha bisogno di intervento umano con più frequenza. Un tasso di escalation elevato per comparator_disagree suggerisce che la normalizzazione dell'Optimizer non è sufficiente per il dominio corrente.

---

## La differenza tra alert operativi e alert di qualità

Gli alert operativi segnalano un problema che richiede intervento immediato. Il sistema non sta funzionando o sta funzionando in modo degradato in modo visibile agli utenti. Esempi: latenza p99 oltre la soglia di timeout, tasso di errore al Pre-Validator oltre il 20%, tasso di escalation oltre il 30%.

Gli alert di qualità segnalano un degrado lento che non è ancora visibile agli utenti ma richiede attenzione nel breve-medio termine. Non svegliano nessuno di notte ma vengono esaminati nella routine di manutenzione. Esempi: confidence dell'Intent Parser in calo del 10% nel corso di una settimana, tasso di correzione del Sanitizer su un campo specifico in aumento del 50% rispetto alla settimana precedente, apparizione di una nuova categoria di disaccordo al Comparatore non vista in precedenza.

La distinzione è importante perché i due tipi di alert richiedono processi di risposta diversi e hanno soglie diverse.

---

## Il canary system

Una piccola percentuale del traffico reale, dell'ordine del 5-10%, viene continuamente valutata in shadow mode contro il golden dataset: i piani prodotti vengono confrontati con i piani attesi per quel tipo di input, e il tasso di corrispondenza viene monitorato come metrica di qualità.

Il canary system è l'unico meccanismo che permette di rilevare il degrado lento del sistema prima che diventi visibile. La distribuzione degli input in produzione cambia nel tempo, i modelli LLM vengono aggiornati dai fornitori, le regole di dominio invecchiano. Senza un sistema di monitoraggio continuo della qualità effettiva, questi cambiamenti sono invisibili finché non producono un problema visibile.

---

## La correlazione tra osservabilità e feedback loop

I dati di osservabilità sono la materia prima del feedback loop. Senza eventi strutturati e metriche aggregate non è possibile identificare sistematicamente dove e come migliorare il sistema.

La relazione va progettata esplicitamente: quali metriche alimentano quali decisioni di miglioramento, con quale frequenza, e con quale processo di review. Un dato di osservabilità che viene emesso ma non viene mai letto o usato è rumore, non informazione.

---

## Punti aperti

La scelta degli strumenti di raccolta, aggregazione, e visualizzazione dei dati di osservabilità è ortogonale al design del sistema ma ha impatti pratici significativi. Sistemi di osservabilità eccessivamente complessi tendono a non essere usati; sistemi troppo semplici non forniscono le informazioni necessarie.

La retention dei log strutturati deve essere bilanciata con i costi di storage. Log ad alta frequenza con molto dettaglio sono necessari per il debugging di casi specifici ma costosi da mantenere a lungo termine. Una strategia di retention differenziata per livello di dettaglio e per esito (tenere più a lungo i log degli errori rispetto a quelli dei successi) è comune ma richiede design esplicito.

Il GDPR e i requisiti di privacy possono imporre vincoli sulla durata della retention e sul contenuto dei log. Se gli input degli utenti contengono dati personali, la pipeline deve garantire che questi non vengano loggati in chiaro o che vengano anonimizzati prima della persistenza. Questo va considerato nel design dell'osservabilità, non aggiunto dopo.
