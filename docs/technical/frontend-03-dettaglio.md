# Frontend — Livello 3: Dettaglio con codice reale

## Store Pinia completo — `stores/customers.store.ts`

```ts
export const useCustomersStore = defineStore('customers', () => {
  const items = ref<Customer[]>([])
  const selectedItem = ref<Customer | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)
  const totalCount = ref(0)
  const page = ref(1)
  const pageSize = ref(25)
  const search = ref('')

  async function fetchAll() {
    loading.value = true
    error.value = null
    try {
      const result = await customersService.getAll({
        search: search.value || undefined,
        page: page.value,
        pageSize: pageSize.value,
      })
      items.value = result.items
      totalCount.value = result.totalCount
    } catch {
      error.value = i18n.global.t('errors.loadCustomers')
    } finally {
      loading.value = false
    }
  }

  async function create(data: CreateCustomerRequest) {
    await customersService.create(data)
    await fetchAll()               // rilegge la lista intera
  }

  async function remove(id: string) {
    await customersService.delete(id)
    items.value = items.value.filter(x => x.id !== id)   // optimistic update, no re-fetch
    totalCount.value--
  }

  return { items, selectedItem, loading, error, totalCount, page, pageSize, search,
            fetchAll, fetchById, create, update, remove }
})
```
Nota le due strategie diverse di aggiornamento dopo una mutazione: `create`/`update` rifanno `fetchAll()` (semplice ma un round-trip in più), `remove` aggiorna `items` localmente (più reattivo, nessun round-trip).

## Pagina lista CRUD completa — `pages/data/CustomersPage.vue`

```vue
<script setup lang="ts">
const { mobile } = useDisplay()
const store = useCustomersStore()
const { can } = usePermission()
const { generalError, handleError, clearErrors } = useApiErrors()

const deleteDialog = ref(false)
const deleteTarget = ref<string | null>(null)

async function confirmDelete() {
  if (!deleteTarget.value) return
  deleteLoading.value = true
  try {
    await store.remove(deleteTarget.value)
    deleteDialog.value = false
  } catch (err) {
    handleError(err, t('common.error'))
  } finally {
    deleteLoading.value = false
  }
}

function onUpdatePage(newPage: number) {
  store.page = newPage
  store.fetchAll()
}

watch(() => store.search, () => {
  store.page = 1
  store.fetchAll()
}, { debounce: 300 } as any)

onMounted(() => store.fetchAll())
</script>

<template>
  <v-data-table-server
    v-if="!mobile"
    :headers="headers"
    :items="store.items"
    :items-length="store.totalCount"
    :loading="store.loading"
    :items-per-page="store.pageSize"
    :page="store.page"
    item-value="id"
    @update:page="onUpdatePage"
  >
    <template #item.actions="{ item }">
      <v-btn v-if="can('customers.write')" icon="mdi-pencil"
             :to="{ name: 'customer-detail', params: { id: item.id } }" />
      <v-btn v-if="can('customers.delete')" icon="mdi-delete"
             @click="openDeleteDialog(item.id)" />
    </template>
  </v-data-table-server>

  <!-- su mobile: v-list con paginazione manuale, non v-data-table-server -->
</template>
```
Punti chiave del pattern:
- **Responsive duplicato**: `v-data-table-server` per desktop, `v-list` + paginazione manuale (`onUpdatePage(store.page ± 1)`) per mobile — pattern ripetuto identico in tutte le pagine lista del progetto, non estratto in un componente condiviso.
- **Permessi UI**: ogni azione di riga è avvolta in `v-if="can('permesso')"` — il permesso non nasconde solo l'azione ma anche l'endpoint lato backend (difesa in profondità: la UI nasconde, il backend rifiuta comunque).
- **Eliminazione**: dialog di conferma locale alla pagina (non un componente `ConfirmDialog.vue` condiviso in questo caso specifico — verifica caso per caso quale pattern seguire).
- ⚠️ **Da verificare**: l'opzione `{ debounce: 300 }` passata a `watch()` **non è un'opzione nativa** dell'API `watch` di Vue 3 — il cast `as any` lo conferma implicitamente. Il progetto usa altrove `watchDebounced` di `@vueuse/core` per lo stesso scopo (vedi skill `search-filter`); qui potrebbe non funzionare come atteso (il debounce verrebbe silenziosamente ignorato da Vue, che eseguirebbe il watcher a ogni keystroke). TODO: verificare se `store.search` è già bindato con un debounce a monte (es. nel v-text-field) o se questa riga va corretta con `watchDebounced`.

