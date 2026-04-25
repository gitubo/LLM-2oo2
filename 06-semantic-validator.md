# Semantic Validator

## Scopo

Il Semantic Validator è il componente più ricco di conoscenza di dominio nell'intera pipeline. Opera dopo il Full Schema Validator, quindi su un documento che è già formalmente valido. Il suo compito è verificare che il piano sia corretto non solo nella forma ma nel significato: che le operazioni abbiano senso nel loro contesto, che le dipendenze riflettano le necessità semantiche del dominio, che non manchino task obbligatori, e che il piano nel suo insieme sia coerente con l'intent che lo ha originato.

La distinzione tra validazione sintattica e semantica è fondamentale: un piano può essere perfettamente valido secondo lo schema JSON e completamente sbagliato rispetto al problema che doveva risolvere.

---

## Il Capability Registry come fonte di verità per la validazione

Il Semantic Validator usa il Capability Registry come riferimento per una categoria specifica di controlli: la correttezza dei nodi trasversali rispetto alla risorsa su cui operano.

Quando il validator incontra un nodo filter, risale la linked list fino al primo nodo contestuale e legge dal registry lo schema della risorsa corrispondente. Verifica che il campo su cui il filter opera esista nello schema, che il tipo del valore usato sia compatibile con il tipo dichiarato, e che il valore rispetti eventuali enum. Lo stesso vale per i nodi sort, che devono operare su campi esistenti nello schema.

Questo tipo di controllo è strutturalmente diverso dagli invarianti assoluti e dalle regole dipendenti dall'intent: non richiede conoscenza del dominio applicativo ma solo la capacità di confrontare il piano con la definizione formale della risorsa. Il registry rende questo confronto possibile in modo deterministico.

---

## Le categorie di controllo

Il Semantic Validator esegue controlli che appartengono a due categorie con natura diversa, che è importante tenere distinte.

La prima categoria comprende gli invarianti assoluti: regole vere indipendentemente dall'intent, perché derivano dalla semantica delle operazioni stesse. Un Limit senza Sort produce risultati non deterministici, sempre. Un nodo di scrittura senza il nodo di acquisizione del lock che lo precede è sempre un problema. Un fetch che viene dopo una trasformazione dei dati che quella stessa trasformazione dovrebbe ricevere come input è sempre circolare. Queste regole possono essere codificate deterministicamente e sono stabili nel tempo.

La seconda categoria comprende le regole dipendenti dall'intent: regole vere solo in presenza di certi tipi di intent, perché derivano da ciò che l'utente vuole ottenere. Un Limit semantico (i 10 più recenti) richiede Sort; un Limit arbitrario (dammi 10 qualsiasi) non lo richiede. Queste regole richiedono accesso all'intent strutturato prodotto dall'Intent Parser per essere applicate correttamente.

---

## Gli invarianti assoluti

Questi controlli vengono applicati sempre, indipendentemente dal contenuto dell'intent. Sono regole di buon senso del dominio che non hanno eccezioni note.

La verifica dell'aciclicità del grafo delle dipendenze garantisce che non esistano dipendenze circolari. Questo può essere fatto con un algoritmo standard di rilevamento dei cicli.

La verifica della connettività garantisce che ogni nodo sia raggiungibile dal grafo e che il grafo abbia un punto di partenza e un punto di arrivo ben definiti. Nodi isolati o sottografi disconnessi sono errori strutturali.

Le precondizioni di operazione codificano le dipendenze semantiche obbligatorie tra tipi di task: fetch prima di transform, sort prima di limit semantico, lock prima di write. Queste regole vengono documentate come coppie (prerequisito, dipendente) con la condizione sotto cui si applicano.

La checklist dei task obbligatori per tipo di operazione è la difesa principale contro le omissioni concordate: per certe categorie di intent, certi task devono essere presenti nel piano. Se un piano di tipo fetch_ordered_subset non contiene un nodo sort, il Semantic Validator lo rileva.

---

## Le regole dipendenti dall'intent

Questi controlli ricevono come input sia il piano sia l'intent strutturato prodotto dall'Intent Parser. Questo è il motivo per cui l'intent strutturato viene propagato lungo tutta la pipeline come contesto immutabile: il Semantic Validator ne ha bisogno.

