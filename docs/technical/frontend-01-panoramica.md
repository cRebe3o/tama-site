# Frontend — Livello 1: Panoramica tecnologie

> Progetto: `apps/frontend` — Vue 3 (Composition API) + Vuetify 3, build con Vite. Verificato in `package.json`.

## Framework e build

| Tecnologia | Versione | Ruolo |
|---|---|---|
| **Vue 3** | ^3.5.0 | Framework UI. Il progetto usa **esclusivamente Composition API** con `<script setup>` (nessun componente Options API riscontrato). |
| **Vite** | ^5.4.0 | Dev server e bundler. Script `dev`/`build`/`preview` in `package.json`. |
| **TypeScript** + **vue-tsc** | ^5.5.0 / ^2.1.0 | Tipizzazione statica; `build` esegue `vue-tsc` prima di `vite build` (fallisce la build se ci sono errori di tipo). |
| **vite-plugin-vuetify** | ^2.0.0 | Integrazione Vuetify con Vite (tree-shaking automatico dei componenti). |

## UI

| Tecnologia | Ruolo |
|---|---|
| **Vuetify 3** (^3.7.0) | Libreria di componenti Material Design. Quasi tutta la UI è costruita con componenti Vuetify (`v-data-table-server`, `v-form`, `v-dialog`, ecc.), con pochi componenti custom condivisi in `components/shared/`. |
| **@mdi/font** (^7.4.0) | Set di icone Material Design Icons usato da Vuetify. |

## Routing e stato

| Tecnologia | Ruolo |
|---|---|
| **vue-router** | ^4.4.0 | Routing SPA, con guardie di navigazione per autenticazione e permessi (vedi [frontend-02](frontend-02-funzionamento.md)). |
| **Pinia** | ^2.2.0 | State management. Uno store per feature/dominio (`customers.store.ts`, `auth.store.ts`, ecc.), tutti scritti in **setup syntax** (`defineStore('id', () => {...})`). |

## Comunicazione con il backend

| Tecnologia | Ruolo |
|---|---|
| **axios** | ^1.7.0 | Client HTTP, istanziato una volta in `plugins/axios.ts` con interceptor centralizzati per JWT, lingua e gestione errori. |
| **@azure/msal-browser** | ^3.0.0 | Libreria Microsoft per il login Azure AD lato client, usata solo se `VITE_AUTH_STRATEGY=msal`. |

## Altro

| Tecnologia | Ruolo |
|---|---|
| **vue-i18n** | ^10.0.0 | Internazionalizzazione, due lingue supportate (`it`/`en`), dizionari in `locales/*.ts`. |
| **@vueuse/core** | ^11.0.0 | Utility Composition API (es. `watchDebounced` per la ricerca con debounce). |

## Testing

| Tecnologia | Ruolo |
|---|---|
| **Vitest** + **@vitest/coverage-v8** | Test runner unitario/integrazione, ambiente `jsdom`. |
| **@vue/test-utils** | Montaggio componenti Vue nei test. |

## Qualità codice

| Tecnologia | Ruolo |
|---|---|
| **ESLint** (`eslint`, `eslint-plugin-vue`, `typescript-eslint`) | Linting JS/TS/Vue. |
| **Prettier** | Formattazione codice. |

## Cosa NON c'è

- Nessuna generazione automatica dei tipi da OpenAPI/Swagger: i tipi TypeScript in `types/api.types.ts` sono scritti e mantenuti **a mano**, in parallelo ai DTO C# del backend — vanno tenuti sincronizzati manualmente quando cambia un contratto API.
- Nessun handler globale di errori JavaScript (`window.onerror`, `app.config.errorHandler`): la cattura errori verso il backend avviene solo per le risposte HTTP 5xx via interceptor axios (vedi [frontend-02](frontend-02-funzionamento.md)).
- Nessun framework CSS aggiuntivo oltre a Vuetify (niente Tailwind/Bootstrap).
