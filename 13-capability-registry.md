# Capability Registry

## Scopo

Il Capability Registry è la sorgente di verità dell'intero sistema. È un documento dichiarativo — nessuna logica, nessuna funzione — che descrive le risorse del dominio, i loro campi, le azioni disponibili su di esse, e il contratto con le API esterne che le espongono.

La sua posizione nell'architettura è trasversale: non è un componente della pipeline sequenziale ma un artefatto di configurazione che viene letto da più componenti in momenti diversi. Il LLM Prompt Builder lo legge per costruire il contesto del Planner. Il Semantic Validator lo legge per verificare la correttezza dei campi e degli operatori. L'Optimizer lo legge per decidere quali nodi trasversali collassare. Il Logical Binding lo legge per costruire il contratto HTTP.

Questa centralità è il suo valore principale: esiste un solo posto dove modificare la definizione di una risorsa, e quella modifica si propaga automaticamente a tutti i componenti che ne dipendono.

---

## La distinzione tra nodi contestuali e trasversali

Prima di descrivere la struttura del registry, è necessario comprendere la distinzione fondamentale tra i due tipi di nodi che compongono un piano.

I nodi contestuali appartengono a una risorsa specifica e si traducono in una chiamata HTTP a un'API esterna. Fetch, delete, update sono nodi contestuali. Conoscono l'entità su cui operano, il metodo HTTP, l'endpoint.

I nodi trasversali operano su qualsiasi collection indipendentemente dalla risorsa. Filter, sort, limit, map sono nodi trasversali. Non fanno chiamate HTTP: vengono eseguiti lato client oppure, quando l'API esterna li supporta nativamente, vengono collassati nel nodo contestuale precedente durante la fase di ottimizzazione.

Il registry è la fonte che definisce questa distinzione per ogni risorsa: non in astratto, ma concretamente per ogni campo e ogni operazione.

---

## Struttura

Il registry è organizzato per risorsa. Ogni risorsa ha tre sezioni principali.

**Schema dei campi** — descrive i campi dell'oggetto di dominio: tipo, valori ammessi se enumerabili, formato se strutturato. Questa sezione serve al Semantic Validator per verificare che i nodi trasversali operino su campi che esistono davvero sulla risorsa, e all'LLM per sapere su quali campi può costruire filtri e ordinamenti.

**Azioni disponibili** — descrive le operazioni che si possono compiere sulla risorsa. Per ogni azione: una descrizione leggibile che va nel prompt, il flag side_effect che attiva meccanismi di validazione aggiuntivi, i parametri diretti accettati dall'azione, e il contratto HTTP.

**Supporto nativo per le operazioni trasversali** — per ogni azione, quali operazioni trasversali l'API esterna è in grado di gestire nativamente, su quali campi, e con quali operatori. Questa è la sezione che l'Optimizer usa per decidere cosa collassare.

---

## Il formato

```json
{
  "registry_version": "1.0",
  "resources": {
    "users": {
      "description": "Utenti registrati della piattaforma",
      "schema": {
        "id":           { "type": "string" },
        "role":         { "type": "string", "enum": ["admin", "viewer", "editor"] },
        "active":       { "type": "boolean" },
        "created_at":   { "type": "string", "format": "date-time" },
        "last_login_at":{ "type": "string", "format": "date-time" }
      },
      "actions": {
        "fetch": {
          "description": "Recupera la lista degli utenti",
          "side_effect": false,
          "params": {
            "role":   { "type": "string", "enum": ["admin","viewer","editor"], "optional": true },
            "active": { "type": "boolean", "optional": true }
          },
          "http": {
            "method": "GET",
            "endpoint": "/users",
            "convention": "suffix"
          },
          "supports": {
            "filter": {
              "fields": {
                "role":   { "operators": ["eq", "neq"] },
                "active": { "operators": ["eq"] }
              }
            },
            "sort":  { "fields": ["email", "role", "created_at"] },
            "limit": false
          }
        },
        "delete": {
          "description": "Elimina gli utenti della collection corrente",
          "side_effect": true,
          "params": {},
          "http": {
            "method": "DELETE",
            "endpoint": "/users",
            "convention": "suffix"
          },
          "supports": {
            "filter": false,
            "sort":   false,
            "limit":  false
          }
        }
      }
    }
  },
  "conventions": {
    "suffix": "field__op=value",
    "json":   "filter={field:{$op:value}}",
    "colon":  "filter=field:op:value",
    "qs":     "where=field op value"
  }
}
```

---

## Le scelte di design

**registry_version a livello radice.** Il registry evolve insieme al dominio. I componenti che lo leggono vengono aggiornati su cicli temporali diversi. Il campo di versione permette a ogni componente di rilevare un disallineamento invece di comportarsi in modo imprevedibile su un formato che non conosce.

