# Concetti architetturali — Frontend — Livello 1: Panoramica dei pattern

> Documento tecnologia-agnostico. Per il confronto approfondito vedi [frontend-02-confronto](frontend-02-confronto.md), per l'ancoraggio al codice reale [frontend-03-nel-progetto](frontend-03-nel-progetto.md).

| Pattern / concetto | Definizione in una frase |
|---|---|
| **Reattività fine-grained** | Il meccanismo fondante di Vue: lo stato è avvolto in contenitori osservabili (`ref`, `reactive`) e il framework traccia automaticamente chi li legge (template, `computed`, `watch`), ri-eseguendo solo quei punti quando il valore cambia — un'applicazione dell'Observer pattern. |
| **Options API** | Stile "storico" di Vue in cui un componente è un oggetto con proprietà fisse (`data`, `computed`, `methods`, `watch`), ognuna raggruppata per *tipo*. |
| **Composition API** | Stile introdotto in Vue 3 in cui la logica di un componente è scritta come funzioni JavaScript/TypeScript (`ref`, `computed`, funzioni) che puoi raggruppare liberamente per *funzionalità* invece che per tipo. |
| **`<script setup>`** | Sintassi zucchero-sintattico sopra la Composition API: tutto ciò che dichiari nel blocco è automaticamente esposto al template, senza un `return` esplicito. |
| **Composable** | Funzione riusabile che incapsula logica stateful con la Composition API (es. `usePermission()`), l'equivalente Vue dei React Hooks personalizzati. |
| **Store pattern (state management centralizzato)** | Stato condiviso tra più componenti, gestito fuori dai singoli componenti in un oggetto centrale con regole su come può essere letto/modificato. |
| **MVVM (Model-View-ViewModel)** | Pattern architetturale dove la View (UI) si lega reattivamente a un ViewModel (stato + logica di presentazione), che a sua volta orchestra il Model (dati/business logic) — le SPA reattive (Vue, React+hooks) sono di fatto varianti di MVVM. |
| **Service layer (API layer)** | Modulo dedicato esclusivamente a effettuare chiamate HTTP verso il backend, isolato da componenti e store. |
| **Optimistic update** | Aggiornare l'interfaccia (o lo stato locale) *prima* di avere conferma dal server, assumendo che l'operazione avrà successo, per dare percezione di maggiore reattività. |
| **Route guard (navigation guard)** | Funzione eseguita dal router prima/dopo una navigazione, usata per bloccare o redirigere l'accesso a una rotta (es. per autenticazione). |
| **Interceptor (HTTP)** | Funzione che intercetta ogni richiesta/risposta HTTP in un punto centrale, per aggiungere comportamento trasversale (token, gestione errori) senza ripeterlo in ogni chiamata. |
| **Single Source of Truth** | Principio per cui un dato ha un solo "proprietario" autorevole nello stato dell'app; tutti gli altri punti che lo mostrano lo derivano da lì, non ne tengono una copia propria. |

## Come si combinano in una SPA moderna

Una SPA Vue/React moderna tipicamente è: **Composition API** (o Hooks) per la logica dei componenti, **store centralizzato** (Pinia/Vuex/Redux) come Single Source of Truth per lo stato condiviso, **service layer** per isolare le chiamate HTTP, **route guard** per proteggere le rotte. Il pattern architetturale complessivo che ne risulta è una forma di **MVVM**: il componente (`View`) non parla mai direttamente con il backend, passa sempre dal ViewModel (store), che a sua volta usa il Model (service layer) per i dati.
