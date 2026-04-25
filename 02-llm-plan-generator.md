# LLM Plan Generator

## Scopo

Il Plan Generator è il componente che produce il piano d'azione a partire dall'intent strutturato. Nell'architettura 2oo2 questo componente è replicato su due canali paralleli e indipendenti. I due modelli ricevono lo stesso input, lavorano separatamente, e producono output che verranno poi confrontati. La loro indipendenza è il presupposto fondamentale del confronto: se i due canali fossero identici, il comparatore non aggiungerebbe alcun valore.

Il piano prodotto è astratto: descrive cosa fare, in quale ordine, con quali dipendenze, ma non come farlo fisicamente. La traduzione in istruzioni concrete eseguibili avviene più avanti nella pipeline.

---

## Cosa produce

L'output atteso è un documento JSON che rappresenta un grafo diretto aciclico di task atomici. Ogni nodo del grafo ha un identificatore, un tipo di operazione che appartiene a un vocabolario definito, un insieme di dipendenze verso altri nodi, e i parametri specifici dell'operazione.

Un esempio concettuale: una richiesta di recupero degli ultimi dieci ordini confermati di clienti premium potrebbe produrre un grafo con quattro nodi in sequenza: un fetch dell'entità orders, un filter per status e categoria cliente, un sort per data di creazione discendente, e un limit a dieci elementi. Le dipendenze tra i nodi definiscono l'ordine di esecuzione.

Il piano non contiene riferimenti a implementazioni specifiche, connessioni, endpoint, o risorse infrastrutturali. Quella è responsabilità del binding, che viene dopo.

---

## La scelta dei due modelli

La selezione dei due modelli non è arbitraria. L'obiettivo è minimizzare la correlazione degli errori, il che richiede che i due modelli abbiano differenze sostanziali nel training, nell'architettura, o nel fornitore.

Usare due istanze dello stesso modello con parametri diversi non è sufficiente. La correlazione rimarrebbe alta perché entrambi condividono gli stessi bias di training. La scelta ottimale è tipicamente un modello di un fornitore rispetto a un modello di un fornitore diverso, oppure un modello generico rispetto a un modello fine-tuned sul dominio specifico.

La correlazione residua non si annulla mai del tutto. Tutti i modelli linguistici moderni sono stati addestrati su corpus sovrapposti e tendono a sbagliare sulle stesse categorie di input difficili. Ridurre la correlazione è un obiettivo di approssimazione, non di eliminazione.

---

## Il ruolo dell'intent strutturato

Il Plan Generator non lavora sul testo originale dell'utente. Riceve l'intent strutturato prodotto dall'Intent Parser. Questo è una scelta progettuale deliberata con conseguenze importanti.

Il modello non deve interpretare il linguaggio naturale: questa responsabilità è già stata gestita a monte. Deve invece tradurre una specifica formale in un piano formale. Il task è più vincolato, il perimetro di variabilità è più stretto, e il risultato è più confrontabile tra i due canali.

L'intent strutturato include anche i binding constraints, che il Plan Generator può usare per prendere decisioni sul piano. Se l'intent specifica che i dati devono essere in tempo reale, il modello può scegliere di non includere task di caching che avrebbero introdotto latenza potenzialmente stale.

---

## La generazione parallela

I due canali vengono avviati in parallelo non appena l'Intent Parser ha prodotto il proprio output. Non c'è un canale primario e uno secondario: entrambi hanno la stessa dignità fino al confronto.

La latenza della generazione è tipicamente il collo di bottiglia principale dell'intera pipeline. Avviare i due modelli in parallelo significa che la latenza complessiva è determinata dal canale più lento, non dalla somma dei due.

---

## I tipi di errore tipici

I modelli linguistici che generano piani strutturati tendono a sbagliare in modi caratteristici che vale la pena conoscere.

L'omissione è il più insidioso: il modello genera un piano plausibile e internamente coerente che manca di un task necessario. Un filtro che dovrebbe precedere un sort, un task di lock che dovrebbe precedere una scrittura. Il piano passa tutti i controlli strutturali perché è ben formato, ma è semanticamente incompleto.

L'ordinamento errato si manifesta quando le dipendenze tra task vengono specificate in modo che non riflette le necessità semantiche dell'operazione. Il grafo può essere aciclico e formalmente valido ma topologicamente sbagliato rispetto all'intent.

La granularità errata può andare in entrambe le direzioni: il modello può collassare operazioni che dovevano essere separate o espandere operazioni che dovevano essere atomiche. Questo crea problemi di comparabilità tra i due canali anche quando entrambi hanno ragione.

L'allucinazione di task non necessari è meno comune ma possibile: il modello include task che non servono all'intent, magari perché li ha visti frequentemente in esempi di training simili.

---

## La questione del prompt

La qualità dell'output del Plan Generator dipende in modo critico dalla qualità del prompt. Un prompt che descrive in modo preciso il vocabolario dei task ammessi, le dipendenze tipiche, i vincoli strutturali del piano, e i casi edge comuni produce output molto più consistenti di un prompt generico.

Il prompt va trattato come codice: versionato, testato, revisionato quando cambiano i requisiti. Le modifiche al prompt devono essere validate sul golden dataset prima del deploy in produzione.

Un punto che emerge spesso in pratica è che i due canali, pur usando modelli diversi, possono condividere lo stesso schema di prompt con solo le istruzioni specifiche del modello adattate. Questo semplifica la manutenzione ma introduce una correlazione a livello di prompt che va considerata.

---

## Pro dell'approccio

Avere due generatori indipendenti crea una naturale diversità di prospettiva che il comparatore può sfruttare. Se un modello tende a essere conservativo (piani più semplici, meno task) e l'altro tende a essere più elaborato, il disaccordo che emerge è spesso informativo: la verità sta da qualche parte tra i due.

La generazione parallela, oltre al beneficio di latenza, ha un vantaggio meno ovvio: se uno dei due modelli produce output irrecuperabile (pre-validation failure), l'altro può ancora procedere. Il sistema non si blocca completamente.

---

## Contro, dubbi e punti aperti

La correlazione degli errori, già discussa nella scelta dei modelli, rimane il limite strutturale più serio. Non si può eliminare, solo gestire con scelte oculate di modelli e di prompt.

Il costo è il doppio rispetto a un singolo modello, sia in termini di token che di latenza. In sistemi ad alto volume questo ha impatti economici non trascurabili che vanno bilanciati con il beneficio della validazione.

La gestione delle versioni dei modelli è un problema pratico rilevante. I fornitori aggiornano i modelli periodicamente, a volte senza preavviso significativo. Un aggiornamento del modello su uno dei due canali può cambiare il comportamento del comparatore in modo non prevedibile. Serve un processo di testing che verifichi il comportamento del sistema ogni volta che uno dei due modelli cambia versione.

Un punto aperto non ancora completamente risolto riguarda il caso in cui i due canali producano piani con granularità diversa ma semanticamente equivalenti. Il comparatore vede un disaccordo strutturale che non è un errore, è una differenza di stile. L'ottimizzatore mitiga questo problema portando i piani in forma canonica, ma non può eliminarlo completamente per tutte le possibili variazioni.