**Le conventions come sezione separata.** Il registry dichiara per ogni azione il nome della convenzione usata dall'API esterna, non la logica di traduzione. La logica vive in un layer separato con un adapter per convenzione. Questo significa che aggiungere una nuova risorsa non richiede mai di scrivere logica di traduzione, e che aggiungere il supporto a una nuova API esterna richiede di scrivere un solo adapter riutilizzabile da tutte le risorse che usano quella convenzione.

**side_effect come dichiarazione esplicita.** Non viene inferito dal metodo HTTP. Un DELETE potrebbe non avere side effect critici in certi contesti, un POST potrebbe averli. La semantica di side effect nel senso che interessa al sistema — ovvero operazioni che modificano stato in modo non reversibile — non è deducibile meccanicamente dal metodo HTTP e deve essere dichiarata consapevolmente.

**supports: false come valore esplicito.** Quando un'operazione trasversale non è supportata nativamente, il valore è false, non un campo assente. La distinzione è importante per l'Optimizer: un campo assente potrebbe indicare un registry incompleto o una versione più vecchia dello schema; false è una dichiarazione consapevole che l'operazione rimane un nodo client.

---

## Come viene usato dai componenti a valle

Il LLM Prompt Builder legge il registry per costruire il contesto del Planner: estrae la descrizione della risorsa, i campi dello schema, le azioni disponibili con i loro parametri, e i campi filtrabili con i rispettivi operatori ammessi. Passa all'LLM solo le risorse rilevanti per la richiesta corrente, non il registry completo.

Il Semantic Validator legge lo schema per verificare che ogni nodo filter e sort nel piano operi su campi che esistono sulla risorsa, e che gli operatori usati siano tra quelli dichiarati come supportati.

L'Optimizer legge la sezione supports di ogni azione contestuale per decidere quali nodi trasversali successivi possono essere collassati in essa. Se filter su un certo campo è supportato nativo mente, il nodo filter viene rimosso dalla linked list e i suoi parametri vengono aggiunti ai parametri del nodo contestuale.

Il Logical Binding legge la sezione http per costruire il contratto della chiamata: metodo, endpoint, e nome della convenzione da applicare per tradurre i parametri.

---

## Pro dell'approccio

La sorgente di verità unica elimina la possibilità di disallineamenti tra componenti. Quando un'API esterna aggiunge il supporto nativo al filter su un nuovo campo, si aggiorna il registry in un posto solo e l'Optimizer inizia automaticamente a collassare quel nodo senza modifiche al codice.

Il formato dichiarativo rende il registry leggibile e modificabile da chi conosce il dominio senza necessità di competenze tecniche sulla pipeline. È un documento di configurazione, non di programmazione.

La separazione tra schema dei campi e azioni disponibili riflette una distinzione semantica reale: lo schema descrive l'oggetto di dominio, le azioni descrivono cosa si può fare su di esso. Questa separazione permette al Semantic Validator di fare type checking sui nodi trasversali risalendo la linked list fino al primo nodo contestuale e usando lo schema della risorsa corrispondente.

---

## Contro, dubbi e punti aperti

La distinzione tra campi filtrabili come parametri diretti dell'azione e campi filtrabili come operazioni trasversali collassabili non è completamente risolta nel formato attuale. Nel formato proposto, un campo può apparire in params (parametro diretto dell'azione fetch) e in supports.filter.fields (filtraggio nativo collassabile). Sono due meccanismi di filtraggio con implementazione diversa sull'API esterna, e la documentazione del registry deve rendere questa distinzione esplicita per evitare ambiguità nell'Optimizer.

Il registry è un single point of failure di conoscenza: se una risorsa è definita in modo sbagliato — un operatore mancante, un campo con il tipo errato, un endpoint sbagliato — l'errore si propaga a tutti i componenti che la usano. La qualità del registry non può essere verificata automaticamente in modo completo: richiede revisione da esperti di dominio e testing di integrazione con le API esterne reali.

La gestione delle versioni del registry in ambienti con deploy frequenti è un problema pratico non banale. Un aggiornamento del registry che aggiunge un nuovo campo obbligatorio potrebbe invalidare piani generati e cached con la versione precedente. Serve una strategia di compatibilità che non è definita nel formato attuale.

Un punto aperto riguarda le risorse con azioni che operano su sottorisorse o su relazioni tra risorse. Il formato attuale modella bene il caso semplice di una risorsa piatta con azioni dirette, ma non copre casi come fetch(orders).filter(user.role=admin) dove il filter attraversa una relazione. Questo limite va riconosciuto esplicitamente nella documentazione per evitare che venga scoperto solo in produzione.
