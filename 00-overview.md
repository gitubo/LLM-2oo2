# Architettura di Inferenza Validata per Generazione di Piani d'Azione

## Il problema che questa architettura affronta

I Large Language Model sono strumenti potenti ma intrinsecamente probabilistici. Questa caratteristica, che li rende flessibili e capaci di ragionamento complesso, è anche la loro fragilità principale: producono output corretti con alta probabilità, ma non con certezza. In contesti dove l'output del modello viene usato per guidare azioni concrete su sistemi reali, quella probabilità residua di errore non è trascurabile.

Il problema si aggrava quando si considerano le caratteristiche degli errori dei LLM. Non si tratta di errori casuali e uniformemente distribuiti, come potrebbe essere il rumore in un segnale elettronico. Gli errori dei modelli linguistici sono sistematici e correlati: tendono a manifestarsi sulle stesse tipologie di input, a seguire gli stessi bias di training, a omettere le stesse categorie di informazione. Questo rende i meccanismi classici di ridondanza, pensati per errori indipendenti, meno efficaci di quanto ci si aspetterebbe in teoria.

Il caso specifico che questa architettura affronta è la generazione automatizzata di piani d'azione strutturati: sequenze ordinate di task atomici, rappresentate come grafi diretti aciclici serializzati in JSON, che vengono poi eseguiti da un sistema a valle. Il vincolo fondamentale è che un piano errato non è solo un output di scarsa qualità, è un'istruzione sbagliata che verrà eseguita.

---

## Lo spunto del 2oo2 e perché non lo seguiamo come riferimento normativo

L'architettura prende ispirazione dal pattern 2oo2 (two-out-of-two) utilizzato nei sistemi safety-critical industriali, come quelli dell'avionica, del controllo ferroviario e degli impianti nucleari. In quel contesto, due canali indipendenti elaborano lo stesso input e il sistema procede solo se entrambi concordano. Il disaccordo è trattato come un segnale di pericolo, indipendentemente da quale dei due canali abbia ragione.

Il principio è elegante: non è necessario sapere chi sbaglia, basta sapere che qualcosa non quadra per fermarsi.

Tuttavia il 2oo2 industriale nasce in un contesto radicalmente diverso. I sistemi hardware ridondanti hanno errori genuinamente indipendenti: la probabilità che due transistor si guastino producendo lo stesso output sbagliato è trascurabile. Due LLM, anche se diversi per architettura e provider, condividono bias di training derivanti dallo stesso corpus di dati su cui è stato addestrato l'intero settore. La correlazione degli errori non è mai zero, e su certi tipi di input può essere molto alta.

Questa architettura usa quindi il 2oo2 come punto di partenza concettuale, adattandolo in modo sostanziale per tenere conto delle specificità dei modelli linguistici: la correlazione degli errori, la difficoltà di definire "accordo" su output testuali, la necessità di normalizzare gli output prima del confronto, e la presenza di componenti deterministici che rompono la simmetria dei due canali.

---

## Cosa questa architettura si propone di risolvere

L'obiettivo primario è ridurre la probabilità che un piano d'azione errato raggiunga l'esecuzione, mantenendo al tempo stesso una disponibilità del sistema accettabile. Questi due obiettivi sono in tensione: aumentare i controlli riduce la probabilità di errori ma aumenta la probabilità di blocchi, ritardi e falsi allarmi.

L'architettura non si propone di eliminare gli errori. Si propone di renderli visibili, categorizzati e gestibili, distinguendo tra errori che possono essere corretti automaticamente, errori che richiedono intervento umano, ed errori che sono segnali di un problema sistemico nel design.

Gli obiettivi secondari, ugualmente importanti, sono l'osservabilità e il miglioramento continuo. Un sistema che valida silenziosamente senza emettere metriche non permette di capire se sta funzionando o se sta degradando lentamente nel tempo. Un sistema che non usa i propri dati di produzione per migliorarsi resta fermo al punto di partenza.

---

## I principi fondamentali su cui si basa

