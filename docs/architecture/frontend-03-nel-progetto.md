# Concetti architetturali — Frontend — Livello 3: dove si vedono nel codice di Tama

Ancoraggio al codice reale dei concetti spiegati in [frontend-02-confronto](frontend-02-confronto.md). Per il dettaglio implementativo completo, vedi [../technical/frontend-03-dettaglio.md](../technical/frontend-03-dettaglio.md).

## Composition API + `<script setup>`

File: `apps/frontend/src/pages/data/CustomersPage.vue`
```ts
<script setup lang="ts">
const store = useCustomersStore()
const { can } = usePermission()
const deleteDialog = ref(false)

async function confirmDelete() { /* ... */ }
</script>
```
Non c'è nessun `export default { data() {...} }` in nessun componente del progetto — è stato verificato con l'esplorazione del codice (report agente frontend) che **tutti** i componenti usano `<script setup>`. Se stai cercando "dove sono i `methods`" di un componente Vue in questo progetto, cerca semplicemente le `function` dichiarate nello `<script setup>`.

## Reattività e Single Source of Truth

Sempre in `CustomersPage.vue`, nota come lo store viene consumato nel template:
```html
<v-data-table-server
  :items="store.items"
  :items-length="store.totalCount"
  :loading="store.loading"
```
Il componente **non copia mai** lo stato dello store in variabili locali (`const items = store.items` non compare da nessuna parte nel progetto, e nemmeno `storeToRefs` — verificato con grep): accede sempre a `store.items` direttamente. È questo che mantiene viva la catena reattiva descritta in [frontend-02](frontend-02-confronto.md): quando `fetchAll()` nello store assegna `items.value = result.items`, la tabella si ridisegna da sola, senza eventi né callback.

I dati derivati sono sempre `computed`, mai copie: le intestazioni tradotte della tabella, ad esempio, sono `const headers = computed(() => [...])` — si ricalcolano da sole al cambio lingua perché leggono `t(...)`, che è reattivo.

## Composable

File: `apps/frontend/src/composables/usePermission.ts`
```ts
export function usePermission() {
  const authStore = useAuthStore()
  function can(permission: string): boolean {
    return authStore.permissions.includes(permission)
  }
  function canAny(...permissions: string[]): boolean {
    return permissions.some(p => authStore.permissions.includes(p))
  }
  return { can, canAny }
}
```
Usato identicamente in **ogni** pagina che deve nascondere/mostrare azioni in base ai permessi (`v-if="can('customers.write')"`). Se dovessi aggiungere una logica riusabile nuova (es. un composable per il debounce coerente con `@vueuse/core`), questa è la cartella (`composables/`) e questo il pattern da seguire: una funzione che ritorna un oggetto con lo stato/le funzioni esposte, niente classi.

## Pinia in setup-syntax

File: `apps/frontend/src/stores/customers.store.ts`
```ts
export const useCustomersStore = defineStore('customers', () => {
  const items = ref<Customer[]>([])
  const loading = ref(false)
  // ...
  async function fetchAll() { /* ... */ }
  return { items, loading, /* ... */, fetchAll }
})
```
Nota la somiglianza strutturale con un composable: `ref`, funzioni, `return` esplicito di ciò che va esposto. La differenza pratica tra questo e un composable "normale" è solo `defineStore(...)`: rende lo stato **condiviso e persistente per tutta la sessione dell'app** invece che locale al singolo componente che lo chiama.

## MVVM: View → ViewModel → Model

Tracciando una singola azione utente attraverso i tre livelli, nel codice reale:

1. **View** — `CustomersPage.vue`: `@click="openDeleteDialog(item.id)"` → poi `store.remove(deleteTarget.value)`.
2. **ViewModel** — `customers.store.ts`, funzione `remove(id)`: chiama il service, poi aggiorna lo stato locale (`items.value = items.value.filter(...)`).
3. **Model** — `customers.service.ts`, funzione `delete(id)`: `api.delete('/api/customers/' + id)`.

Il componente **non** chiama mai `customersService` direttamente, e lo store **non** chiama mai `axios` direttamente (chiama sempre `customersService`) — questa disciplina a due hop è quella imposta dal `CLAUDE.md` del frontend ("nessuna chiamata axios/fetch diretta nei componenti o negli store").

## Interceptor HTTP

File: `apps/frontend/src/plugins/axios.ts`:
```ts
api.interceptors.request.use(config => {
  const authStore = useAuthStore()
  if (authStore.token) config.headers.Authorization = `Bearer ${authStore.token}`
  return config
})
```
Ogni `services/*.service.ts` importa `api` da qui (`import { api } from '@/plugins/axios'`) invece di usare `axios` direttamente — è così che **tutte** le ~19 famiglie di chiamate del progetto ottengono automaticamente il token, senza doverlo aggiungere manualmente in ciascuna.

## Route guard

File: `apps/frontend/src/router/index.ts`:
```ts
router.beforeEach(async (to) => {
  const authStore = useAuthStore()
  if (to.meta.requiresAuth && !authStore.isAuthenticated) {
    return { name: 'login', query: { returnUrl: to.fullPath } }
  }
  if (to.meta.permission && !authStore.permissions.includes(to.meta.permission as string)) {
    return { name: 'dashboard' }
  }
})
```
Da notare: questo guard usa `authStore.permissions`, la stessa lista di permessi letta dal claim JWT `"permissions"` popolato dal backend (vedi documentazione concetti backend, sezione claims-based authorization) — è la **stessa fonte di verità**, replicata sul client per UX, non un sistema di permessi separato o ridondante da mantenere manualmente allineato.

## Optimistic/local update dopo conferma server

```ts
async function remove(id: string) {
  await customersService.delete(id)          // aspetta conferma server
  items.value = items.value.filter(x => x.id !== id)   // poi aggiorna localmente
  totalCount.value--
}
```
Da `customers.store.ts` — nota che l'`await` precede l'aggiornamento locale: se la `delete` fallisce, l'`throw` propagato dall'interceptor axios interrompe la funzione **prima** di modificare `items`, quindi lo stato locale non si disallinea mai da quello reale sul server in caso di errore.
