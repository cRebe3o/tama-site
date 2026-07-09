# Livello 2 — Dove si inserisce nel progetto esistente

Il livello 1 ha elencato *cosa* creare. Questo documento spiega *dove* si aggancia a quello che in Tama esiste **già** e non va reinventato: struttura del menu, permessi, convenzioni di naming, e quale dominio esistente conviene usare come riferimento diretto da cui copiare (non "ispirarsi", proprio copiare la struttura e adattare i nomi).

## Il dominio di riferimento: `accessories`, non `customers`

La documentazione tecnica di Tama ([../technical/backend-03-dettaglio.md](../technical/backend-03-dettaglio.md), [../technical/frontend-03-dettaglio.md](../technical/frontend-03-dettaglio.md)) usa **Customers** come esempio ricorrente perché è il dominio più ricco (validazioni, audit, relazioni con i preventivi). Per "PLUS"/"EXTRA" conviene invece guardare **`accessories`** — confermato nel codice come collection reale con indici propri (vedi [../technical/00-indice.md](../technical/00-indice.md), sezione "punti di attenzione") — perché è strutturalmente il caso più vicino: pochi campi semplici, nessuna relazione con altre entità, nessuna logica di business oltre al CRUD.

Concretamente, per ogni file che si crea per `Extra`, il file equivalente già esistente per `Accessory` è la controparte da tenere aperta a fianco mentre si scrive:

| Da creare per `Extra` | Controparte esistente da copiare |
|---|---|
| `Features/Extras/*` | `Features/Accessories/*` |
| `Features/_Shared/Documents/ExtraDocument.cs` | `Features/_Shared/Documents/AccessoryDocument.cs` |
| `Features/_Shared/Repositories/ExtraRepository.cs` | `Features/_Shared/Repositories/AccessoryRepository.cs` |
| `stores/extras.store.ts` | `stores/accessories.store.ts` |
| `services/extras.service.ts` | `services/accessories.service.ts` |
| `pages/plus/ExtrasPage.vue` | pagina lista di un dominio semplice sotto `pages/data/` |