Un esempio concreto: il campo requires_realtime nell'intent strutturato, se vero, esclude la presenza di task di caching nel piano. Il Semantic Validator verifica questa coerenza. Se il piano include un task di caching e l'intent specifica dati in tempo reale, c'è un'inconsistenza.

Un altro esempio: il campo ordering.required nell'intent strutturato, se vero, rende obbligatoria la presenza di un nodo sort nel piano. Se ordering.required è false, il sort non è obbligatorio anche se il piano include un limit.

La lista completa di queste regole è un documento di dominio, non tecnico. Va costruita e mantenuta dagli esperti del dominio specifico in cui il sistema opera.

---

## La dipendenza dall'Intent Parser

Il Semantic Validator dipende dalla correttezza dell'intent strutturato per applicare le regole della seconda categoria. Se l'Intent Parser ha prodotto un intent sbagliato, il Semantic Validator applicherà le regole corrette per l'intent sbagliato, approvando o rifiutando il piano in modo non corrispondente alle intenzioni reali dell'utente.

Questo è uno dei motivi per cui la qualità dell'Intent Parser è così critica: i suoi errori si propagano non solo al Plan Generator ma anche al Semantic Validator, che è l'ultimo componente prima del Comparatore ad avere accesso alla conoscenza di dominio.

---

## Comportamento in caso di fallimento

Il fallimento del Semantic Validator è più serio di quello del Full Schema Validator. Significa che il piano era formalmente corretto ma semanticamente sbagliato. Questo tipo di errore non può essere corretto automaticamente: richiederebbe di capire perché il modello ha prodotto un piano semanticamente sbagliato, il che implica o un problema nel prompt o un caso non previsto dalla tassonomia.

Il fallimento viene loggato con dettaglio, categorizzato per tipo di regola violata, e tipicamente porta a escalation verso revisione umana piuttosto che a retry automatico. Un retry sullo stesso input con lo stesso prompt ha alta probabilità di produrre lo stesso errore semantico.

---

## Esempio pratico

Un piano arriva al Semantic Validator con questa sequenza: fetch di orders, filter per status, limit a 10. L'intent strutturato contiene ordering.required: true con campo created_at e direzione DESC.

Il Semantic Validator verifica le regole dipendenti dall'intent e trova la violazione: ordering.required è true ma il piano non contiene un nodo sort. Il piano viene rifiutato con una descrizione precisa: "il piano non soddisfa il requisito di ordinamento esplicitato nell'intent, un nodo sort con campo created_at e direzione DESC deve precedere il nodo limit".

Questo fallimento viene loggato e categorizzato come omissione di task obbligatorio per intent con ordering. L'analisi di questo pattern nel tempo può rivelare che il modello omette sistematicamente il sort quando il limit è esplicito nell'input, suggerendo di aggiungere un esempio specifico nel prompt.

---

## Pro dell'approccio

Il Semantic Validator è il componente che porta conoscenza di dominio nella pipeline in modo esplicito, testabile, e manutenibile. Le regole sono documentate, le eccezioni sono gestibili, il comportamento è prevedibile.

La separazione tra invarianti assoluti e regole dipendenti dall'intent rende il componente più facile da ragionare: gli invarianti sono stabili e non richiedono manutenzione frequente, le regole dipendenti evolvono con la tassonomia degli intent.

---

## Contro, dubbi e punti aperti

La qualità del Semantic Validator dipende completamente dalla completezza della checklist e delle regole di dominio. Le regole mancanti non producono errori visibili: producono piani validati che poi falliscono in esecuzione o producono risultati sbagliati. Scoprire le regole mancanti richiede osservazione del comportamento in produzione, il che significa che il sistema imparerà alcune di esse solo dopo aver sbagliato.

Il confine tra responsabilità del Semantic Validator e responsabilità dell'Optimizer non è sempre netto. Alcune trasformazioni (come collassare due filter sequenziali in uno con AND) potrebbero essere considerate ottimizzazioni o potrebbero essere considerate correzioni di un piano mal strutturato. La scelta di dove mettere una certa logica ha implicazioni sull'ordine di esecuzione e sulla visibilità degli errori.

Un punto aperto rilevante riguarda la gestione delle regole di dominio che cambiano nel tempo. Se una precondizione obbligatoria viene rimossa o aggiunta, i piani generati prima della modifica potrebbero non essere più validi. Serve un meccanismo di versioning delle regole che permetta di gestire questa transizione senza invalidiamo piani correttamente generati sotto le regole precedenti.
