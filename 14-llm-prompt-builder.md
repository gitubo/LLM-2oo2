# LLM Prompt Builder

## Scopo

Il LLM Prompt Builder è il componente che si interpone tra il Capability Registry e il Plan Generator. Il suo compito è leggere il registry e costruire il contesto da passare all'LLM: non il registry completo, ma esattamente le informazioni necessarie per generare il piano relativo alla richiesta corrente.

È un componente deterministico: dato lo stesso intent strutturato e lo stesso registry, produce sempre lo stesso prompt. Non c'è logica probabilistica, non ci sono decisioni ambigue. Questo lo rende testabile in isolamento e il suo output verificabile staticamente.

---

## Perché esiste come componente separato

La tentazione ovvia è includere il registry nel system prompt direttamente, passando la definizione completa di tutte le risorse all'LLM. Questa soluzione ha due problemi.

Il primo è il costo in token. Un registry con decine di risorse, ciascuna con schema, azioni e supporto per le operazioni trasversali, può essere molto grande. Passarlo intero per ogni richiesta è costoso e riempie la finestra di contesto con informazioni irrilevanti per la richiesta specifica.

Il secondo è la qualità del piano generato. Un LLM che riceve più informazioni del necessario tende a produrre piani più rumorosi: task che fanno riferimento a risorse non pertinenti, parametri copiati da esempi sbagliati, ambiguità nell'interpretazione di campi con nomi simili su risorse diverse. Meno contesto rilevante significa meno variabilità indesiderata.

Il LLM Prompt Builder risolve entrambi i problemi selezionando dal registry solo le risorse pertinenti alla richiesta e costruendo un contesto denso e preciso.

---

## Cosa produce

Il Prompt Builder produce due elementi distinti che vengono passati al Plan Generator.

Il system prompt contiene le istruzioni strutturali: il formato atteso dell'output, le regole che il Planner deve rispettare (usare solo risorse e campi dichiarati, non inferire, non ottimizzare), e il vocabolario completo delle risorse rilevanti estratto dal registry.

Il vocabolario per ciascuna risorsa include: la descrizione leggibile, i campi dello schema con i loro tipi e valori ammessi, le azioni disponibili con i relativi parametri, e per ogni azione i campi su cui è possibile filtrare, ordinare o limitare con i rispettivi operatori. Queste informazioni delimitano lo spazio delle scelte dell'LLM: non può usare un operatore non dichiarato, non può filtrare su un campo non presente nello schema, non può riferirsi a una risorsa non inclusa nel contesto.

---

## La selezione delle risorse rilevanti

La selezione delle risorse da includere nel contesto è la decisione più importante del Prompt Builder. Includere troppo degrada la qualità del piano. Escludere qualcosa di necessario rende impossibile generare il piano corretto.

La selezione può avvenire in modi diversi a seconda della complessità del dominio. In domini con poche risorse, includere sempre tutte è accettabile. In domini con molte risorse, si usa l'intent strutturato prodotto dall'Intent Parser come chiave: le entità menzionate nell'intent identificano le risorse primarie, e le risorse correlate (se il registry le dichiara) vengono incluse automaticamente.

Questa dipendenza dall'intent strutturato rafforza ulteriormente l'importanza dell'Intent Parser: un intent mal classificato può portare il Prompt Builder a includere le risorse sbagliate nel contesto, producendo un piano generato su basi errate.

---

## Il vincolo fondamentale sul comportamento del Planner

Il system prompt prodotto dal Prompt Builder deve includere un vincolo esplicito e non ambiguo: il Planner produce il piano più esplicito possibile senza ottimizzazioni. Non collassa mai nodi trasversali nel nodo contestuale precedente, non inferisce parametri non dichiarati, non abbrevia la linked list.

Questo vincolo è essenziale perché l'ottimizzazione è responsabilità dell'Optimizer, non del Planner. Se il Planner ottimizza, l'output dei due canali sarà influenzato da scelte diverse di ottimizzazione che l'Optimizer non riuscirà sempre a normalizzare, producendo falsi disaccordi al Comparatore.

La separazione di responsabilità tra Planner e Optimizer dipende interamente dalla qualità di questo vincolo nel prompt.

---

## La relazione con i due canali

Il Prompt Builder produce un solo prompt che viene passato identicamente a entrambi i canali del 2oo2. Questo è corretto: la variabilità tra i due canali deve venire dalla diversità dei modelli, non da prompt diversi. Un prompt diverso per i due canali introdurrebbe una variabile non controllata che renderebbe il disaccordo al Comparatore meno interpretabile.

---

## Esempio concettuale

Per una richiesta come "dammi i primi 10 utenti amministratori attivi", l'Intent Parser identifica la risorsa users. Il Prompt Builder legge dal registry la definizione di users e costruisce un contesto che include:

La descrizione della risorsa. I campi dello schema con i loro tipi: id (string), role (string con enum admin/viewer/editor), active (boolean), created_at (date-time), last_login_at (date-time). L'azione fetch con i suoi parametri diretti e i campi filtrabili: role con operatori eq e neq, active con operatore eq. Il supporto per sort sui campi email, role, created_at. L'assenza di supporto nativo per limit.

L'LLM riceve questo contesto e produce un piano che usa solo questi elementi. Non può inventare un campo status che non esiste nello schema. Non può usare un operatore gt su role perché non è dichiarato come supportato per quel campo.

---

## Pro dell'approccio

Il Prompt Builder trasforma il registry da documento di configurazione a strumento di vincolo attivo sul comportamento dell'LLM. I limiti del piano generabile non sono imposti solo dalla validazione a posteriori ma dalla costruzione del contesto a priori.

Questo riduce la frequenza dei fallimenti al Semantic Validator: un LLM che non ha mai visto nel contesto un operatore non valido ha meno probabilità di usarlo.

La natura deterministica del componente lo rende verificabile: per ogni intent strutturato e ogni versione del registry, il prompt prodotto è prevedibile e ispezionabile. Questo semplifica enormemente il debugging di comportamenti inattesi del Planner.

---

## Contro, dubbi e punti aperti

Il Prompt Builder è l'unico punto dove si decide cosa l'LLM può vedere. Questa centralità lo rende critico: un bug nella selezione delle risorse rilevanti è invisibile al Semantic Validator, perché il validator verifica la correttezza del piano rispetto al registry ma non verifica che il Planner abbia ricevuto il contesto giusto.

La selezione automatica delle risorse rilevanti dall'intent strutturato funziona bene per richieste semplici ma può fallire su richieste che attraversano più risorse in modo non esplicito. Se l'intent menziona un attributo di una risorsa correlata senza nominare la risorsa, il Prompt Builder potrebbe non includerla nel contesto.

Un punto aperto riguarda la gestione del contesto quando il registry evolve durante il ciclo di vita di una richiesta. In scenari con elaborazione asincrona o con retry, è possibile che il prompt venga costruito su una versione del registry diversa da quella usata per la validazione a valle. Serve una strategia esplicita per garantire la consistenza della versione del registry lungo tutta la pipeline per una singola richiesta.