## Pagina form Create/Edit — `pages/users/UserDetailPage.vue`

A differenza della skill `vuetify-dialog-form` (pensata per un `v-dialog`), qui il pattern Create/Edit è applicato a una **pagina intera** condivisa fra le due route `user-create` e `user-detail`:

```vue
<script setup lang="ts">
const isCreate = route.name === 'user-create'
const userId = route.params.id as string

const formRef = ref()
const form = reactive({ email: '', displayName: '', password: '', isActive: true, /* ... */ })
const saving = ref(false)
const { fieldErrors, generalError, handleError, clearErrors } = useApiErrors()

const rules = {
  required: (v: string) => !!v?.trim() || t('validation.required'),
  email: (v: string) => /.+@.+\..+/.test(v) || t('validation.email'),
  passwordMin: (v: string) => v.length >= 8 || t('validation.passwordMin'),
}

onMounted(async () => {
  await Promise.all([rolesStore.fetchRoles(), groupsStore.fetchGroups()])
  if (!isCreate) {
    await usersStore.fetchUser(userId)
    // popola form.* da usersStore.selectedUser
  }
})

async function save() {
  const { valid } = await formRef.value.validate()
  if (!valid) return

  saving.value = true
  clearErrors()
  try {
    if (isCreate) await usersStore.createUser(form)
    else await usersStore.updateUser(userId, { displayName: form.displayName, /* ... */ })
    toast.success(t('common.saved'))
    await router.push({ name: 'users' })
  } catch (err) {
    handleError(err, t('common.error'))
  } finally {
    saving.value = false
  }
}
</script>

<template>
  <v-form ref="formRef" @submit.prevent="save">
    <v-text-field v-model="form.email" :disabled="!isCreate"
                  :rules="isCreate ? [rules.required, rules.email] : []"
                  :error-messages="fieldErrors['email']" />
    <v-text-field v-if="isCreate" v-model="form.password" type="password"
                  :rules="[rules.required, rules.passwordMin]" />
    <v-switch v-if="!isCreate" v-model="form.isActive" />
    <v-btn type="submit" color="primary" :loading="saving">{{ t('common.save') }}</v-btn>
  </v-form>
</template>
```
Pattern da replicare per un nuovo form Create/Edit:
- **Stesso componente per create e edit**, distinto da `isCreate = route.name === '<entità>-create'`, con campi condizionali (`v-if="isCreate"` per password/campi solo-creazione, `v-if="!isCreate"` per campi solo-modifica come `isActive`).
- `formRef.value.validate()` **prima** di procedere al salvataggio — non affidarsi solo alle regole inline.
- `useApiErrors()` mappa gli errori 400 di FluentValidation (`fieldErrors`, chiavi in **lowercase**, es. `fieldErrors['displayname']` non `displayName`) sui singoli campi via `:error-messages`, e un eventuale errore generale (`generalError`) mostrato in un `v-alert`.
- `saving` pilota lo stato `:loading` del bottone submit, evitando doppio submit.

## Composable permessi — `composables/usePermission.ts`

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
Usato ovunque nei template per condizionare `v-if` su bottoni/azioni. I permessi disponibili arrivano dal claim `permissions` del JWT, letti in `auth.store.ts` al login/refresh.

## Client axios — `plugins/axios.ts` (vedi anche [frontend-02](frontend-02-funzionamento.md))

