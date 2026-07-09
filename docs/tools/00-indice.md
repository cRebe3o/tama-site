# Tama — Vue DevTools

Guida all'uso delle **Vue DevTools** (estensione browser / standalone) applicata al progetto Tama (`apps/frontend`, Vue 3 Composition API + Pinia + Vuetify 3 + vue-router).

Ogni pannello di DevTools ha un documento dedicato, diviso su 3 livelli, dal più semplice al più avanzato:
- **Livello 1 — Base**: cos'è il pannello, come aprirlo, cosa mostra.
- **Livello 2 — Intermedio**: workflow tipico di utilizzo durante lo sviluppo quotidiano su Tama.
- **Livello 3 — Avanzato**: tecniche mirate per debug di problemi specifici, con esempi presi da file reali del progetto.

## Documenti

| Documento | Contenuto |
|---|---|
| [devtools-components](devtools-components.md) | Albero componenti, ispezione props/state/emits, editing live, "Open in editor", scoping su componenti nascosti in `v-if`/`v-for` |
| [devtools-pinia](devtools-pinia.md) | Ispezione store, time-travel su mutazioni, editing diretto dello state, differenza fra store "options" e "setup syntax" |
| [devtools-router](devtools-router.md) | Rotta corrente, `meta` (permessi/`requiresAuth`), verifica del route matching, debug dei redirect nei navigation guard |
| [devtools-timeline](devtools-timeline.md) | Eventi Component/Pinia/Router su una timeline temporale unica, correlazione fra un click utente e le mutazioni di stato che genera |

## Prerequisiti e installazione

- Estensione **Vue DevTools** installata nel browser: [Chrome Web Store](https://chromewebstore.google.com/detail/vuejs-devtools/nhdogjmejiglipccpnnnanhbledajbpd), [Edge Add-ons](https://microsoftedge.microsoft.com/addons/detail/vuejs-devtools/olofadcdnkkjdfgjcmjaadnlehnnihnl), [Firefox Add-ons](https://addons.mozilla.org/firefox/addon/vue-js-devtools/). In alternativa il pacchetto standalone `vue-devtools` (Electron) per ambienti senza estensioni (es. WebView).
- Per Vue 3 serve l'estensione **v6 o successiva** (la v5 supporta solo Vue 2). Esiste anche la nuova generazione **v7** (ripensata, distribuita anche come plugin Vite `vite-plugin-vue-devtools` che apre i pannelli in overlay dentro la pagina stessa): i pannelli descritti qui esistono in entrambe, ma nomi e disposizione possono variare leggermente — questa guida segue la terminologia della v6.
- Il progetto va avviato in modalità sviluppo (`pnpm --filter frontend dev`, Vite): DevTools **non funziona su build di produzione** a meno che `app.config.devtools` non venga forzato esplicitamente (Tama non lo fa — comportamento di default Vite/Vue: devtools disattivati in produzione).
- DevTools si aggancia automaticamente a qualunque app Vue 3 montata nella pagina (`main.ts` → `app.mount('#app')`).

## Quale pannello per quale problema

Tabella di orientamento rapido — dal sintomo al pannello giusto:

| Sintomo | Pannello | Cosa guardare |
|---|---|---|
| La tabella/lista è vuota o mostra dati vecchi | **Pinia** | `items`, `loading`, `error`, `search` dello store del dominio |
| Un dialog/form renderizza male o non si apre | **Components** | State locale della pagina (`dialog`, `editedItem`), props del componente Vuetify |
| Cliccando un link si finisce su login o dashboard | **Routes** + **Pinia** | `meta.permission` della rotta vs array `permissions` dello store `auth` |
| URL apre la pagina 404 inaspettatamente | **Routes** | Quale rotta ha vinto il match (`name: 'not-found'` = path sbagliato nel link) |
| "Cosa succede davvero quando clicco X?" | **Timeline** | Sequenza click → azioni/mutazioni Pinia → navigazioni, con timestamp |
| La ricerca sembra lenta o spara troppe richieste | **Timeline** | Un solo evento fetch ~300ms dopo l'ultimo tasto = debounce ok |
| Il menu evidenzia la sezione sbagliata | **Routes** + **Pinia** | `meta.section` della rotta vs store `navigation` |

## Se il tab Vue non compare o resta vuoto

Checklist nell'ordine di probabilità:

1. **Build di produzione**: si sta guardando il sito pubblicato invece del dev server locale. DevTools funziona solo su `pnpm dev`.
2. **Pagina caricata prima dell'estensione**: ricaricare la pagina con i DevTools del browser già aperti; a volte serve chiudere e riaprire i DevTools stessi.
3. **Icona estensione grigia** con tooltip "Vue.js not detected": l'app non si è montata — controllare la Console per errori JavaScript che bloccano `main.ts`.
4. **Versione estensione**: con l'estensione v5 (legacy) un'app Vue 3 non viene rilevata; verificare di avere la v6+.
5. **Finestre in incognito / profili separati**: l'estensione va abilitata esplicitamente per la navigazione in incognito nelle impostazioni del browser.

## Perché conviene usarli su Tama in particolare

Il frontend usa **Pinia in setup-syntax** per (quasi) ogni dominio (`stores/customers.store.ts`, `stores/auth.store.ts`, `stores/navigation.store.ts`, ecc.) e **vue-router con guardie basate su permessi** (`router/index.ts`). Questi due pattern sono esattamente i due casi dove DevTools dà il massimo vantaggio rispetto a `console.log`:

1. Con **oltre 15 store Pinia** attivi contemporaneamente (uno per dominio dati), è impraticabile tenere traccia manualmente di quale store contiene cosa: il pannello Pinia elenca tutti gli store montati e il loro state in tempo reale.
2. Con **guardie di navigazione che fanno redirect silenziosi** (`router.beforeEach` in `router/index.ts` reindirizza a `login` o `dashboard` senza errori in console), il pannello Router è spesso l'unico modo rapido per capire *perché* una rotta non si è aperta.
