# Feedback Loop

## Scopo e premessa

Il feedback loop è il meccanismo attraverso cui il sistema usa i dati prodotti in produzione per migliorare se stesso nel tempo. Senza di lui, il sistema sarebbe statico: le sue capacità rimarrebbero quelle del momento del deploy, indipendentemente da quante informazioni accumula sull'uso reale.

La premessa fondamentale è che un feedback loop mal progettato è più pericoloso di nessun feedback loop. Un sistema che impara dai propri errori senza verificare che stia imparando la cosa giusta amplifica i propri bias invece di correggerli. Progettare il feedback loop significa progettare non solo il meccanismo di apprendimento ma soprattutto i vincoli che impediscono l'amplificazione degli errori.

---

## I due tipi di feedback loop e perché non sono equivalenti

Un loop di miglioramento usa i dati di produzione per identificare dove il sistema funziona male e per guidare modifiche che migliorano quella situazione. Richiede dati affidabili, processi di review, e misure di effetto delle modifiche.

Un loop di degrado si verifica quando il sistema usa il proprio output come fonte di truth per migliorarsi. Se gli errori non sono etichettati correttamente, il sistema impara a produrre gli stessi errori con maggiore frequenza. Questo è un rischio concreto nei sistemi basati su LLM, dove il confine tra output corretto e output plausibile ma sbagliato non è sempre visibile automaticamente.

La distinzione operativa tra i due è il ruolo della verifica umana: il loop di miglioramento richiede che almeno una parte del segnale di feedback sia verificata da esseri umani con conoscenza di dominio. Il loop di degrado tipicamente bypassa questa verifica per ragioni di scalabilità, e paga il prezzo in qualità.

---

## Le sorgenti di segnale

Non tutte le sorgenti di segnale hanno lo stesso valore per il feedback loop. Ordinarle per affidabilità è importante per capire dove investire.

Il ground truth dell'esecuzione è il segnale più affidabile: il piano è stato eseguito e il risultato era corretto o sbagliato. Questo segnale è binario, non ambiguo, e corrisponde direttamente alla qualità che si vuole misurare. Il problema è che richiede di sapere se il risultato era corretto, il che non è sempre automaticamente verificabile.

Il feedback esplicito di un operatore umano che ha visto l'output e può dire se era appropriato è altamente affidabile ma raro e non scalabile. È prezioso per costruire il golden dataset ma non può essere la sola fonte di segnale per il miglioramento continuo.

Il disaccordo del Comparatore è un segnale debole: segnala che c'era un problema ma non dice chi aveva ragione. Non va usato come segnale diretto di qualità ma come indicatore che un caso merita revisione umana.

Le correzioni del Sanitizer sono un segnale indiretto: non dicono se il piano finale era corretto, ma dicono che il modello ha prodotto un output che si discostava dallo schema atteso in un modo specifico. Questo è utile per identificare pattern sistematici nel comportamento del modello.

La confidence dell'Intent Parser è un segnale predittivo: non dice se l'output era sbagliato, dice quanto era probabile che fosse sbagliato. Utile per prioritizzare i casi da revisionare, non per alimentare direttamente il miglioramento.

---

## I tre ritmi del feedback loop

Il feedback loop non opera a una sola velocità. Componenti diversi del sistema evolvono su scale temporali diverse, e tentare di aggiornare tutto alla stessa velocità porta a instabilità o a lentezza.

Il ritmo real-time, nell'ordine dei secondi, è appropriato per le soglie operative: la soglia di agreement del Comparatore, il numero massimo di retry, i timeout. Questi parametri possono essere aggiustati automaticamente in risposta a condizioni anomale senza richiedere review umana, perché sono reversibili immediatamente e hanno effetti limitati.

Il ritmo periodico, nell'ordine dei giorni, è appropriato per i prompt dei modelli, le regole del Sanitizer, e la checklist del Semantic Validator. Questi aggiornamenti richiedono analisi dei pattern aggregati, una decisione umana, e test di regressione sul golden dataset prima del deploy. Non sono urgenti ma sono importanti.

Il ritmo lungo, nell'ordine delle settimane o dei mesi, è appropriato per cambiamenti alla tassonomia degli intent, allo schema JSON, ai modelli LLM, e alle regole dell'Optimizer. Questi cambiamenti hanno impatti a cascata su molti componenti e richiedono un processo formale di coordinamento, testing esteso, e deploy pianificato.

---

## Il golden dataset

Il golden dataset è il patrimonio più importante del sistema. È una raccolta di esempi verificati da esperti di dominio, ciascuno con l'input originale, l'intent strutturato atteso, e le proprietà del piano corretto. Cresce nel tempo con ogni caso revisionato dalla human review queue.