**Separazione delle responsabilità per componente.** Ogni componente nella pipeline ha una responsabilità precisa e non si occupa di problemi che competono ad altri. Il validatore sintattico non ragiona sulla semantica, l'ottimizzatore non valida, il comparatore non corregge. Questa separazione rende ogni componente testabile in isolamento e il comportamento del sistema prevedibile.

**Validazione progressiva.** Gli output dei modelli vengono validati in stadi successivi, ognuno dei quali assume che gli stadi precedenti abbiano già svolto il loro lavoro. Questo permette di applicare controlli progressivamente più costosi solo su materiale che ha già superato i controlli precedenti, e di produrre messaggi di errore precisi che indicano esattamente dove e perché qualcosa ha fallito.

**Normalizzazione prima del confronto.** Il confronto tra i due canali avviene su piani canonici, non su output grezzi. Due piani semanticamente equivalenti ma espressi in forma diversa vengono ricondotti alla stessa rappresentazione prima del confronto, riducendo i falsi disaccordi che degraderebbero inutilmente la disponibilità del sistema.

**Separazione tra decisioni semantiche e decisioni infrastrutturali.** Le scelte su cosa fare (quali task, in quale ordine, con quali vincoli) vengono prese nella parte di pianificazione della pipeline. Le scelte su come farlo fisicamente (quale endpoint, quale connessione, quale istanza) vengono prese dopo la validazione, quando il piano è già stato approvato. Questo evita che condizioni runtime transitorie inquinino la logica di validazione.

**Osservabilità come proprietà di design, non come aggiunta successiva.** Ogni componente emette eventi strutturati con un trace_id comune che permette di ricostruire il percorso di un singolo input attraverso l'intera pipeline. Le metriche aggregate permettono di rilevare degrado prima che diventi un problema visibile agli utenti.

---

## Il flusso ad altissimo livello

Un input in linguaggio naturale entra nel sistema e viene processato in sequenza attraverso i seguenti macro-stadi:

**Comprensione dell'intent.** L'input viene trasformato da linguaggio naturale in una rappresentazione strutturata dell'obiettivo dell'utente. Questa rappresentazione non è un piano, è una specifica: descrive cosa l'utente vuole ottenere, con quali vincoli, con quale grado di certezza. Se l'input è genuinamente ambiguo, il sistema può chiedere chiarimento invece di procedere con un'interpretazione arbitraria.

**Costruzione del contesto per il Planner.** Il Capability Registry viene letto per estrarre le risorse rilevanti per la richiesta. Da esse viene costruito il system prompt del Planner: schema dei campi, azioni disponibili, operatori ammessi per ogni campo. Il Planner non riceve il registry completo ma solo le informazioni necessarie per la richiesta specifica. Questo vincola lo spazio delle scelte del modello prima ancora che la validazione intervenga.

**Generazione parallela del piano.** Due modelli linguistici distinti, selezionati per minimizzare la correlazione degli errori, generano indipendentemente un piano d'azione a partire dallo stesso intent strutturato e dallo stesso contesto. Il piano prodotto è esplicito e non ottimizzato: ogni nodo trasversale è separato, nessuna ottimizzazione viene fatta dal Planner.

**Pipeline di validazione per canale.** Ogni piano generato attraversa una pipeline di validazione che procede per stadi: una pre-validazione sintattica rapida che scarta output incomprensibili, un sanitizer che corregge piccole deviazioni recuperabili, una validazione sintattica completa tramite JSON Schema, una validazione semantica che verifica la coerenza del piano con l'intent, le regole di dominio, e lo schema delle risorse definito nel registry.

**Ottimizzazione.** Il piano validato viene portato in forma canonica. I nodi trasversali che l'API esterna supporta nativamente, secondo quanto dichiarato nel Capability Registry, vengono collassati nel nodo contestuale precedente. I nodi rimanenti vengono normalizzati per eliminare varianti equivalenti che produrrebbero falsi disaccordi al confronto.

**Binding logico.** Prima del confronto, ogni nodo contestuale viene arricchito con le informazioni di implementazione derivate dal registry: metodo HTTP, endpoint, convenzione di traduzione dei parametri. Nei casi con più implementazioni disponibili, la scelta viene guidata dai binding constraints dell'intent.

