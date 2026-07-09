# Livello 1 — Panoramica: cosa c'è da fare

Nessun codice in questo documento: solo la mappa completa dei passaggi, nell'ordine in cui vanno affrontati, e il motivo di quell'ordine. Il dettaglio di ciascuno è nei livelli successivi.

## L'idea di fondo: "vertical slice", non "layer per layer"

Tama **non** si costruisce una feature scrivendo prima *tutti* i modelli, poi *tutti* i repository, poi *tutti* gli handler. Si costruisce **una fetta verticale completa** — dal database fino al pixel sullo schermo — per una singola entità (qui: `Extra`). Backend e frontend restano comunque due mondi separati con il proprio ordine interno, ma **dentro** ciascuno si procede "dal basso": prima il dato, poi chi lo legge/scrive, poi chi lo espone.

Motivo pratico: ogni passaggio, una volta completato, è testabile da solo (es. con Swagger/Scalar per il backend, prima ancora che esista una riga di frontend). Se si sbaglia un passaggio, l'errore si isola subito invece di emergere solo alla fine con l'intera feature assemblata.

## Backend — 6 passaggi

1. **Documento MongoDB** (`ExtraDocument`) — la forma del dato così come vive nel database. Non è una "migration": MongoDB non ha schema rigido, la struttura è implicita nella classe C#.
2. **Repository** (`IExtraRepository` / `ExtraRepository`) — l'unico punto del codice che parla direttamente con la collection Mongo `extras`.
3. **Query** (lettura: `GetExtras`, `GetExtraById`) — nessuna mutazione, solo lettura e mappatura a DTO di risposta.
4. **Command Create** (`CreateExtra`) — Command + Validator (FluentValidation) + Handler, che scrive un nuovo documento.
5. **Command Update/Delete** (`UpdateExtra`, `DeleteExtra`) — stesso schema del Create, con la differenza di dover prima recuperare il documento esistente.
6. **Endpoints** (`ExtraEndpoints.cs`) — l'unico punto dove tutto quanto sopra diventa raggiungibile via HTTP, con la registrazione dei permessi (`extras.read`, `extras.write`, `extras.delete`) e l'aggancio in `MapAllEndpoints()`.

A questo punto la feature è **completa e testabile lato backend**, indipendentemente dal frontend (via Scalar/Swagger UI su `/scalar` o `/swagger`, con un JWT valido). Il livello 3 include una checklist di verifica intermedia esattamente per questo punto — conviene eseguirla prima di iniziare il frontend.

## Frontend — 6 passaggi

7. **Tipi TypeScript** (`types/api.types.ts`) — l'equivalente TypeScript dei DTO C# creati al passaggio 3-5. Mantenuti a mano, non generati.
8. **Service** (`services/extras.service.ts`) — le chiamate HTTP pure (axios), nessuna logica.
9. **Store Pinia** (`stores/extras.store.ts`) — lo stato condiviso dell'app per gli extra (`items`, `loading`, `error`, ecc.) e le azioni che orchestrano il service.
10. **Pagina lista** (`pages/plus/ExtrasPage.vue`) — tabella con elenco, ricerca, azioni di modifica/eliminazione.
11. **Dialog di creazione/modifica** — form per creare/modificare un extra (descrizione + costo).
12. **Route + voce di menu** — registrazione in `router/index.ts` (con `meta.permission`) e aggiunta della nuova sezione "PLUS" in `config/sections.config.ts` (menu laterale).

A questo punto la feature è visibile e utilizzabile da un utente con i permessi giusti.

## Trasversale, da non dimenticare

Questi tre punti **non sono un passaggio numerato a sé**, perché toccano quasi tutti i passaggi sopra invece di essere un file isolato:

- **Permessi**: `extras.read` / `extras.write` / `extras.delete` vanno seminati nel database (collection `permissions`) e assegnati almeno al ruolo amministratore, altrimenti nessuno — nemmeno l'amministratore — vedrà la nuova sezione, anche a codice perfettamente scritto. E dopo l'assegnazione serve **rifare il login**: i permessi viaggiano dentro il JWT emesso al momento del login, un token già in tasca non si aggiorna da solo.
- **Traduzioni**: ogni stringa visibile (titolo pagina, colonne tabella, messaggi di errore, voce di menu) va aggiunta **sia** in `locales/it.ts` **sia** in `locales/en.ts`. Il progetto non ammette testo hardcoded nei template.
- **Audit log**: se si vuole tracciare chi ha creato/modificato/eliminato un extra (come avviene per i clienti), va aggiunta la scrittura esplicita di un `AuditLogDocument` in ciascun Handler Create/Update/Delete — non è automatico.

## Cosa NON serve fare (rispetto ad altri stack che si potrebbero conoscere)

Per chi arriva da un progetto con Entity Framework / SQL / ORM relazionale, questi passaggi **non esistono** in Tama e vanno consapevolmente saltati:

- Nessuna migration SQL da generare o applicare: la "creazione della tabella" è semplicemente il primo documento che viene inserito nella collection `extras` (MongoDB la crea automaticamente al primo insert).
- Nessun mapping ORM/AutoMapper da configurare: la conversione tra Document Mongo e DTO di risposta si scrive a mano, dentro l'Handler.
- Nessuna generazione automatica di client API per il frontend: i tipi TypeScript si scrivono a mano e vanno tenuti allineati ai record C# "a occhio".

## Ordine consigliato di lettura del livello 3

Il documento di dettaglio ([03-dettaglio](03-dettaglio.md)) segue esattamente questi 12 passaggi, nello stesso ordine. Non è necessario scriverli tutti prima di testare: dopo il passaggio 6 il backend è già verificabile da solo, e dopo il passaggio 9 anche lo store è ispezionabile in console/DevTools prima ancora di avere una UI.