Ha tre usi distinti e ugualmente importanti. Come base per i test di regressione: ogni modifica al sistema viene verificata contro il golden dataset per garantire che non abbia peggiorato i casi noti. Come benchmark per confrontare versioni diverse dei modelli: quando un modello viene aggiornato, il golden dataset permette di misurare oggettivamente se il comportamento è migliorato o peggiorato. Come training signal pulito: se si vuole fare fine-tuning di un modello, il golden dataset è l'unica fonte di dati verificati disponibile.

La qualità del golden dataset è più importante della sua dimensione. Cento esempi accuratamente selezionati e verificati sono più utili di mille esempi generati automaticamente senza verifica.

---

## La human review queue

I casi che richiedono revisione umana vengono inseriti in una coda strutturata. Ogni caso nella coda include il trace_id, il motivo della revisione, l'input originale, i due piani prodotti dai canali, il diff del Comparatore, e una domanda specifica a cui il revisore deve rispondere.

La domanda specifica è importante: non si chiede al revisore di rileggere tutto dall'inizio e capire autonomamente cosa è successo. Gli si presenta il contesto già strutturato e una domanda precisa come "il piano A o il piano B è più coerente con questo intent?". Questo riduce il carico cognitivo e aumenta la consistenza delle risposte.

Ogni revisione produce tre output: un verdetto sul caso specifico che risolve l'escalation, un'etichetta per il golden dataset, e una nota opzionale su pattern osservati che potrebbe suggerire modifiche al sistema.

---

## La contaminazione del feedback

Il rischio principale del feedback loop è la contaminazione: usare come segnale di qualità dati che non sono stati verificati e che potrebbero essere sbagliati.

Il principio di separazione del segnale stabilisce quali sorgenti possono alimentare quali tipi di aggiornamento. Il segnale verificato da umano (ground truth di esecuzione, feedback esplicito, revisione nella human review queue) può alimentare aggiornamenti ai prompt, allo schema, alle regole. Il segnale automatico non verificato (disaccordo del Comparatore, confidence bassa dell'Intent Parser) può solo aprire un caso nella human review queue o aggiustare soglie operative, mai aggiornare direttamente la logica del sistema.

Questo principio riduce la scalabilità del feedback loop ma ne garantisce la correttezza. È un compromesso consapevole.

---

## Il degrado silenzioso e come rilevarlo

La forma più pericolosa di degrado del sistema non è il fallimento improvviso ma il peggioramento lento e graduale. Le metriche aggregate sembrano stabili perché i casi che degradano sono una minoranza, ma quella minoranza cresce nel tempo.

Le cause tipiche di degrado silenzioso sono la deriva della distribuzione degli input (gli utenti iniziano a usare il sistema in modi non previsti), gli aggiornamenti non annunciati dei modelli LLM da parte dei fornitori, e l'invecchiamento delle regole di dominio in risposta a cambiamenti del business.

Il canary system descritto nel documento di osservabilità è il meccanismo principale per rilevare questo tipo di degrado. Ma anche analizzare la distribuzione dei casi nella human review queue nel tempo è utile: se emergono nuove categorie di casi che prima non venivano escalati, questo segnala che qualcosa nel dominio è cambiato.

---

## Punti aperti

Il punto più difficile del feedback loop è ottenere ground truth affidabile a scala. Verificare manualmente l'esito di ogni piano eseguito è impossibile. Verificarne un campione statisticamente rappresentativo è necessario ma richiede risorse e processi. Automatizzare la verifica è possibile solo per categorie di operazioni dove esiste un oracolo (un sistema di riferimento che può dire se il risultato era corretto), e non sempre questo oracolo esiste.

La gestione del feedback loop in ambienti regolamentati dove ogni modifica al sistema richiede un processo formale di approvazione è un problema aperto. I ritmi naturali del feedback loop, soprattutto quello periodico, possono essere incompatibili con processi di approvazione che richiedono settimane. Trovare un equilibrio tra la velocità di miglioramento e i requisiti regolatori è una sfida specifica di certi domini che non ha una soluzione universale.

Un ultimo punto riguarda la gestione del feedback loop quando il sistema è nuovo e il golden dataset è piccolo. Nelle prime settimane di produzione, quasi tutti i dati sono nuovi e non verificati. Il feedback loop deve essere molto conservativo in questa fase, rischiando di perdere opportunità di miglioramento rapido. Trovare il giusto punto di equilibrio tra prudenza e velocità di apprendimento nelle fasi iniziali è una decisione che dipende fortemente dal dominio e dalla tolleranza al rischio.
