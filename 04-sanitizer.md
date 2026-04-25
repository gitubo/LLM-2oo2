# Sanitizer

## Scopo

Il Sanitizer è il componente che opera tra la pre-validazione e la validazione completa. Riceve materiale che ha superato il Pre-Validator (è JSON con i macro-campi presenti) ma non ha ancora affrontato il Full Schema Validator. Il suo compito è correggere automaticamente le deviazioni piccole e recuperabili dall'output grezzo del modello, portando il documento a uno stato dove la validazione completa ha alte probabilità di successo.

La metafora più accurata è quella di un normalizzatore: non cambia la sostanza del piano, sistema la forma. Aggiunge campi opzionali con valori di default documentati, normalizza tipi di dato (un numero espresso come stringa diventa un numero), rimuove campi extra non previsti dallo schema, standardizza formati come date e identificatori.

---

## La regola fondamentale del Sanitizer

Il Sanitizer deve correggere solo ciò che sa correggere con certezza e senza rischi di alterare la semantica. Ogni volta che esiste anche il minimo dubbio sul fatto che una correzione possa cambiare il significato del piano, il Sanitizer non corregge. Lascia passare il materiale così com'è e lascia che il Full Schema Validator fallisca con un messaggio preciso.

Questa regola esiste perché le correzioni silenziosamente sbagliate sono più pericolose degli errori espliciti. Un errore che fa fallire il Full Schema Validator è visibile e gestibile. Una correzione che introduce un valore di default semanticamente sbagliato è invisibile e si propaga.

---

## Cosa può correggere

Le categorie di correzione ammesse sono limitate e ben definite.

Campi obbligatori con default non ambiguo: se un campo che deve essere presente ha un valore di default che è corretto per quasi tutti i contesti del dominio, il Sanitizer può aggiungerlo. Il punto critico è "quasi tutti i contesti": se esiste anche solo una situazione ragionevole dove il default potrebbe essere sbagliato, quella correzione non appartiene al Sanitizer ma richiede una decisione esplicita.

Conversioni di tipo non perdenti: un valore numerico rappresentato come stringa può essere convertito nel tipo corretto senza ambiguità. Un booleano rappresentato come stringa "true" o "false" può essere convertito. Una data in formato non standard può essere normalizzata a ISO 8601 se il formato originale è non ambiguo.

Rimozione di campi non previsti dallo schema: se il modello ha aggiunto campi extra che non fanno parte dello schema definito, il Sanitizer li rimuove. Questo è sicuro perché campi non previsti sono per definizione ignorati dal sistema a valle.

Normalizzazione di identificatori: se gli ID dei task devono seguire un formato specifico e il modello ha prodotto ID validi ma in formato diverso, il Sanitizer può normalizzarli garantendo di aggiornare coerentemente tutti i riferimenti (le dipendenze che puntano a quegli ID).

---

## Cosa non può correggere

Il Sanitizer non può e non deve correggere errori semantici o strutturali. Se un nodo del piano manca di un campo che non ha un valore di default ragionevole, il Sanitizer deve lasciare il campo mancante e permettere al Full Schema Validator di segnalarlo. Se un campo ha un valore che non appartiene all'insieme dei valori ammessi, il Sanitizer non può scegliere arbitrariamente un valore alternativo.

In particolare, il Sanitizer non deve mai inferire l'intent dell'utente per prendere decisioni di correzione. Se la direzione di un ordinamento (ascendente o discendente) non è specificata, aggiungere un default è già problematico; aggiungere un default basato sull'interpretazione del tipo di query è fuori dal perimetro del Sanitizer.

---

## Logging delle correzioni

Ogni correzione applicata deve essere loggata con dettaglio sufficiente da essere auditabile. Il log di una correzione include il campo modificato, il valore originale (anche se era assente), il valore applicato, e la regola di sanitizzazione che ha determinato la scelta.

I log del Sanitizer sono uno strumento diagnostico fondamentale. Aggregati nel tempo, mostrano quali tipi di correzione vengono applicate più frequentemente. Un'alta frequenza di correzioni su un campo specifico non è normale: indica che il prompt del Plan Generator ha un problema sistematico su quel campo, e la soluzione non è rendere il Sanitizer più aggressivo ma migliorare il prompt a monte.

Il campo confidence associato a ciascuna correzione (alta se il default è quasi universalmente corretto, bassa se è una scelta più arbitraria) permette di prioritizzare quali pattern meritano attenzione.

---

## Esempio pratico

Il Plan Generator produce un piano con un nodo filter che ha tutti i campi richiesti eccetto il campo case_sensitive, che è opzionale ma se presente deve essere un booleano. Il modello ha prodotto il valore "false" come stringa invece di come booleano.

Il Sanitizer corregge la conversione di tipo: "false" stringa diventa false booleano. Log della correzione: campo task[2].case_sensitive, originale "false" (string), applicato false (boolean), regola type_coercion_boolean, confidence alta.

Stesso scenario ma il campo case_sensitive non è presente. Se il dominio definisce un default di false per questo campo, il Sanitizer lo aggiunge. Log della correzione: campo task[2].case_sensitive, originale assente, applicato false (boolean), regola default_case_sensitive, confidence alta se il default è praticamente sempre corretto nel dominio.

Ora un caso che il Sanitizer non deve correggere: il nodo filter manca del campo operator, che può essere "and" o "or". Non esiste un default non ambiguo: "and" è più comune ma "or" è semanticamente diverso. Il Sanitizer lascia il campo mancante. Il Full Schema Validator segnala l'errore. Questo è il comportamento corretto.

---

## Pro dell'approccio

Separare la correzione automatica dalla validazione completa è architetturalmente corretto perché attribuisce responsabilità diverse a componenti diversi. Il Full Schema Validator può essere binario (passa o non passa) senza preoccuparsi di cosa si potrebbe correggere. Il Sanitizer può essere focalizzato sul recupero senza preoccuparsi di cosa è formalmente valido.

Il Sanitizer riduce il numero di fallimenti al Full Schema Validator che non sono errori reali ma sono deviazioni banali di formato. Questo migliora la disponibilità del sistema e riduce il rumore nelle metriche di qualità.

---

## Contro, dubbi e punti aperti

Il rischio principale del Sanitizer è la creazione di una zona grigia di responsabilità. Quando un piano superato il Sanitizer e il Full Schema Validator poi fallisce in produzione, è difficile capire se l'errore fosse nel piano originale o nell'applicazione di un default sbagliato. Questo richiede che i log del Sanitizer vengano correlati con i fallimenti downstream.

Il fatto che il Sanitizer sia lo stesso su entrambi i canali lo rende un single point of failure condiviso. Un bug nel Sanitizer che converte silenziosamente un valore sbagliato in qualcosa di formalmente valido lo fa su entrambi i canali simultaneamente. Il comparatore vedrà accordo e il piano passerà. La difesa è mantenere il Sanitizer semplice e testarlo esaustivamente come si farebbe con codice critico.

Un punto aperto riguarda l'evoluzione dello schema nel tempo. Quando vengono aggiunti nuovi campi obbligatori, bisogna decidere se hanno default sanitizzabili e aggiornare il Sanitizer contestualmente. Questo richiede un processo di coordinamento tra i team che gestiscono lo schema e quelli che gestiscono il Sanitizer.
