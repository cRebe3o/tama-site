# Concetti architetturali — Indice

Documentazione **tecnologia-agnostica**: spiega i pattern e le scelte architetturali generali (Minimal API, Vertical Slice vs Clean/Onion Architecture, CQRS, Composition API, MVVM, ecc.), confrontandoli con le alternative più comuni. Il progetto Tama è citato solo come esempio concreto per ancorare la teoria al codice reale.

Per la documentazione tecnica specifica di Tama (come funziona *questo* progetto, senza il confronto teorico con le alternative), vedi [../technical/00-indice.md](../technical/00-indice.md).

## Come è organizzata

Ogni area (backend/frontend) ha 3 livelli, dal più astratto al più concreto:

| Livello | Cosa contiene |
|---|---|
| **01 — Panoramica dei pattern** | Elenco dei concetti/pattern rilevanti, una definizione minima ciascuno, nessun confronto |
| **02 — Confronto con le alternative** | Spiegazione approfondita di ciascun pattern, messo a confronto con le alternative più comuni (es. Layered vs Clean vs Vertical Slice), con diagrammi comparativi |
| **03 — Dove si vedono nel codice** | Ogni concetto ancorato a un file/snippet reale del repository Tama |

## Documenti

| Documento | Argomenti |
|---|---|
| [backend-01-pattern](backend-01-pattern.md) | Minimal API, Vertical Slice, Layered, Clean/Onion, CQRS, Mediator, Pipeline/Decorator, Repository, DI e service lifetime, Document DB, embedding/denormalizzazione, counter pattern, DTO, Claims-based auth |
| [backend-02-confronto](backend-02-confronto.md) | Minimal API vs Controller; Layered vs Clean/Onion vs Vertical Slice (con diagramma); CQRS+Mediator; Pipeline=Decorator; Repository con Document DB; Document-oriented vs Relazionale (transazioni, denormalizzazione, contatori); DI lifetimes e captive dependency; Claims vs Role-based |
| [backend-03-nel-progetto](backend-03-nel-progetto.md) | Ogni concetto ancorato al codice: `Features/Customers/`, `DraftDocument` (embedding/denormalizzazione), cascade delete con transazioni commentate, counter pattern, `IServiceScopeFactory` nel middleware |
| [frontend-01-pattern](frontend-01-pattern.md) | Reattività fine-grained, Options API, Composition API, `<script setup>`, Composable, Store pattern, MVVM, Service layer, Optimistic update, Route guard, Interceptor, Single Source of Truth |
| [frontend-02-confronto](frontend-02-confronto.md) | Reattività come Observer pattern (e differenza con React); Options vs Composition API (con diagramma); Composable vs custom Hook; Pinia vs Vuex; MVVM applicato a una SPA (con diagramma); Single Source of Truth; Interceptor=Decorator; Optimistic vs re-fetch; sicurezza a due livelli |
| [frontend-03-nel-progetto](frontend-03-nel-progetto.md) | Ogni concetto ancorato a `CustomersPage.vue` (incluso accesso reattivo agli store), `customers.store.ts`, `plugins/axios.ts`, `router/index.ts` |

## I due concetti-chiave del progetto, in una frase

- **Backend**: Tama usa **Vertical Slice Architecture** (organizzazione per feature, non per layer tecnico) con **CQRS via MediatR** (Command/Query + Handler) invece di Controller+Service+Repository classici o di Clean/Onion Architecture a progetti separati — una scelta pragmatica che privilegia coesione e velocità di sviluppo su un dominio non eccessivamente complesso.
- **Frontend**: Tama usa **Composition API + Pinia setup-syntax**, realizzando di fatto un pattern **MVVM** (componente=View, store=ViewModel, service=Model) imposto esplicitamente come regola di progetto (`CLAUDE.md`: "nessuna logica di business nei componenti", "nessuna chiamata HTTP diretta fuori dai service").

Questi due principi — organizzare per funzionalità coesa, e separare rigidamente presentazione/orchestrazione/accesso dati — sono lo stesso principio applicato simmetricamente sui due lati dello stack.