**Confronto tra i canali.** I due piani canonici vengono confrontati. Se concordano, si procede. Se divergono, il sistema categorizza il disaccordo e decide il comportamento conseguente.

**Binding fisico ed esecuzione.** Il piano concordato viene tradotto in istruzioni concrete eseguibili, risolvendo le risorse infrastrutturali disponibili in quel momento. Questa fase avviene dopo la validazione, in modo che condizioni runtime transitorie non influenzino le decisioni di pianificazione.

---

## Schema del flusso

```
Input utente (linguaggio naturale)
           |
           v
   [ Intent Parser ]
     Confidence < soglia?  -->  Clarification  -->  utente
           |
           v
   [ LLM Prompt Builder ]  <--  Capability Registry
           |
     (stesso contesto)
     |             |
     v             v
 [ LLM A ]     [ LLM B ]        <-- modelli diversi, parallel
     |             |
     v             v
[ Pre-Validator ] [ Pre-Validator ]
  fail?             fail?
  | retry            | retry
     |             |
     v             v
[ Sanitizer ]  [ Sanitizer ]    <-- correzioni recuperabili, loggato
     |             |
     v             v
[ Full Schema  [ Full Schema
  Validator ]   Validator ]     <-- JSON Schema completo
  fail?          fail?
  | escalate      | escalate
     |             |
     v             v
[ Semantic     [ Semantic
  Validator ]   Validator ]     <-- regole dominio + registry
  fail?          fail?
  | escalate      | escalate
     |             |
     v             v
[ Optimizer ]  [ Optimizer ]    <-- collasso nodi via registry
     |             |
     v             v
[ Logical      [ Logical
  Binding ]     Binding ]       <-- contratto HTTP da registry
     |             |
     v             v
     +------+------+
            |
            v
      [ Comparatore ]
      disaccordo?  -->  categorizza  -->  retry / escalate / human review
            |
            v
     Piano concordato
            |
            v
   [ Physical Binding ]         <-- risoluzione risorse runtime
     risorsa non disponibile?  -->  fallover loggato / errore esplicito
            |
            v
      [ Dispatcher ]
            |
            v
         Esecuzione
```

Il Capability Registry non è nella sequenza verticale perché non elabora il flusso: viene letto trasversalmente da LLM Prompt Builder, Semantic Validator, Optimizer, e Logical Binding. È una dipendenza di configurazione, non un componente di processing.

---

## Gestione degli errori

Gli errori nella pipeline sono trattati come eventi strutturati, non come eccezioni generiche. Ogni tipo di errore ha un comportamento definito:

La pre-validazione fallisce sul materiale irrecuperabile: se il modello non ha prodotto JSON valido con i campi fondamentali, il piano viene scartato e si richiede una nuova generazione. Non ha senso passare materiale incomprensibile ai componenti successivi.

Il sanitizer agisce sui problemi recuperabili senza interrompere il flusso, ma logga ogni correzione applicata. Un'alta frequenza di correzioni su un campo specifico è un segnale che il prompt del modello ha un problema sistematico, non un errore occasionale.

La validazione completa fallisce sul materiale che il sanitizer non ha potuto correggere. Questo fallimento porta a un'analisi strutturata: il piano era formalmente sbagliato in un modo che non era prevedibile dal design del sanitizer, e questo va investigato.

Il comparatore che rileva disaccordo tra i due canali non produce semplicemente un errore, produce una categorizzazione del disaccordo: strutturale, topologico, di binding logico. Questo permette di distinguere i casi dove il disaccordo è un segnale forte (i due modelli hanno interpretato l'intent in modo diverso) dai casi dove è rumore (piccole differenze di formulazione che l'ottimizzatore avrebbe dovuto normalizzare).

---

## Osservabilità

Ogni componente emette eventi strutturati che includono un trace_id comune, il nome del componente, il canale di appartenenza, l'esito, le correzioni o trasformazioni applicate, e la latenza. Questi eventi permettono di ricostruire il percorso di ogni singolo input attraverso la pipeline.

Le metriche aggregate di interesse sono il tasso di correzione del sanitizer per regola, il tasso di disaccordo del comparatore per categoria, la distribuzione della confidence dell'intent parser, la latenza per stadio ai diversi percentili, e il tasso di escalation per motivo.

