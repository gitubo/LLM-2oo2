# Logical Binding

## Scopo

Il Logical Binding è il componente che arricchisce ogni nodo del piano canonico con informazioni sulla categoria di implementazione da usare per eseguire quel task. Non risolve risorse fisiche (quale server, quale connessione, quale endpoint specifico) ma decide quale tipo di implementazione è semanticamente appropriato per quel task nel contesto dell'intent.

La distinzione tra Logical Binding e Physical Binding è la distinzione tra una decisione semantica e una decisione infrastrutturale. Quale tipo di sorgente dati usare (database, cache, API) è una decisione semantica che dipende dall'intent e dal dominio. Quale istanza specifica di quel tipo usare in questo momento è una decisione infrastrutturale che dipende dallo stato del sistema.

Questa separazione è necessaria perché il Logical Binding avviene prima del Comparatore e deve essere deterministico e replicabile su entrambi i canali, mentre il Physical Binding avviene dopo e può dipendere da condizioni runtime.

---

## La relazione con il Capability Registry

Nel contesto specifico di questa architettura, dove ogni nodo contestuale si traduce in una chiamata HTTP a un'API esterna definita nel registry, il Logical Binding e il registry si sovrappongono parzialmente.

Il registry dichiara per ogni azione il contratto HTTP: metodo, endpoint, e convenzione di traduzione dei parametri. Questo è esattamente la decisione che il Logical Binding deve prendere per i nodi contestuali. In questo senso, il registry è già una forma di Logical Binding dichiarativo per la categoria di implementazione HTTP.

La distinzione rimane rilevante per i casi in cui esistono più implementazioni possibili per la stessa risorsa — il registry descrive l'API esterna canonica, ma potrebbero esistere alternative come cache, repliche, o sistemi legacy che il Logical Binding può selezionare basandosi sui binding constraints dell'intent. Nei casi più semplici, dove una sola implementazione è disponibile per ogni risorsa, il Logical Binding si riduce a leggere il contratto HTTP dal registry.

---

## Il problema che risolve

Un nodo del piano con tipo "fetch" e entità "orders" può essere eseguito in modi diversi: una query diretta al database primario, una lettura dalla cache, una chiamata a un servizio di business logic, una query all'API di un sistema legacy. Queste implementazioni non sono equivalenti: hanno caratteristiche diverse di latenza, freschezza dei dati, side effect, e costo.

La scelta tra queste implementazioni non è arbitraria né puramente infrastrutturale. Dipende dall'intent: se l'utente ha chiesto dati in tempo reale, la cache non è appropriata. Se l'utente ha chiesto dati storici oltre una certa data, solo il sistema legacy li ha. Questa è conoscenza di dominio, non configurazione infrastrutturale.

Il Logical Binding materializza questa conoscenza in modo deterministico, producendo una scelta che può essere verificata e confrontata.

---

## Come funziona

Il Logical Binding riceve per ogni nodo del piano il tipo di operazione, i parametri, e l'intent strutturato con i suoi binding constraints. Applica una serie di regole deterministiche che selezionano la categoria di implementazione appropriata.

Le regole seguono una logica di eliminazione: a partire dall'insieme completo delle implementazioni disponibili per quel tipo di operazione, le regole eliminano quelle incompatibili con i vincoli dell'intent fino a identificare la categoria appropriata. Se rimane più di una categoria dopo l'eliminazione, si applica una regola di preferenza di default che può essere configurata per il dominio.

I binding constraints prodotti dall'LLM durante la generazione del piano (requires_realtime, side_effects_acceptable, ecc.) sono input del Logical Binding ma non sono l'unica fonte di decisione. Vengono integrati con le regole di dominio codificate nel componente, che possono avere priorità superiore.

---

## La relazione con l'intent strutturato

Il Logical Binding ha bisogno dell'intent strutturato per due motivi distinti.

Il primo è l'uso diretto dei binding constraints: se l'intent specifica requires_realtime: true, il Logical Binding usa questa informazione per escludere le implementazioni basate su cache.

Il secondo è la verifica di coerenza: i binding constraints prodotti dall'LLM vengono verificati contro l'intent. Se l'LLM ha prodotto requires_realtime: false ma l'intent originale richiedeva dati in tempo reale, questa inconsistenza viene rilevata qui e segnalata, prima che produca un binding sbagliato.

---

## Il Logical Binding come arricchimento del piano canonico

L'output del Logical Binding è il piano canonico arricchito con un campo binding per ogni nodo, che contiene la categoria di implementazione selezionata e le motivazioni della scelta. Questo campo diventa parte del piano che viene poi confrontato dal Comparatore.

Il fatto che il binding logico faccia parte del confronto è una caratteristica importante dell'architettura: se i due canali hanno prodotto piani canonici identici ma il Logical Binding ha scelto categorie di implementazione diverse per lo stesso nodo, il Comparatore rileva questo disaccordo. Questo può succedere se i binding constraints dei due piani sono inconsistenti, e il disaccordo è un segnale informativo.

---

## Esempio pratico

Un piano canonico contiene un nodo fetch per l'entità orders. L'intent strutturato contiene requires_realtime: true e side_effects_acceptable: false.

Il Logical Binding ha a disposizione queste categorie per il fetch di orders: database_primary (real-time, no side effect), cache_layer (potenzialmente non aggiornato, no side effect), order_service (real-time, con side effect di audit log), legacy_api (storico oltre 90 giorni, no side effect).

La regola requires_realtime: true elimina cache_layer. La regola side_effects_acceptable: false elimina order_service. La legacy_api viene eliminata perché il campo created_at nell'intent non è nel range storico. Rimane database_primary.

Il nodo viene arricchito con binding: { category: "database_primary", reason: "realtime_required, no_side_effects" }.

---

## Pro dell'approccio

La separazione delle responsabilità tra Logical e Physical Binding evita che condizioni runtime transitorie inquinino la validazione del piano. Il Logical Binding è deterministico e stabile, il che lo rende adatto a essere replicato su due canali e confrontato.

Avere la scelta del tipo di implementazione documentata come parte del piano arricchisce enormemente l'osservabilità: è possibile auditare non solo cosa è stato fatto ma perché è stato scelto quel tipo di implementazione.

---

## Contro, dubbi e punti aperti

Il Logical Binding condivide con il Sanitizer il problema di essere identico su entrambi i canali. Un bug nelle regole di selezione si manifesta su entrambi i canali producendo accordo al Comparatore, anche se la scelta è sbagliata.

La dipendenza dai binding constraints prodotti dall'LLM introduce un rischio: se l'LLM produce constraints inconsistenti o sbagliati, il Logical Binding fa la scelta sbagliata con fiducia. La verifica di coerenza tra constraints e intent mitiga questo rischio ma non lo elimina completamente.

La gestione dell'aggiunta di nuove categorie di implementazione richiede di aggiornare le regole del Logical Binding. Se una nuova categoria viene aggiunta senza aggiornare le regole, non viene mai selezionata. Se viene aggiunta ma le regole di eliminazione non la considerano correttamente, può essere selezionata in casi inappropriati. Questo richiede un processo di deploy coordinato.

Un punto aperto più profondo riguarda i casi dove nessuna categoria disponibile soddisfa tutti i vincoli dell'intent. Il Logical Binding deve avere un comportamento definito per questo scenario: segnalare il problema e bloccare il piano, scegliere la categoria meno incompatibile con una segnalazione, o escalare verso una decisione umana. La scelta dipende dal dominio e dalla criticità dell'operazione.