```ts
export const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  withCredentials: true,
})

api.interceptors.request.use(config => {
  const authStore = useAuthStore()
  if (authStore.token) config.headers.Authorization = `Bearer ${authStore.token}`
  config.headers['Accept-Language'] = authStore.language ?? 'it'
  useServerWakeStore().start()
  return config
})

api.interceptors.response.use(
  res => { useServerWakeStore().stop(true); return res },
  async error => {
    useServerWakeStore().stop(false)
    const status = error.response?.status
    const isAuthRequest = /* url include /login o /msal-login */ true
    if (status === 401 && !isLoggingOut && !isAuthRequest) {
      isLoggingOut = true
      try { await useAuthStore().logout() } finally { isLoggingOut = false }
    }
    if (status >= 500 && !error.config.url.includes('/api/error-logs')) {
      api.post('/api/error-logs', { level: 'Error', source: 'Frontend', message: /* ... */ }).catch(() => {})
    }
    return Promise.reject(error)
  }
)
```

## Come aggiungere una nuova view/rotta seguendo le convenzioni del progetto

1. `services/<entità>.service.ts` — metodi CRUD che chiamano `api` e ritornano `.then(r => r.data)`.
2. `stores/<entità>.store.ts` — `defineStore('<entità>', () => {...})` con lo stesso scheletro di `customers.store.ts` (state: `items/selectedItem/loading/error/totalCount/page/pageSize/search`; actions: `fetchAll/fetchById/create/update/remove`).
3. `pages/.../<Entità>Page.vue` — lista, seguendo il pattern `v-data-table-server` desktop + `v-list` mobile di `CustomersPage.vue`.
4. Se serve create/edit: `pages/.../<Entità>DetailPage.vue`, unico componente per le due route `<entità>-create`/`<entità>-detail`, seguendo il pattern di `UserDetailPage.vue`.
5. In `router/index.ts`, aggiungi le route con `component: () => import(...)` (lazy, sempre) e `meta: { requiresAuth: true, permission: '<entità>.read', title: '...', section: '...' }`.
6. Aggiungi le chiavi di traduzione in `locales/it.ts` **e** `en.ts` sotto il namespace `<entità>`.
7. Aggiungi i tipi in `types/api.types.ts` (`<Entità>`, `Create<Entità>Request`, `Update<Entità>Request`) — **a mano**, tenendoli sincronizzati con i record C# del backend.
8. Se la entità compare nel menu, aggiorna `config/sections.config.ts`.

Skill di progetto correlate: `vue-feature` (scaffolding store+service+page+route), `vuetify-dialog-form` (per dialog invece di pagina full), `pagination`, `search-filter`, `permissions`.

## Come intervenire in sicurezza nel frontend

**Convenzioni obbligatorie** (da `apps/frontend/CLAUDE.md`, verificate nel codice):
- Tutti i componenti in `<script setup lang="ts">`.
- **Nessuna chiamata axios/fetch diretta** in componenti o store — sempre passando dal service layer.
- **Nessun testo hardcoded** nei template — sempre `t('chiave')`, chiavi strutturate `dominio.elemento` in entrambi `it.ts`/`en.ts`.
- Ogni route protetta deve avere `meta.permission`.
- Componenti indicativamente non oltre ~150 righe — se crescono, estrarre in composable.
- Preferire `v-if` a `v-show` per elementi condizionati dai permessi (evita che l'elemento nascosto resti comunque nel DOM).
- Colori sempre dal tema Vuetify (`color="primary"`, `color="error"`), mai esadecimali hardcoded.

**Dove aggiungere codice nuovo**: segui la struttura a 8 passi sopra; componenti realmente riusabili tra più feature vanno in `components/shared/`, non duplicati per pagina.

**Cosa NON toccare senza motivo**:
- `plugins/axios.ts` — è il punto singolo di attach del JWT e di gestione 401/5xx; una modifica sbagliata rompe autenticazione o logging errori per **tutta** l'app.
- `plugins/vuetify.ts` (blocco `defaults`) — cambia l'aspetto di default di ogni componente Vuetify nell'app, non solo di uno.
- `router/index.ts` (guardie `beforeEach`/`afterEach`) — logica di sicurezza condivisa da tutte le route.
- `types/api.types.ts` — va aggiornato in sincronia con i DTO C# backend: un disallineamento qui produce errori silenziosi a runtime (TypeScript non può verificare la forma reale della risposta HTTP).