Se in fase di implementazione reale un dettaglio di `accessories` risultasse diverso da quanto descritto qui (il codice è l'unica fonte di verità, questa guida è un punto di partenza — vedi l'avviso nell'[indice](00-indice.md)), va seguito il codice reale, non questa guida.

## Menu laterale: da "DATI" a "PLUS"

Il menu laterale di Tama non è hardcoded nel template di ogni pagina: è generato da una configurazione unica, `apps/frontend/src/config/sections.config.ts` (citata in [../technical/frontend-03-dettaglio.md](../technical/frontend-03-dettaglio.md), passo 8). Ogni voce del menu (sezione + sotto-voci) corrisponde a un blocco di questa configurazione, che associa:

- un'**etichetta** (chiave di traduzione, non stringa fissa),
- un'**icona** Material Design Icons (`mdi-*`),
- una lista di **item**, ciascuno con nome rotta e permesso richiesto.

"DATI" è una sezione esistente con più voci (Clienti, Accessori, Serie, ecc.); "PLUS" è una **sezione nuova**, allo stesso livello di "DATI" nell'array di configurazione, inizialmente con una sola voce ("Extra"). Il livello 3 mostra il blocco di configurazione esatto da aggiungere.

Il menu **non** ha una logica propria di visibilità: ogni voce compare solo se `authStore.permissions` (letta dal componente che itera `sections.config.ts`) contiene il permesso associato — stesso composable `usePermission` descritto in [../architecture/frontend-03-nel-progetto.md](../architecture/frontend-03-nel-progetto.md). Questo significa che **finché il permesso `extras.read` non esiste ed è assegnato**, la voce "PLUS → Extra" resta invisibile anche a codice completo — non è un bug da cercare, è il comportamento atteso.

## Permessi: naming e propagazione

La convenzione, verificata nel codice reale ([../technical/backend-03-dettaglio.md](../technical/backend-03-dettaglio.md), "come intervenire in sicurezza"), è `<entità_plurale>.<azione>`, azione sempre una di `read`/`write`/`delete`. Per Extra:

- `extras.read` — vedere l'elenco e i dettagli
- `extras.write` — creare e modificare (un solo permesso per entrambe le operazioni, come per `customers.write`)
- `extras.delete` — eliminare

Questi tre permessi vanno **seminati** (creati come documenti nella collection `permissions`) e poi **assegnati** ad almeno un ruolo (tipicamente "Amministratore") perché qualcuno possa effettivamente usarli. In Tama questo passaggio è coperto dalla skill di progetto `permissions`, richiamata anche in [../technical/backend-03-dettaglio.md](../technical/backend-03-dettaglio.md) punto 8: seed permesso → assegnazione a ruolo → claim nel JWT al login → guardia sia lato route (`meta.permission`) sia lato menu (`sections.config.ts`) sia lato bottoni della pagina (`v-if="can(...)"`). Sono **quattro punti diversi**, non uno solo: dimenticarne anche solo uno lascia la feature parzialmente inaccessibile o, peggio, un bottone visibile che poi fallisce con 403 alla chiamata.

Operativamente, per creare e assegnare i permessi non serve scrivere codice: il modo più diretto è usare le pagine di amministrazione già esistenti — **System → Permissions** per creare `extras.read`, `extras.write`, `extras.delete`, e **System → Roles** per assegnarli al ruolo desiderato (sono le stesse pagine `system/permissions`/`system/roles` citate in [../tools/devtools-pinia.md](../tools/devtools-pinia.md) a proposito degli store `permissions` e `auth`). In alternativa, per ambienti che partono da un database vuoto, si possono aggiungere al `DataSeeder` del backend.

> ⚠️ **Dopo l'assegnazione serve un nuovo login.** I permessi viaggiano dentro il claim `permissions` del JWT, che viene costruito **al momento del login**: un token emesso prima dell'assegnazione non contiene i permessi nuovi, e l'utente continuerà a non vedere la sezione (e a ricevere 403 sulle chiamate) finché non esce e rientra. Quando "il permesso c'è ma non funziona", la prima cosa da controllare è l'array `permissions` nello store `auth` con il pannello Pinia di Vue DevTools ([../tools/devtools-pinia.md](../tools/devtools-pinia.md)): se lì i permessi `extras.*` non compaiono, il problema è il token vecchio, non il codice.

## Route e i18n: stesso schema di ogni altra entità

Nessuna variazione rispetto al passo 5 di [../technical/frontend-03-dettaglio.md](../technical/frontend-03-dettaglio.md) ("Come aggiungere una nuova view/rotta"): la rotta `extras` va registrata con `component: () => import(...)` lazy e `meta: { requiresAuth: true, permission: 'extras.read', title: '...', section: 'plus' }` — l'unica novità rispetto a un'entità sotto "DATI" è il valore di `section`, che deve corrispondere alla chiave della nuova sezione "PLUS" in `sections.config.ts` (serve a evidenziare la sezione corretta nel menu quando si è su quella pagina, come già descritto in `devtools-router.md` per il debug di questo campo).

Le chiavi di traduzione seguono lo stesso namespace-per-entità già in uso: tutto sotto `extras.*` in `it.ts`/`en.ts` (`extras.title`, `extras.columnDescription`, `extras.columnCost`, `extras.created`, `extras.deleted`, ecc.), più le nuove voci di errore sotto `errors.*` (`errors.loadExtras`, `errors.createExtra`, ...) — stesso pattern di `errors.loadCustomers` visto in `customers.store.ts`.

## Cosa NON bisogna creare da zero

Riutilizzabili così come sono, senza toccarli:

- `plugins/axios.ts` (client HTTP centralizzato — allega automaticamente il JWT).
- `composables/usePermission.ts` (già generico, funziona con qualunque stringa di permesso).
- I pipeline behavior MediatR (`LoggingBehavior`, `ValidationBehavior`) — si agganciano automaticamente a qualunque nuovo Command/Query per assembly-scan, nessuna registrazione manuale per-feature.
- `ExceptionHandlingMiddleware` — già gestisce `ValidationException`/`NotFoundException` generici; nessun errore custom serve per un CRUD semplice come Extra.

L'unico registro **manuale** da non dimenticare, sul lato backend, è `Features/_Shared/Extensions/EndpointExtensions.cs::MapAllEndpoints()`: la riga `app.MapExtraEndpoints();` va aggiunta a mano, esattamente come per ogni altra feature — è il passaggio esplicitamente segnalato come "facile da dimenticare" in [../technical/backend-03-dettaglio.md](../technical/backend-03-dettaglio.md).

## Riepilogo: la checklist "non-file" da spuntare

Oltre ai 12 file/passaggi del livello 1, prima di considerare la feature completa vanno verificati questi punti che **non sono un singolo file** ma una modifica a qualcosa di condiviso:

- [ ] `MapAllEndpoints()` aggiornato con `MapExtraEndpoints()`
- [ ] `sections.config.ts` aggiornato con la sezione "PLUS"
- [ ] `router/index.ts` aggiornato con la rotta `extras` e `meta.permission`
- [ ] `it.ts` **e** `en.ts` aggiornati con il namespace `extras.*`
- [ ] Permessi `extras.read`/`write`/`delete` seminati e assegnati a un ruolo
- [ ] Nuovo login effettuato dopo l'assegnazione dei permessi (il JWT emesso prima non li contiene)
- [ ] (Opzionale, se serve tracciamento) `AuditLogDocument` scritto negli Handler Create/Update/Delete

Il livello 3 mostra il codice esatto per ognuno di questi punti.
