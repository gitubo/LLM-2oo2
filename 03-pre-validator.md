# Pre-Validator (Validazione Sintattica di Primo Livello)

## Scopo

Il Pre-Validator è il primo componente che incontra l'output grezzo del Plan Generator. Il suo compito è esclusivamente quello di determinare se il materiale in arrivo è sufficientemente strutturato da poter essere processato dai componenti successivi, in particolare dal Sanitizer.

Non valuta la correttezza del piano. Non controlla i valori, non verifica i tipi, non applica regole di dominio. Risponde a una sola domanda: c'è abbastanza struttura qui da poter lavorarci sopra?

Il principio è il fail fast: è meglio scartare subito un output irrecuperabile che passarlo al Sanitizer, dove fallirebbe comunque dopo aver consumato risorse e prodotto messaggi di errore meno informativi.

---

## Cosa controlla

Il Pre-Validator applica un insieme minimo di verifiche che rispecchiano i requisiti strutturali fondamentali del formato atteso.

Prima verifica che l'output sia JSON valido e parseable. Se il modello ha prodotto testo libero, markdown con blocchi di codice, JSON parziale o troncato, il pre-validator lo rileva immediatamente e scarta il materiale.

Poi verifica la presenza dei macro-campi obbligatori definiti a priori: tipicamente un campo metadata che contiene informazioni sul piano stesso, e un campo plan che contiene la lista dei task. Non controlla il contenuto di questi campi, solo la loro esistenza.

Verifica che il campo plan sia un array e che contenga almeno un elemento. Un piano vuoto non è un piano.

Questa è la lista completa dei controlli. Qualsiasi verifica aggiuntiva appartiene al Full Schema Validator.

---

## Comportamento in caso di fallimento

Se il Pre-Validator fallisce, il piano è considerato irrecuperabile e si avvia immediatamente un retry della generazione sullo stesso canale. Non viene passato nulla al Sanitizer.

Il retry avviene con lo stesso prompt, salvo che l'implementazione preveda un meccanismo di variazione del prompt per il retry (temperatura diversa, istruzioni aggiuntive sulla struttura attesa). La scelta dipende dalla causa più probabile del fallimento: se i fallimenti sono rari e casuali, il retry semplice è sufficiente; se sono sistematici, serve una strategia di prompt diversa.

Esiste un numero massimo di retry oltre il quale il canale viene considerato in failure e il sistema si comporta di conseguenza, che può significare procedere con un solo canale o escalare all'operatore.

---

## Esempio pratico

Un Plan Generator potrebbe produrre output come il seguente in caso di problema:

    Ecco il piano che hai richiesto:
    
    1. Recupera gli ordini
    2. Filtra per status confermato
    3. Ordina per data

Questo è testo libero, non JSON. Il Pre-Validator lo scarta immediatamente senza tentare di interpretarlo.

Un altro caso tipico è JSON parziale dovuto a un output troncato per limite di token:

    {"metadata": {"version": "1.0"}, "plan": [{"id": "t1", "task_type": "fetch"

Questo fallisce il parsing JSON. Il Pre-Validator lo scarta.

Un caso meno ovvio è JSON valido ma senza i campi attesi:

    {"result": "ok", "tasks": [...]}

Questo è JSON parseable ma i campi "metadata" e "plan" non ci sono. Il Pre-Validator lo scarta perché il Sanitizer non ha un punto d'appoggio su cui lavorare.

---

## Logging

Il Pre-Validator emette un evento strutturato per ogni esecuzione, indipendentemente dall'esito. L'evento include il trace_id, il canale, l'esito, e in caso di fallimento la categoria dell'errore (non JSON, campo mancante, plan vuoto, ecc.).

Questi log hanno un valore diagnostico importante. Un tasso di fallimento alto al Pre-Validator è un segnale che il modello sta sistematicamente producendo output malformati, il che può indicare un problema nel prompt, un aggiornamento del modello che ha cambiato il comportamento, o un tipo di input per cui il modello non è stato adeguatamente istruito.

---

## Pro dell'approccio

La semplicità è il vantaggio principale. Un componente con poche responsabilità è facile da testare, da capire, e da mantenere. Non ci sono casi grigi: o l'output supera i controlli o viene scartato.

La separazione dal Sanitizer è corretta dal punto di vista del design. Il Sanitizer lavora su materiale che ha già una struttura di base, e può quindi fare assunzioni ragionevoli su cosa correggere. Se il Sanitizer dovesse gestire anche output completamente malformati, diventerebbe più complesso e meno prevedibile.

La rapidità del fail fast riduce la latenza nei casi di errore: invece di passare materiale irrecuperabile attraverso tutti gli stadi della pipeline per vederlo fallire all'ultimo, il problema viene identificato al primo stadio.

---

## Contro, dubbi e punti aperti

Il Pre-Validator introduce una dipendenza sul formato dei macro-campi che deve essere mantenuta allineata con il resto del sistema. Se lo schema evolve e i nomi dei campi fondamentali cambiano, il Pre-Validator va aggiornato contestualmente. Questo non è un problema tecnico serio ma è un rischio di disallineamento in ambienti con deploy frequenti.

La soglia tra "irrecuperabile" e "recuperabile dal Sanitizer" non è sempre netta. Ci sono casi grigi dove il materiale ha qualche struttura ma non è abbastanza da rendere sicura la correzione automatica. La decisione di dove tracciare questa linea è soggettiva e dipende dalla tolleranza al rischio del caso d'uso specifico.

Un punto aperto riguarda la gestione del caso in cui entrambi i canali falliscano il Pre-Validator contemporaneamente. La probabilità è bassa se i modelli sono diversi, ma non è zero. Il sistema deve avere un comportamento definito per questo caso, che tipicamente è l'escalation immediata con un messaggio chiaro all'operatore.
