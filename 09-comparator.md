# Comparatore

## Scopo

Il Comparatore è il punto di convergenza dell'architettura 2oo2. Riceve i due piani canonici arricchiti con binding logico prodotti dai due canali paralleli e determina se concordano. Se concordano, il sistema ha raggiunto sufficiente fiducia per procedere. Se divergono, il sistema deve prendere una decisione su come gestire il disaccordo.

La premessa fondamentale del Comparatore è che il disaccordo tra due canali indipendenti è un segnale informativo, non necessariamente un errore. Indica che il problema era abbastanza difficile o ambiguo da produrre risposte diverse in due elaborazioni indipendenti.

---

## Cosa misura il Comparatore

Il Comparatore non misura la correttezza. Misura la consistenza tra i due canali. Questa distinzione è importante e spesso fraintesa.

Un sistema con alta consistenza non è necessariamente un sistema corretto: due canali possono concordare su un piano sbagliato. Un sistema con bassa consistenza non è necessariamente un sistema che sbaglia: può solo avere due canali con stili di generazione molto diversi che producono rappresentazioni diverse di piani equivalenti (che l'Optimizer avrebbe dovuto normalizzare).

La consistenza è una proxy della correttezza, non una misura diretta. È una proxy utile perché, in presenza di canali sufficientemente indipendenti, alta consistenza è correlata positivamente con correttezza. Ma il legame non è diretto e va tenuto presente quando si interpretano le metriche.

---

## I livelli di confronto

Il confronto avviene su più livelli con severità crescente. La scelta del livello dipende dal dominio e dalla tolleranza al falso allarme.

Il confronto strutturale verifica che i due piani abbiano lo stesso numero di nodi e la stessa struttura macro. È il confronto più grossolano e meno soggetto a falsi disaccordi.

Il confronto topologico verifica che il grafo delle dipendenze sia identico: gli stessi nodi, nelle stesse relazioni, con lo stesso ordine relativo. Due piani con gli stessi task ma dipendenze diverse sono piani diversi anche se hanno la stessa struttura macro.

Il confronto dei parametri verifica che i parametri di ogni nodo corrispondente siano identici. Questo è il livello più sensibile ai falsi disaccordi dovuti a variazioni di stile nella generazione.

Il confronto del binding logico verifica che la categoria di implementazione scelta per ogni nodo sia identica. Un disaccordo qui può indicare che i binding constraints dei due piani erano diversi, il che è a sua volta un segnale di inconsistenza nella generazione.

---

## La categorizzazione del disaccordo

Quando il Comparatore rileva un disaccordo, non si limita a segnalarlo: lo categorizza. La categorizzazione è essenziale per decidere come gestirlo e per analizzare i pattern nel tempo.

Un disaccordo strutturale (numero di nodi diverso) è tipicamente il più serio: i due modelli hanno prodotto piani con complessità diversa, il che suggerisce interpretazioni sostanzialmente diverse dell'intent.

Un disaccordo topologico (stessi nodi ma dipendenze diverse) può indicare un'omissione in uno dei due piani, oppure può essere un caso borderline dove entrambi gli ordini sono validi ma l'Optimizer non li ha normalizzati. Distinguere questi casi richiede analisi.

Un disaccordo sui parametri può essere rumore (piccole differenze di formulazione) o può essere sostanziale (un filtro con operatore diverso, un limite numerico diverso). L'analisi del diff specifico è necessaria.

Un disaccordo sul binding logico è spesso un segnale di inconsistenza nei binding constraints prodotti dai due modelli, che merita investigazione specifica.

---

## Il comportamento in caso di disaccordo

Il Comparatore non prende decisioni autonome sul da farsi quando rileva disaccordo. Produce un evento strutturato con il tipo di disaccordo, il diff dettagliato, e la categorizzazione, e il sistema di orchestrazione decide il comportamento conseguente basandosi su regole configurate.

I comportamenti tipici in caso di disaccordo sono: retry di entrambi i canali (se il disaccordo potrebbe essere dovuto a variabilità casuale), retry del solo canale con più correzioni del Sanitizer (potenzialmente quello meno affidabile), escalazione verso revisione umana, o nel caso di piani strutturalmente diversi, rifiuto dell'intera operazione.

Non esiste una risposta corretta universale: la politica di gestione del disaccordo dipende dal dominio, dalla criticità dell'operazione, e dalla disponibilità di un operatore umano.

---

## Il falso allarme e la disponibilità del sistema

Il falso allarme è il caso in cui il Comparatore rileva un disaccordo tra piani semanticamente equivalenti che l'Optimizer non ha normalizzato. Questo produce un blocco inutile che degrada la disponibilità del sistema senza corrispondere a un problema reale.

Il tasso di falso allarme è il principale determinante della disponibilità del sistema: se il Comparatore genera disaccordo frequentemente su piani che sarebbero stati eseguiti correttamente, il sistema è inutilizzabile in produzione anche se tecnicamente funziona.

Ridurre il tasso di falso allarme è responsabilità dell'Optimizer, che deve garantire che varianti equivalenti producano lo stesso piano canonico. Ma l'Optimizer ha limiti: non può catturare tutte le possibili equivalenze, e alcune differenze di stile nella generazione non sono normalizzabili senza rischiare trasformazioni lossy.

Il monitoraggio del tasso di falso allarme è quindi critico per la salute del sistema. Un aumento del tasso suggerisce che i modelli hanno cambiato stile di generazione (aggiornamento di versione) o che sono emersi nuovi pattern di input non coperti dall'Optimizer.

---

## Esempio pratico

Canale A produce: [fetch(orders), filter(status=confirmed), filter(amount>100), sort(created_at DESC), limit(10)]. Canale B produce: [fetch(orders), filter(amount>100 AND status=confirmed), sort(created_at DESC), limit(10)].

L'Optimizer del canale A ha normalizzato i due filter ma non li ha collassati perché la regola di collasso non era presente. L'Optimizer del canale B ha ricevuto un piano già con i filter collassati dal modello.

Il Comparatore vede un disaccordo strutturale: il piano A ha 5 nodi, il piano B ne ha 4. Categorizza il disaccordo come strutturale con nota "differenza nel numero di nodi filter". L'analisi rivela che si tratta di varianti equivalenti e il caso viene usato per aggiungere la regola di collasso all'Optimizer.

---

## Pro dell'approccio

Il Comparatore trasforma il disaccordo in informazione strutturata. Non è solo un gate che blocca, è uno strumento diagnostico che nel tempo rivela dove il sistema ha maggiore variabilità e dove le regole di normalizzazione sono incomplete.

Il fatto che il confronto avvenga su piani canonici già validati significa che il Comparatore può fare confronti semplici e deterministici, senza dover ragionare su strutture eterogenee o applicare giudizi soggettivi.

---

## Contro, dubbi e punti aperti

Il Comparatore misura consistenza, non correttezza. Questo limite strutturale non ha soluzione nell'ambito del 2oo2: è intrinseco al pattern. La difesa è il Semantic Validator a monte, che cattura le inconsistenze semantiche prima che raggiungano il Comparatore.

La politica di gestione del disaccordo è parametrizzabile ma non universale. Definire la risposta corretta a ogni tipo di disaccordo richiede conoscenza del dominio e dell'uso che viene fatto del sistema, e cambia nel tempo man mano che si accumula esperienza.

Un punto aperto riguarda i casi dove uno dei due canali non raggiunge il Comparatore perché bloccato da un fallimento nella propria pipeline. In quel momento il sistema ha un solo piano validato. Procedere con un solo piano o bloccare completamente? La risposta dipende dal grado di fiducia che si ha nel singolo canale e dalla criticità dell'operazione, ed è una decisione che va presa esplicitamente nel design del sistema.
