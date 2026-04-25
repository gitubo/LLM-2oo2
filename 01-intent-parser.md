# Intent Parser

## Scopo

L'Intent Parser è il primo componente della pipeline e il solo che non ha ridondanza. Il suo compito è trasformare un input in linguaggio naturale in una rappresentazione strutturata e non ambigua dell'obiettivo dell'utente. Quello che produce non è un piano d'azione, è una specifica: descrive cosa si vuole ottenere, con quali vincoli espliciti, con quale grado di certezza.

Tutto il resto del sistema lavora a partire dall'output dell'Intent Parser, assumendolo come ground truth. Questa posizione nella pipeline lo rende il componente con il maggiore impatto potenziale sulla correttezza complessiva: un errore qui si propaga silenziosamente attraverso tutti gli stadi successivi, con la piena fiducia del sistema.

---

## Cosa produce

L'output è un documento strutturato che contiene almeno i seguenti elementi:

Il tipo di intent, che appartiene a una tassonomia definita a priori. Non è una descrizione libera ma una classificazione in una categoria nota, come fetch_ordered_subset, aggregate_with_grouping, o transform_and_filter. Questa tassonomia va costruita e mantenuta dagli esperti di dominio.

I parametri dell'operazione, estratti dall'input: l'entità su cui operare, i filtri, i criteri di ordinamento, i limiti, e qualsiasi altro vincolo che l'utente abbia espresso esplicitamente.

I binding constraints, che sono i requisiti semantici dell'operazione che guidano poi la scelta delle implementazioni a valle. Esempi tipici sono la necessità di dati in tempo reale, l'accettabilità o meno di dati potenzialmente non aggiornati, la presenza o assenza di side effect accettabili.

Un campo ambiguities, che elenca esplicitamente gli aspetti dell'input che non è stato possibile determinare con sufficiente certezza. Questo campo è tra i più importanti dell'intero output perché trasforma un unknown implicito in un unknown esplicito, che il sistema può gestire consapevolmente.

Un valore di confidence, che indica quanto il parser è sicuro della propria interpretazione. Questo valore non è ornamentale: al di sotto di una certa soglia il sistema deve comportarsi diversamente, chiedendo chiarimento invece di procedere.

---

## Come si può implementare

Esistono tre approcci principali, ciascuno con caratteristiche diverse.

Un parser basato su grammatica formale definisce in modo esplicito le strutture sintattiche ammesse per ciascun tipo di intent e fa il parsing dell'input contro queste strutture. Il risultato è deterministico e testabile, con correlazione zero rispetto ai modelli LLM usati negli stadi successivi. Il limite è la rigidità: input formulati in modo insolito o che non rientrano nelle strutture previste dalla grammatica producono fallimenti o parsing parziali.

Un parser basato su LLM dedicato usa un modello linguistico con il solo scopo di estrarre l'intent strutturato. Non pianifica, non ragiona sull'esecuzione, si limita a classificare e a estrarre parametri. È robusto a formulazioni diverse, gestisce lingue e varianti stilistiche, ma reintroduce non-determinismo nella pipeline.

Un approccio ibrido usa il parser deterministico per gli intent comuni e ben formati, e il parser LLM per i casi che la grammatica non riesce a gestire. Il parser deterministico ha priorità e il parser LLM interviene solo come fallback. La confidence del risultato è calcolata differentemente nei due casi, riflettendo la diversa natura dei due approcci.

---

## Il problema dell'ambiguità genuina

Alcuni input sono ambigui non per limitazioni del parser ma per limitazioni dell'input stesso. L'esempio classico è una richiesta come "i migliori clienti": migliori per fatturato, per numero di ordini, per margine, per retention? Nessun parser può risolvere questa ambiguità senza tornare all'utente.

Il campo ambiguities dell'output serve esattamente a questo. Quando il parser non riesce a determinare un aspetto dell'intent con sufficiente certezza, invece di scegliere silenziosamente, lo marca come ambiguo con le possibili interpretazioni. Il sistema può quindi decidere di chiedere chiarimento prima di procedere.

