# Full Schema Validator (Validazione Sintattica Completa)

## Scopo

Il Full Schema Validator è il secondo livello di validazione sintattica e opera dopo il Sanitizer. A questo punto il documento ha già una struttura di base verificata dal Pre-Validator e ha subito le correzioni recuperabili del Sanitizer. Il Full Schema Validator applica la verifica formale completa contro lo schema JSON definito per il piano d'azione.

Il suo comportamento è binario: il documento rispetta lo schema oppure no. Non corregge, non inferisce, non applica default. Se qualcosa non va, produce un messaggio di errore preciso che indica esattamente quale campo, quale tipo atteso, quale valore trovato.

---

## Perché esiste una separazione tra Pre-Validator e Full Schema Validator

Applicare direttamente il Full Schema Validator sull'output grezzo del modello produrrebbe un'alta frequenza di fallimenti per deviazioni banali che il Sanitizer avrebbe potuto correggere. Questo abbassa la disponibilità del sistema inutilmente e rende i messaggi di errore meno informativi, perché non si può distinguere un errore strutturale reale da una stringa dove ci si aspettava un numero.

Applicando il Full Schema Validator dopo il Sanitizer, quasi tutti i fallimenti che raggiungono questo stadio sono errori reali, non deviazioni banali di formato. I fallimenti sono quindi più significativi e più informativi.

---

## Cosa verifica

Lo schema JSON formale copre la struttura completa del documento, inclusi tutti i campi, i loro tipi, i valori ammessi per i campi enumerati, i vincoli di formato, le relazioni di cardinalità (array con almeno N elementi, stringhe con un pattern specifico), e i campi obbligatori versus opzionali.

In particolare, per i nodi del piano, verifica che ogni nodo abbia un identificatore univoco, che il tipo di operazione appartenga al vocabolario definito, che i campi specifici di ciascun tipo di operazione siano presenti e corretti, e che le dipendenze dichiarate referenzino identificatori che esistono nel piano.

Quest'ultimo controllo, la verifica delle referenze nelle dipendenze, è tecnicamente fuori dal perimetro di un JSON Schema puro ma è essenziale per garantire che il grafo sia internamente consistente. Può essere implementato come un passaggio aggiuntivo dopo la validazione dello schema.

---

## Comportamento in caso di fallimento

Il fallimento del Full Schema Validator produce un evento strutturato che include il trace_id, il canale, la lista degli errori di validazione con path JSON precisi, e una categorizzazione dell'errore.

Questo evento non porta automaticamente a un retry del Plan Generator. Prima viene analizzato: il fallimento è un caso non previsto dallo schema che suggerisce di aggiornare il Sanitizer? È un errore che emerge sistematicamente su un tipo di input specifico, indicando un problema nel prompt? È un caso genuinamente raro che richiede escalation?

Il retry al Plan Generator è una possibilità ma non la risposta automatica. Un retry che usa lo stesso prompt sullo stesso input ha alta probabilità di produrre lo stesso errore.

---

## Il valore diagnostico dei fallimenti

I fallimenti al Full Schema Validator sono informazione preziosa, non semplici errori operativi. Nel tempo, i pattern di fallimento rivelano molto sulla qualità del sistema:

Se lo stesso campo fallisce frequentemente con lo stesso tipo di errore, il prompt del Plan Generator non sta istruendo adeguatamente il modello su quel campo, oppure il Sanitizer dovrebbe coprire quella correzione.

Se i fallimenti emergono su tipi specifici di operazione, quei task potrebbero essere particolarmente difficili da generare correttamente per il modello, il che suggerisce di aggiungere esempi specifici nel prompt o di rivedere la definizione di quel tipo di operazione.

Se i fallimenti sono rari e distribuiti su molti campi diversi, si tratta probabilmente di variabilità normale del modello che il Sanitizer non è in grado di coprire completamente. Questo può essere accettabile se il tasso rimane sotto una certa soglia.

---

## Relazione con lo schema di dominio

Lo schema JSON validato dal Full Schema Validator è l'artefatto formale che definisce il contratto tra il Plan Generator e il resto della pipeline. Tutti i componenti successivi, dal Semantic Validator all'Optimizer al Binding, possono fare assunzioni su ciò che hanno ricevuto basandosi su questo contratto.

La gestione delle versioni dello schema è un aspetto critico. Quando lo schema evolve, tutti i componenti che dipendono da esso devono essere aggiornati coordinatamente. Un meccanismo di versioning esplicito nel campo metadata del documento è utile per rilevare disallineamenti.

---

## Esempio pratico

Un piano con tre nodi arriva al Full Schema Validator dopo il Sanitizer. Il nodo di tipo sort ha il campo direction con valore "descending" invece del valore atteso "DESC". Il Sanitizer non ha corretto questo perché la regola di normalizzazione per i valori dell'enum direction non è stata inclusa (è un campo con valori ammessi specifici, e "descending" è comprensibile ma non valido).

Il Full Schema Validator fallisce con un messaggio preciso: task[2].direction deve essere uno di ["ASC", "DESC"], trovato "descending". Questo fallimento va analizzato: dovrebbe essere il Sanitizer a normalizzare questo pattern? È un caso abbastanza comune da giustificarlo?

---

## Pro dell'approccio

La verifica formale completa tramite JSON Schema è deterministico, testabile, e non dipende da componenti probabilistici. La sua correttezza può essere garantita con test unitari esaustivi.

Separare la validazione sintattica dalla validazione semantica (che arriva dopo) mantiene ciascun validatore focalizzato su una categoria di problemi. Il Full Schema Validator non sa nulla di semantica del dominio e non deve saperne.

---

## Contro, dubbi e punti aperti

Lo schema JSON deve essere mantenuto allineato con il vocabolario dei task e le aspettative dei componenti successivi. In ambienti con sviluppo attivo, questa sincronizzazione richiede disciplina e automazione.

I JSON Schema validator standard non supportano tutti i tipi di vincoli che si potrebbero volere, in particolare vincoli che richiedono di guardare più campi contemporaneamente (ad esempio, se task_type è "sort" allora il campo sort_field è obbligatorio). Questi vincoli possono essere implementati come validazioni aggiuntive fuori dallo schema puro, ma introducono complessità.

Un punto aperto riguarda la profondità della verifica delle referenze nelle dipendenze. Verificare che ogni dipendenza punti a un nodo esistente è relativamente semplice. Verificare che il grafo risultante sia aciclico è un passo aggiuntivo che tecnicamente appartiene al Semantic Validator ma potrebbe essere implementato qui per semplicità. La scelta del confine tra i due componenti non è obbligata.