Un sistema di alerting differenzia tra alert operativi, che segnalano un problema immediato, e alert di qualità, che segnalano un degrado lento che richiede attenzione ma non intervento d'emergenza.

---

## Feedback loop

I dati prodotti dalla pipeline in produzione alimentano un processo di miglioramento continuo che opera su tre ritmi temporali distinti.

In tempo reale, le soglie operative possono essere aggiustate automaticamente in risposta a condizioni anomale, come un picco nel tasso di escalation che suggerisce di abbassare temporaneamente la soglia di accettazione del comparatore.

Su base periodica, i log aggregati vengono analizzati per identificare pattern sistemici: correzioni ricorrenti del sanitizer, tipi di disaccordo frequenti, categorie di input con bassa confidence. Questi pattern guidano aggiornamenti ai prompt dei modelli, alle regole del sanitizer, e alle checklist del validatore semantico.

Su cicli lunghi, i cambiamenti alla tassonomia degli intent, allo schema JSON, e ai modelli LLM vengono gestiti con un processo formale che include test di regressione sul golden dataset.

Il golden dataset, composto dai casi revisionati da esperti di dominio nel corso del tempo, è il patrimonio principale del sistema. Permette di misurare l'effetto di ogni modifica e di rilevare degrado dovuto a drift della distribuzione degli input o a cambiamenti nei comportamenti dei modelli forniti da terze parti.

---

## I limiti che questa architettura non supera

L'architettura riduce significativamente la probabilità di errori sistemici e li rende visibili quando si verificano. Non elimina alcune categorie di problemi che è onesto riconoscere.

L'omissione concordata rimane il failure mode più insidioso: entrambi i modelli possono generare un piano plausibile e internamente coerente che omette lo stesso task critico, e il comparatore vederà accordo. La difesa, la checklist del validatore semantico, è efficace solo per le omissioni che si conosce a priori di dover controllare.

La latenza cumulativa della pipeline è dell'ordine di 1-2 secondi prima che l'esecuzione inizi. Per piani che agiscono su stato mutabile, questo intervallo introduce una finestra temporale in cui il sistema potrebbe cambiare tra la pianificazione e l'esecuzione.

La qualità dell'intera catena dipende in modo critico dalla qualità dell'intent parser, che è l'unico componente senza ridondanza. Un intent mal interpretato si propaga inalterato attraverso tutta la pipeline con la piena fiducia del sistema.

Questi limiti sono documentati non perché non abbiano soluzioni parziali, ma perché conoscerli è necessario per valutare correttamente quando questa architettura è adatta al problema in esame e quando non lo è.

---

## Aree aperte non documentate

Un'area che questa documentazione non affronta in modo strutturato riguarda la validazione semantica dei piani che contengono azioni con side effect, ovvero operazioni che modificano stato in modo non reversibile come delete o update.

Per questi casi esiste un approccio basato su un secondo passaggio LLM — un componente che traduce il piano in linguaggio naturale e un secondo componente che confronta questa descrizione con l'intent originale per verificare la corrispondenza semantica. Il pattern è concettualmente vicino a quello dell'LLM-as-Judge analizzato nelle fasi iniziali del design, e condivide con esso sia i vantaggi che i limiti: funziona quando i due LLM hanno bassa correlazione e quando il componente di traduzione è rigorosamente vincolato a non inferire l'intent, ma resta vulnerabile agli errori concordati e alla possibilità che la descrizione del piano sia artificialmente vicina all'intent originale.

La difesa alternativa più robusta per i side effect è deterministica: il Semantic Validator verifica che ogni piano con un nodo distruttivo abbia almeno un nodo filter esplicito e sufficientemente specifico a monte. Questa garanzia strutturale non valuta se il filter è quello semanticamente corretto, ma garantisce che l'operazione non operi sull'intera collection senza restrizioni.

La scelta tra le due strategie, o la loro combinazione, dipende dal grado di criticità delle operazioni con side effect nel dominio specifico e dalla disponibilità ad accettare i costi di latenza e complessità del secondo passaggio LLM.