La soglia di confidence sotto cui richiedere chiarimento è una delle decisioni di design più importanti dell'intera architettura. Una soglia troppo alta genera troppi blocchi e degrada l'esperienza. Una soglia troppo bassa permette al sistema di procedere con interpretazioni sbagliate con piena fiducia.

---

## La posizione nella pipeline e il problema della ridondanza

L'Intent Parser è a monte di tutto il 2oo2. I due canali paralleli partono dopo di lui, entrambi ricevendo lo stesso intent strutturato come input. Questo significa che un errore dell'Intent Parser viene propagato identicamente a entrambi i canali, che produrranno piani coerenti con l'intent sbagliato, che il comparatore vedrà concordare.

La replicazione dell'Intent Parser non risolve completamente questo problema. Due parser sulla stessa tipologia di input tendono a sbagliare nelle stesse direzioni, rendendo il guadagno della ridondanza marginale rispetto al costo. Esistono però alcune strategie che riducono il rischio.

Una è la validazione a posteriori: dopo che l'intent è stato prodotto, un componente separato verifica la coerenza tra l'intent strutturato e l'input originale in linguaggio naturale. È un task più semplice dell'estrazione completa e può essere implementato con un modello più piccolo e specializzato, riducendo la correlazione.

Un'altra è la propagazione dell'input originale come contesto immutabile lungo tutta la pipeline. In questo modo, ogni componente che prende decisioni semantiche può consultare sia l'intent strutturato sia il testo originale, e il comparatore in caso di disaccordo ha accesso a entrambe le rappresentazioni.

---

## Come si testa

Il testing dell'Intent Parser non può essere fatto solo con test unitari automatici. Richiede la costruzione di un golden dataset di esempi reali, ciascuno con l'intent atteso documentato da un esperto di dominio.

Il golden dataset deve coprire i casi nominali, gli input formulati in modo insolito ma semanticamente chiari, i casi genuinamente ambigui con la risposta attesa (che è: riconoscerli come ambigui), e i casi fuori dominio dove il parser deve segnalare che non riesce a classificare l'input.

La qualità del parser si misura sulla capacità di distinguere questi quattro tipi, non solo sui casi nominali. Un parser che non sa dire "non so" è più pericoloso di uno che fallisce esplicitamente.

---

## Pro dell'approccio

Separare la comprensione dell'intent dalla generazione del piano è uno dei principi più solidi di questa architettura. Significa che il modello che genera il piano lavora su una specifica precisa e non ambigua, non su linguaggio naturale grezzo. Questo riduce la variabilità dell'output e rende il comportamento dei due canali più confrontabile.

La rappresentazione strutturata dell'intent è anche il punto di ancoraggio per il validatore semantico e per il binding logico a valle: entrambi possono verificare la coerenza delle loro decisioni rispetto a una specifica formale, non rispetto all'interpretazione soggettiva di un testo.

---

## Contro, dubbi e punti aperti

Il fatto che l'Intent Parser sia un single point of failure senza ridondanza è il limite più serio dell'architettura. Mitigarlo senza rendere il sistema eccessivamente complesso o costoso è un problema aperto.

La tassonomia degli intent è un artefatto di dominio che richiede manutenzione attiva. Quando emergono nuovi tipi di intent non previsti dalla tassonomia, il parser li tratterà come ambigui o li classificherà erroneamente nella categoria più vicina. Questo tipo di degrado è lento e silenzioso, e viene rilevato solo se si monitorano i pattern di ambiguità nel tempo.

La calibrazione del valore di confidence è tutt'altro che banale, specialmente con parser basati su LLM. I modelli linguistici tendono a essere mal calibrati: esprimono alta confidence anche su interpretazioni sbagliate. Tecniche di post-calibrazione esistono ma aggiungono complessità al sistema.

La scelta della soglia di confidence per la richiesta di chiarimento interrompe il flusso e crea attrito nell'esperienza utente. In alcuni contesti questo è accettabile, in altri no. L'architettura assume che sia sempre possibile tornare all'utente per chiarimento, ma questa assunzione non regge in tutti i casi d'uso, specialmente quelli dove il sistema è completamente automatico senza interazione umana.
