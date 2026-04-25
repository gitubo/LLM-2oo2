# Optimizer

## Scopo

L'Optimizer è l'ultimo componente della pipeline per-canale prima del Comparatore. Riceve un piano che ha già superato tutti i controlli di validità e lo trasforma in una forma canonica: una rappresentazione standard e non ambigua dove piani semanticamente equivalenti producono la stessa struttura JSON.

La ragione di esistenza dell'Optimizer è direttamente connessa al Comparatore: due piani equivalenti ma rappresentati diversamente verrebbero visti come piani diversi dal confronto. Questo produrrebbe falsi disaccordi che degraderebbero la disponibilità del sistema senza corrispondere a errori reali.

L'Optimizer risolve questo problema prima che raggiunga il Comparatore, applicando regole di riscrittura che collassano le varianti equivalenti in una sola forma.

---

## La canonicizzazione come obiettivo

L'obiettivo formale dell'Optimizer è creare una funzione che mappa ogni classe di equivalenza di piani in un unico rappresentante. Due piani che producono lo stesso risultato quando eseguiti devono produrre lo stesso output dall'Optimizer.

Questo è un obiettivo di approssimazione, non di completezza: non è necessario (né possibile in generale) catturare tutte le equivalenze possibili. È sufficiente catturare le equivalenze che si manifestano frequentemente nei piani generati dai modelli, riducendo così il tasso di falsi disaccordi a un livello accettabile.

---

## Le regole di riscrittura

Le regole di riscrittura dell'Optimizer sono trasformazioni locali del grafo che preservano la semantica. Ciascuna regola ha una precondizione esplicita che ne delimita l'applicabilità.

La commutatività dei filter è l'esempio canonico: due nodi filter sequenziali senza dipendenze tra loro producono lo stesso risultato indipendentemente dall'ordine. L'Optimizer può ordinarli in modo canonico, ad esempio alfabeticamente per campo filtrato. La precondizione è che i due filter siano genuinamente indipendenti: se il secondo filter usa un campo calcolato dal primo, la commutatività non vale e la regola non si applica.

Il collasso di filter sequenziali: due nodi filter sequenziali possono essere collassati in un singolo nodo filter con condizioni combinate tramite AND. La precondizione è che i filter siano indipendenti e che il motore di esecuzione a valle supporti condizioni composte. Se questa assunzione non vale, la regola non deve essere inclusa.

La normalizzazione degli identificatori dei nodi: se i nodi del piano vengono nominati con identificatori che variano tra generazioni diverse (nomi autogenerati, UUID, ecc.), l'Optimizer può rinominarli in modo canonico garantendo di aggiornare coerentemente tutte le referenze nelle dipendenze.

La normalizzazione dell'ordinamento delle dipendenze: se le liste di dipendenze di ciascun nodo non hanno un ordine semantico definito, l'Optimizer le ordina in modo canonico (ad esempio, per identificatore del nodo dipendente).

---

## Il principio di soundness

La proprietà più importante dell'Optimizer è la soundness: non può mai produrre un piano che si comporta diversamente dall'originale quando eseguito. Può essere incompleto (non trova tutte le equivalenze), ma non può essere scorretto (non modifica la semantica).

Questo principio richiede che ogni regola di riscrittura sia accompagnata da una dimostrazione (anche informale ma esplicita) che la trasformazione preserva la semantica, e da test che verificano questa proprietà su casi concreti.

La tentazione di aggiungere regole di ottimizzazione "ovvie" senza verificare rigorosamente le precondizioni è il rischio principale di questo componente.

---

## Il rischio di ottimizzazioni lossy

Un'ottimizzazione lossy è una trasformazione che cambia il comportamento del piano in modo non rilevato dai validatori. Questo è il failure mode più insidioso dell'Optimizer perché opera dopo tutti i controlli di validità.

Il caso tipico è una regola che assume la commutatività o l'idempotenza di un'operazione che non la possiede in tutte le circostanze. Un filter che esegue una query su un sistema esterno come side effect non è idempotente: collassarlo con un altro filter cambia il comportamento del piano in modo non visibile dalla struttura.

La difesa principale è la documentazione esplicita delle precondizioni di ogni regola e il testing su casi limite. Non è sufficiente che la regola sia corretta nel caso comune: deve essere verificata anche nei casi al confine.

---

## Esempio pratico

Un piano prodotto dal canale A ha questi nodi in sequenza: fetch, filter per status, filter per importo, sort, limit. Il canale B ha prodotto: fetch, filter per importo, filter per status, sort, limit. Entrambi sono semanticamente equivalenti: i due filter operano su campi diversi e possono essere applicati in qualsiasi ordine.

Senza Optimizer, il Comparatore vedrebbe una differenza topologica tra i due piani e segnalerebbe un disaccordo. Con l'Optimizer, entrambi i piani vengono trasformati in: fetch, filter per importo AND status (collasso in un nodo), sort, limit. Il Comparatore vede due piani identici e segnala accordo.

Questo è il caso ideale. Un caso più complesso emerge quando i due filter non sono completamente indipendenti, ad esempio se il filtro per importo deve essere applicato dopo che il filtro per status ha già ristretto l'insieme per ragioni di efficienza o per vincoli del motore di esecuzione. In questo caso la regola di collasso non si applica e l'ordine dei filter deve essere preservato.

---

## La relazione con il Comparatore

L'Optimizer e il Comparatore sono progettati in tandem. Il Comparatore definisce il livello di dettaglio a cui il confronto avviene; l'Optimizer garantisce che differenze irrilevanti a quel livello siano state eliminate prima del confronto.

Se il Comparatore confronta al livello topologico (stesso grafo di dipendenze, stesso tipo di nodi, stessa struttura), l'Optimizer deve garantire che varianti equivalenti al livello topologico producano la stessa struttura topologica.

---

## Pro dell'approccio

L'Optimizer migliora la disponibilità del sistema riducendo i falsi disaccordi al Comparatore. Senza di lui, variazioni banali di stile nella generazione del piano (ordine dei filter, nomi degli identificatori) produrrebbero disaccordi che non corrispondono a differenze semantiche reali.

L'Optimizer è anche l'unico componente che può semplificare il piano, riducendo il numero di nodi dove ci sono nodi equivalenti. Un piano più semplice è più facile da eseguire, da debuggare, e da comprendere.

---

## Contro, dubbi e punti aperti

L'Optimizer è il componente con il maggiore rischio di introdurre errori silenziosamente. Opera dopo tutti i validator, quindi un'ottimizzazione lossy non viene rilevata da nessun controllo successivo nella pipeline. La responsabilità di garantire la correttezza delle regole di riscrittura è interamente su chi progetta e mantiene il componente.

Il fatto che l'Optimizer sia identico su entrambi i canali significa che un suo bug si manifesta ugualmente su entrambi, producendo accordo al Comparatore anche quando entrambi i piani sono stati trasformati in modo sbagliato.

Un punto aperto riguarda la completezza della canonicizzazione. Quante equivalenze esistono nel dominio specifico? È difficile rispondere a questa domanda a priori. In pratica, le equivalenze si scoprono osservando i falsi disaccordi al Comparatore nel tempo: ogni disaccordo che viene classificato come "variante equivalente" è un candidato per una nuova regola dell'Optimizer.

Questo suggerisce un processo iterativo: partire con poche regole ben verificate, osservare i falsi disaccordi in produzione, aggiungere regole progressivamente. È preferibile a partire con molte regole e scoprire solo in produzione che alcune erano sbagliate.
