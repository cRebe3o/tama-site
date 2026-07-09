# Livello 3 — Dettaglio: codice completo, passo per passo

Segue esattamente i 12 passaggi elencati in [01-panoramica](01-panoramica.md). Ogni sezione ha il codice completo del file e, dove serve, il "perché" della scelta. Le convenzioni generali (naming, struttura cartelle, pipeline MediatR) sono quelle già descritte in [../technical/backend-03-dettaglio.md](../technical/backend-03-dettaglio.md) e [../technical/frontend-03-dettaglio.md](../technical/frontend-03-dettaglio.md): qui non vengono ripetute, solo applicate.

---

# BACKEND

## Passaggio 1 — Documento MongoDB

**File nuovo**: `apps/backend/Tama.Api/Features/_Shared/Documents/ExtraDocument.cs`

```csharp
using MongoDB.Bson;
using MongoDB.Bson.Serialization.Attributes;

namespace Tama.Api.Features._Shared.Documents;

public class ExtraDocument
{
    public ObjectId Id { get; set; }
    public string Description { get; set; } = string.Empty;

    [BsonRepresentation(BsonType.Decimal128)]
    public decimal Cost { get; set; }

    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

**Perché è fatto così**

- Nessun attributo `[BsonElement]`/`[Key]`: il progetto usa un `ConventionPack` globale registrato in `AddMongoDb` (vedi [../technical/backend-03-dettaglio.md](../technical/backend-03-dettaglio.md), "cosa NON toccare senza motivo") che mappa automaticamente `PascalCase` C# → `camelCase` nei documenti Mongo. Non serve annotare nulla campo per campo.
- `decimal` per `Cost`, non `double`: è un valore monetario, e `double` introdurrebbe errori di arrotondamento binario (il classico `0.1 + 0.2 ≠ 0.3`) — inaccettabili per un valore che finirà stampato nei preventivi.
- `[BsonRepresentation(BsonType.Decimal128)]` esplicito su `Cost`: senza questo attributo, il serializzatore di default del driver .NET salva `decimal` come **stringa** BSON (scelta storica del driver per non perdere precisione), il che renderebbe scorretti ordinamenti e impossibili somme direttamente in query Mongo. `Decimal128` è il tipo decimale nativo di MongoDB (dalla versione 3.4): mantiene la precisione **e** resta un numero. È l'unico attributo BSON che serve in tutta la classe — se il progetto avesse già registrato globalmente un `DecimalSerializer` con rappresentazione `Decimal128` (da verificare in `AddMongoDb`), l'attributo diventerebbe ridondante ma non dannoso.
- `CreatedAt`/`UpdatedAt`: presenti su **ogni** documento del progetto (stesso pattern di `CustomerDocument`), anche se qui non c'è ancora un caso d'uso che li legga attivamente — è la convenzione, non un'eccezione da giustificare caso per caso.
- Nessun campo di relazione: a differenza di `DraftDocument` (che referenzia `CustomerId`/`SeriesId`), `ExtraDocument` è un'entità "foglia" — non punta a nient'altro e, salvo sviluppi futuri, nessun'altra collection lo referenzia ancora.

## Passaggio 2 — Repository

**Files nuovi**: `Features/_Shared/Repositories/IExtraRepository.cs` e `ExtraRepository.cs`

```csharp
// 📄 IExtraRepository.cs
namespace Tama.Api.Features._Shared.Repositories;

public interface IExtraRepository
{
    Task<List<ExtraDocument>> GetAllAsync(CancellationToken ct);
    Task<ExtraDocument?> GetByIdAsync(ObjectId id, CancellationToken ct);
    Task InsertAsync(ExtraDocument doc, CancellationToken ct);
    Task UpdateAsync(ExtraDocument doc, CancellationToken ct);
    Task DeleteAsync(ObjectId id, CancellationToken ct);
}

// 📄 ExtraRepository.cs
public class ExtraRepository(IMongoDatabase db) : IExtraRepository
{
    private readonly IMongoCollection<ExtraDocument> _col = db.GetCollection<ExtraDocument>("extras");

    public async Task<List<ExtraDocument>> GetAllAsync(CancellationToken ct) =>
        await _col.Find(FilterDefinition<ExtraDocument>.Empty)
            .SortBy(x => x.Description)
            .ToListAsync(ct);

    public async Task<ExtraDocument?> GetByIdAsync(ObjectId id, CancellationToken ct) =>
        await _col.Find(x => x.Id == id).FirstOrDefaultAsync(ct);

    public async Task InsertAsync(ExtraDocument doc, CancellationToken ct) =>
        await _col.InsertOneAsync(doc, cancellationToken: ct);

    public async Task UpdateAsync(ExtraDocument doc, CancellationToken ct) =>
        await _col.ReplaceOneAsync(x => x.Id == doc.Id, doc, cancellationToken: ct);

    public async Task DeleteAsync(ObjectId id, CancellationToken ct) =>
        await _col.DeleteOneAsync(x => x.Id == id, ct);
}
```

**Registrazione** — `Features/_Shared/Extensions/RepositoryExtensions.cs::AddRepositories()` (MODIFICA, aggiungi una riga):

```csharp
services.AddScoped<IExtraRepository, ExtraRepository>();
```

**Perché è fatto così**

- Il nome della collection (`"extras"`) è passato come stringa letterale a `GetCollection<T>`, non c'è una costante centralizzata — stessa scelta osservata in tutti gli altri repository del progetto.
- `SortBy(x => x.Description)` in `GetAllAsync`: per una lista di sole ~5-20 righe come gli extra (non paginata, a differenza di `customers`), ordinare alfabeticamente lato database evita di doverlo fare lato frontend ad ogni render.
- Nessuna paginazione (`Skip`/`Limit`) in `GetAllAsync`, a differenza di `ICustomerRepository.GetPagedAsync`: gli extra sono un elenco corto e stabile (tipicamente configurato una volta dall'amministrazione), non ha senso pagarne il costo di complessità. Se in futuro diventasse una lista lunga, si aggiungerebbe la paginazione allora — non prematuramente ora.
- `AddScoped`, come gli altri repository del progetto: è il lifetime già in uso (lo conferma indirettamente il fatto che `ExceptionHandlingMiddleware` debba creare uno scope apposito per risolvere `IErrorLogRepository`, che è scoped — vedi [../architecture/backend-03-nel-progetto.md](../architecture/backend-03-nel-progetto.md), sezione sui DI lifetimes). Tecnicamente un repository così semplice potrebbe anche vivere come singleton (incapsula solo `IMongoCollection<T>`, che è thread-safe), ma seguire il lifetime degli altri repository conta più della micro-ottimizzazione: un lifetime "speciale" sarebbe un'eccezione da ricordare, e diventerebbe una captive dependency il giorno in cui il repository ricevesse una dipendenza scoped.
- **Indici**: per una collection così piccola non serve alcun indice per le prestazioni. Se però si volesse impedire due extra con la stessa descrizione, il punto giusto è un indice **unique** su `description` aggiunto in `EnsureIndexesAsync` (dove il progetto crea tutti gli indici all'avvio) — con l'avvertenza già documentata in [../technical/backend-03-dettaglio.md](../technical/backend-03-dettaglio.md): un indice unique su una collection che contiene già duplicati fa **fallire l'avvio dell'applicazione**. Il tentativo di inserire un duplicato a runtime diventerebbe automaticamente un **409 Conflict**, grazie al case `MongoWriteException`/`DuplicateKey` già presente in `ExceptionHandlingMiddleware` — nessun codice da scrivere negli Handler per gestirlo.

## Passaggio 3 — Query (lettura)

**Files nuovi**: `Features/Extras/GetExtras/` e `Features/Extras/GetExtraById/`

```csharp
// 📄 Features/Extras/GetExtras/GetExtrasQuery.cs
public record GetExtrasQuery : IRequest<List<ExtraResponse>>;

// 📄 Features/Extras/ExtraResponse.cs (nella radice della feature: condiviso da tutte le operazioni)
public record ExtraResponse(string Id, string Description, decimal Cost);

// 📄 Features/Extras/GetExtras/GetExtrasHandler.cs
public class GetExtrasHandler(IExtraRepository repo) : IRequestHandler<GetExtrasQuery, List<ExtraResponse>>
{
    public async Task<List<ExtraResponse>> Handle(GetExtrasQuery request, CancellationToken ct)
    {
        var docs = await repo.GetAllAsync(ct);
        return docs.Select(d => new ExtraResponse(d.Id.ToString(), d.Description, d.Cost)).ToList();
    }
}

// 📄 Features/Extras/GetExtraById/GetExtraByIdQuery.cs
public record GetExtraByIdQuery(string Id) : IRequest<ExtraResponse>;

// 📄 Features/Extras/GetExtraById/GetExtraByIdHandler.cs
public class GetExtraByIdHandler(IExtraRepository repo) : IRequestHandler<GetExtraByIdQuery, ExtraResponse>
{
    public async Task<ExtraResponse> Handle(GetExtraByIdQuery request, CancellationToken ct)
    {
        if (!ObjectId.TryParse(request.Id, out var id))
            throw new NotFoundException($"Extra {request.Id} non trovato");

        var doc = await repo.GetByIdAsync(id, ct)
            ?? throw new NotFoundException($"Extra {request.Id} non trovato");

        return new ExtraResponse(doc.Id.ToString(), doc.Description, doc.Cost);
    }
}
```

**Perché è fatto così**

- `ExtraResponse` è un `record` unico condiviso da tutte le operazioni (lo restituiscono anche `CreateExtra` e `UpdateExtra` nei passaggi successivi): la convenzione stretta del progetto prevederebbe un `Response` per sottocartella-operazione, ma qui i quattro Response sarebbero **identici carattere per carattere** — Extra ha solo 2 campi visibili, non c'è nulla in più da mostrare in una vista "dettaglio" che già non sia nella lista. La duplicazione non comprerebbe niente; il file vive nella **radice** di `Features/Extras/` (non dentro una sottocartella operazione) proprio per non suggerire un'appartenenza che non ha. Se in futuro un'operazione avesse bisogno di campi propri, si creerebbe il suo Response dedicato allora.
- `GetExtrasQuery` non ha parametri (né filtri, né paginazione): coerente con la scelta del Passaggio 2 di non paginare — è un `record` "vuoto" solo per rispettare l'interfaccia `IRequest<T>` di MediatR, che richiede comunque un oggetto messaggio anche quando non porta dati.
- `ObjectId.TryParse` + `NotFoundException` in `GetExtraByIdHandler`: un id malformato (es. un id di un'altra entità copiato per errore nell'URL) deve dare **404**, non **500** — se si passasse la stringa non valida direttamente a un filtro Mongo si otterrebbe un'eccezione di parsing non gestita, che `ExceptionHandlingMiddleware` mapperebbe a 500 generico. Il controllo esplicito qui è ciò che rende l'errore corretto e comprensibile lato frontend.

## Passaggio 4 — Command Create

**Files nuovi**: `Features/Extras/CreateExtra/`

```csharp
// 📄 CreateExtraCommand.cs
public record CreateExtraCommand(string Description, decimal Cost) : IRequest<ExtraResponse>;

// 📄 CreateExtraValidator.cs
public class CreateExtraValidator : AbstractValidator<CreateExtraCommand>
{
    public CreateExtraValidator()
    {
        RuleFor(x => x.Description).NotEmpty().MaximumLength(200);
        RuleFor(x => x.Cost).GreaterThanOrEqualTo(0);
    }
}

// 📄 CreateExtraHandler.cs
public class CreateExtraHandler(IExtraRepository repo) : IRequestHandler<CreateExtraCommand, ExtraResponse>
{
    public async Task<ExtraResponse> Handle(CreateExtraCommand request, CancellationToken ct)
    {
        var now = DateTime.UtcNow;
        var doc = new ExtraDocument
        {
            Id = ObjectId.GenerateNewId(),
            Description = request.Description,
            Cost = request.Cost,
            CreatedAt = now,
            UpdatedAt = now
        };

        await repo.InsertAsync(doc, ct);

        return new ExtraResponse(doc.Id.ToString(), doc.Description, doc.Cost);
    }
}
```

**Perché è fatto così**

- `RuleFor(x => x.Cost).GreaterThanOrEqualTo(0)`: `decimal` di per sé accetta valori negativi, ma un costo negativo non ha senso di dominio — la regola non è nel tipo (`decimal` resta generico), è nel Validator, dove appartengono le regole di business specifiche di questa Command (stesso principio spiegato per `ReservationValidator` nel documento generico di riferimento, [../generic/features/reservations-implementation.md](../generic/features/reservations-implementation.md)).
- Nessuna scrittura di `AuditLogDocument` in questa versione base: a differenza di `CreateCustomerHandler`, che la scrive sempre, per un dominio "di configurazione" come Extra il tracciamento è opzionale — va aggiunto solo se il progetto decide che serve sapere chi ha cambiato un costo. Il livello 2 la segnala come possibile estensione, non come passaggio obbligatorio. Se si decide di tracciare, il blocco da aggiungere in coda all'Handler (prima del `return`) è lo stesso di `CreateCustomerHandler` in [../technical/backend-03-dettaglio.md](../technical/backend-03-dettaglio.md): `IAuditLogRepository` e `IHttpContextAccessor` in più nel costruttore, e un `auditRepo.InsertAsync(new AuditLogDocument { EntityType = "Extra", Action = "Created", After = doc.ToBsonDocument(), ... })` con attore letto dai claim del JWT.
- Il Validator **non** viene richiamato esplicitamente da nessuna parte nell'Handler: viene eseguito automaticamente da `ValidationBehavior` (pipeline MediatR) prima che `Handle()` sia mai chiamato — se la validazione fallisce, questo codice non viene proprio eseguito. Approfondito in [../architecture/backend-03-nel-progetto.md](../architecture/backend-03-nel-progetto.md), sezione "Pipeline behavior".

## Passaggio 5 — Command Update e Delete

**Files nuovi**: `Features/Extras/UpdateExtra/` e `Features/Extras/DeleteExtra/`

```csharp
// 📄 UpdateExtra/UpdateExtraCommand.cs
public record UpdateExtraCommand(string Id, string Description, decimal Cost) : IRequest<ExtraResponse>;

// 📄 UpdateExtra/UpdateExtraValidator.cs
public class UpdateExtraValidator : AbstractValidator<UpdateExtraCommand>
{
    public UpdateExtraValidator()
    {
        RuleFor(x => x.Id).NotEmpty();
        RuleFor(x => x.Description).NotEmpty().MaximumLength(200);
        RuleFor(x => x.Cost).GreaterThanOrEqualTo(0);
    }
}

// 📄 UpdateExtra/UpdateExtraHandler.cs
public class UpdateExtraHandler(IExtraRepository repo) : IRequestHandler<UpdateExtraCommand, ExtraResponse>
{
    public async Task<ExtraResponse> Handle(UpdateExtraCommand request, CancellationToken ct)
    {
        if (!ObjectId.TryParse(request.Id, out var id))
            throw new NotFoundException($"Extra {request.Id} non trovato");

        var existing = await repo.GetByIdAsync(id, ct)
            ?? throw new NotFoundException($"Extra {request.Id} non trovato");

        existing.Description = request.Description;
        existing.Cost = request.Cost;
        existing.UpdatedAt = DateTime.UtcNow;

        await repo.UpdateAsync(existing, ct);

        return new ExtraResponse(existing.Id.ToString(), existing.Description, existing.Cost);
    }
}

// 📄 DeleteExtra/DeleteExtraCommand.cs
public record DeleteExtraCommand(string Id) : IRequest;

// 📄 DeleteExtra/DeleteExtraHandler.cs
public class DeleteExtraHandler(IExtraRepository repo) : IRequestHandler<DeleteExtraCommand>
{
    public async Task Handle(DeleteExtraCommand request, CancellationToken ct)
    {
        if (!ObjectId.TryParse(request.Id, out var id))
            throw new NotFoundException($"Extra {request.Id} non trovato");

        var existing = await repo.GetByIdAsync(id, ct)
            ?? throw new NotFoundException($"Extra {request.Id} non trovato");

        await repo.DeleteAsync(existing.Id, ct);
    }
}
```

**Perché è fatto così**

- `UpdateExtraHandler` modifica `existing` (il documento appena letto) invece di costruire un `ExtraDocument` nuovo da zero: a differenza dell'esempio generico con record immutabili C#, qui `ExtraDocument` è una `class` mutabile (coerente con l'uso di MongoDB.Driver visto in `CustomerDocument`), quindi si può aggiornare in place — più corto, stesso risultato, e `ReplaceOneAsync` sovrascrive comunque l'intero documento lato database.
- `DeleteExtraHandler` **legge prima di eliminare** anche se non userebbe il valore di `existing` per altro: è così che si ottiene un **404** pulito se l'id non esiste, invece di una `DeleteOneAsync` "silenziosa" che non solleva errore se non trova nulla da cancellare (comportamento di default dei driver Mongo) — senza questo controllo, cancellare due volte lo stesso id restituirebbe sempre "200 OK" anche la seconda volta, quando in realtà non è stato cancellato nulla.
- Nessuna verifica di "referenze in uso" prima di eliminare (es. "questo extra è usato in un preventivo, impedisci la cancellazione"): dato che nell'esempio didattico "PLUS"/"EXTRA" non è ancora collegato a nessun'altra entità (vedi Passaggio 1), non c'è nulla da controllare. Se in futuro un preventivo referenziasse gli extra, andrebbe applicato lo stesso trade-off già documentato per `DeleteCustomerHandler` in [../architecture/backend-03-nel-progetto.md](../architecture/backend-03-nel-progetto.md): Mongo non ha integrità referenziale automatica, la decisione (bloccare, o permettere e lasciare un riferimento "orfano") è responsabilità del codice applicativo.

## Passaggio 6 — Endpoints

**File nuovo**: `Features/Extras/ExtraEndpoints.cs`

```csharp
public static class ExtraEndpoints
{
    public static IEndpointRouteBuilder MapExtraEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/extras").WithTags("Extras").RequireAuthorization();

        group.MapGet("/", async (IMediator mediator, CancellationToken ct) =>
            Results.Ok(await mediator.Send(new GetExtrasQuery(), ct)))
            .RequireAuthorization(p => p.RequireClaim("permissions", "extras.read"))
            .WithName("GetExtras");

        group.MapGet("/{id}", async (string id, IMediator mediator, CancellationToken ct) =>
            Results.Ok(await mediator.Send(new GetExtraByIdQuery(id), ct)))
            .RequireAuthorization(p => p.RequireClaim("permissions", "extras.read"))
            .WithName("GetExtraById");

        group.MapPost("/", async (CreateExtraCommand cmd, IMediator mediator, CancellationToken ct) =>
        {
            var result = await mediator.Send(cmd, ct);
            return Results.Created($"/api/extras/{result.Id}", result);
        })
            .RequireAuthorization(p => p.RequireClaim("permissions", "extras.write"))
            .WithName("CreateExtra")
            .WithSummary("Crea un nuovo extra")
            .Produces<ExtraResponse>(StatusCodes.Status201Created)
            .ProducesProblem(StatusCodes.Status400BadRequest);

        group.MapPut("/{id}", async (string id, UpdateExtraCommand cmd, IMediator mediator, CancellationToken ct) =>
        {
            if (cmd.Id != id) return Results.BadRequest("L'id nel body non corrisponde all'id nella URL");
            return Results.Ok(await mediator.Send(cmd, ct));
        })
            .RequireAuthorization(p => p.RequireClaim("permissions", "extras.write"))
            .WithName("UpdateExtra");

        group.MapDelete("/{id}", async (string id, IMediator mediator, CancellationToken ct) =>
        {
            await mediator.Send(new DeleteExtraCommand(id), ct);
            return Results.NoContent();
        })
            .RequireAuthorization(p => p.RequireClaim("permissions", "extras.delete"))
            .WithName("DeleteExtra");

        return app;
    }
}
```

**Registrazione — MODIFICA obbligatoria**, `Features/_Shared/Extensions/EndpointExtensions.cs::MapAllEndpoints()`:

```csharp
app.MapExtraEndpoints();
```

> ⚠️ Senza questa riga, tutto il codice dei passaggi 1-6 compila ma **l'endpoint non esiste a runtime** — nessun errore visibile finché non si prova a chiamarlo (404 generico di ASP.NET, non un errore che indichi la causa reale). È l'errore più facile da fare in tutta questa guida, segnalato esplicitamente anche in [../technical/backend-03-dettaglio.md](../technical/backend-03-dettaglio.md).

**Perché è fatto così**

- `RequireAuthorization()` sul gruppo (senza argomenti) più `RequireAuthorization(p => ...)` per singolo endpoint: il primo garantisce "serve un JWT valido qualunque esso sia", il secondo aggiunge "e deve avere questo specifico permesso" — sono due controlli distinti, il secondo non sostituisce il primo.
- `GET` usa `extras.read` per **entrambi** gli endpoint di lettura (lista e dettaglio): un solo permesso per entrambe le query, coerente con la scelta fatta anche per `write` (create+update). Il progetto non separa i permessi più finemente di `read`/`write`/`delete` per nessuna entità esistente.
- Il backend **è la vera barriera di sicurezza**: anche se il frontend nasconde bottoni con `v-if="can(...)"`, un client HTTP diverso dal frontend ufficiale (Postman, uno script) che avesse un JWT valido ma senza il claim giusto riceverebbe comunque **403 Forbidden** da questo controllo lato server — la UI che nasconde è solo comodità, non sicurezza (difesa in profondità, già discussa in [../technical/frontend-03-dettaglio.md](../technical/frontend-03-dettaglio.md)).
- `.WithSummary`/`.Produces`/`.ProducesProblem` (mostrati sul POST, da replicare sugli altri endpoint): non cambiano il comportamento a runtime — alimentano lo schema OpenAPI da cui Scalar e Swagger generano la documentazione interattiva, cioè esattamente lo strumento usato per la verifica intermedia descritta più sotto. Ometterli funziona, ma la documentazione API risulterebbe muta su cosa risponde l'endpoint.

## Seed dei dati di esempio (opzionale, solo per sviluppo/demo)

Se si vuole popolare il database con le 5 righe di esempio di questa guida al primo avvio (stesso meccanismo con cui vengono seminati i template `.docx`, vedi [../technical/backend-03-dettaglio.md](../technical/backend-03-dettaglio.md)), si aggiunge un blocco nel `DataSeeder` esistente:

```csharp
var existing = await extraRepo.GetAllAsync(ct);
if (existing.Count == 0)
{
    var now = DateTime.UtcNow;
    var seedExtras = new[]
    {
        ("Sopralluogo aggiuntivo", 50.00m),
        ("Rilievo misure fuori zona", 80.00m),
        ("Trasporto entro 30 km", 40.00m),
        ("Pratica detrazione fiscale", 120.00m),
        ("Certificazione posa", 80.00m),
    };
    foreach (var (description, cost) in seedExtras)
    {
        await extraRepo.InsertAsync(new ExtraDocument
        {
            Id = ObjectId.GenerateNewId(), Description = description, Cost = cost,
            CreatedAt = now, UpdatedAt = now
        }, ct);
    }
}
```

Il controllo "collection vuota" prima di seminare è lo stesso pattern usato per i template: il seed **non deve mai duplicare** dati a ogni riavvio dell'applicazione.

## Verifica intermedia: testare il backend da solo

Prima di scrivere una riga di frontend, conviene verificare i passaggi 1-6 da Scalar (`/scalar`) o Swagger UI (`/swagger`) con l'app avviata in locale:

1. **Seminare e assegnare i permessi** `extras.read`/`extras.write`/`extras.delete` (vedi [02-nel-progetto](02-nel-progetto.md) per il come).
2. **Fare login** con un utente che ha quei permessi e usare il token come Bearer in Scalar/Swagger (bottone "Authorize"). Attenzione al punto più disorientante di tutta la guida: i permessi vivono dentro il claim `permissions` del JWT, che viene costruito **al momento del login** — un token ottenuto *prima* dell'assegnazione dei permessi non li contiene, e ogni chiamata risponderà 403 anche se nel database è tutto corretto. Se "il codice è giusto e il permesso c'è" ma arriva 403, il token in tasca è vecchio: rifare il login.
3. `GET /api/extras` → **200** con `[]` (o con le 5 righe di esempio, se si è aggiunto il seed).
4. `POST /api/extras` con `{ "description": "Sopralluogo aggiuntivo", "cost": 50.00 }` → **201**; ripetere il GET per rivedere la riga in lista.
5. `POST /api/extras` con `{ "description": "", "cost": -5 }` → **400** con l'oggetto `errors` che elenca entrambe le violazioni: è la conferma che `ValidationBehavior` + `CreateExtraValidator` sono agganciati.
6. `DELETE /api/extras/{id}` con un id ben formato ma inesistente → **404**: è la conferma del controllo esplicito nel `DeleteExtraHandler`.

Se questi punti passano, il backend è finito: ogni problema che emergerà dopo è, per esclusione, nel frontend.

---

# FRONTEND

## Passaggio 7 — Tipi TypeScript

**File esistente — MODIFICA**: `apps/frontend/src/types/api.types.ts` (aggiungi in fondo)

```typescript
export interface Extra {
  id: string
  description: string
  cost: number
}

export interface CreateExtraRequest {
  description: string
  cost: number
}

export interface UpdateExtraRequest {
  description: string
  cost: number
}
```

**Perché è fatto così**

- `cost: number` in TypeScript, non un tipo "decimale" separato: JSON non ha un tipo decimale nativo, il `decimal` C# viene serializzato come numero e ricevuto come `number` — la precisione che `decimal` garantisce lato server (niente errori di arrotondamento nel calcolo/salvataggio) non ha un vero equivalente lato client, dove il valore viaggia com'è già "risolto" dal backend. Non c'è nulla da fare qui, è solo da sapere per non cercare un tipo che non esiste.
- `CreateExtraRequest` e `UpdateExtraRequest` sono identici in questo caso (entrambi solo `description`+`cost`): restano comunque due interfacce separate, non un alias, perché è la convenzione osservata per ogni altra entità (`CreateCustomerRequest`/`UpdateCustomerRequest` sono anch'essi tenuti distinti anche quando si sovrappongono) — se in futuro Update acquisisse un campo in più che Create non ha (successo già visto con `isActive` per gli utenti), la separazione è già pronta.

## Passaggio 8 — Service

**File nuovo**: `apps/frontend/src/services/extras.service.ts`

```typescript
import { api } from '@/plugins/axios'
import type { Extra, CreateExtraRequest, UpdateExtraRequest } from '@/types/api.types'

export const extrasService = {
  getAll: () =>
    api.get<Extra[]>('/api/extras').then(r => r.data),

  getById: (id: string) =>
    api.get<Extra>(`/api/extras/${id}`).then(r => r.data),

  create: (data: CreateExtraRequest) =>
    api.post<Extra>('/api/extras', data).then(r => r.data),

  update: (id: string, data: UpdateExtraRequest) =>
    api.put<Extra>(`/api/extras/${id}`, data).then(r => r.data),

  delete: (id: string) =>
    api.delete(`/api/extras/${id}`),
}
```

**Perché è fatto così**

- `getAll()` ritorna direttamente `Extra[]`, non un oggetto paginato (`{ items, totalCount }`) come `customersService.getAll()`: coerente con la scelta di non paginare fatta a livello di backend (Passaggio 2/3) — il service rispecchia esattamente la forma della risposta reale dell'endpoint, non ne inventa una diversa.
- Nessuna chiamata `axios` diretta: importa sempre `api` da `plugins/axios.ts`, mai `import axios from 'axios'` — è la regola non negoziabile del `CLAUDE.md` frontend citata in [../architecture/frontend-03-nel-progetto.md](../architecture/frontend-03-nel-progetto.md), che garantisce che ogni chiamata porti automaticamente il Bearer token.

## Passaggio 9 — Store Pinia

**File nuovo**: `apps/frontend/src/stores/extras.store.ts`

```typescript
import { defineStore } from 'pinia'
import { ref } from 'vue'
import { extrasService } from '@/services/extras.service'
import type { Extra, CreateExtraRequest, UpdateExtraRequest } from '@/types/api.types'
import i18n from '@/plugins/i18n'

export const useExtrasStore = defineStore('extras', () => {
  const items = ref<Extra[]>([])
  const selectedItem = ref<Extra | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)

  async function fetchAll() {
    loading.value = true
    error.value = null
    try {
      items.value = await extrasService.getAll()
    } catch {
      error.value = i18n.global.t('errors.loadExtras')
    } finally {
      loading.value = false
    }
  }

  async function create(data: CreateExtraRequest) {
    await extrasService.create(data)
    await fetchAll()
  }

  async function update(id: string, data: UpdateExtraRequest) {
    await extrasService.update(id, data)
    await fetchAll()
  }

  async function remove(id: string) {
    await extrasService.delete(id)
    items.value = items.value.filter(x => x.id !== id)
  }

  return { items, selectedItem, loading, error, fetchAll, create, update, remove }
})
```

**Perché è fatto così**

- Niente `page`/`pageSize`/`search`/`totalCount` nello state, a differenza dello scheletro standard descritto in [../technical/frontend-03-dettaglio.md](../technical/frontend-03-dettaglio.md) punto 2: quei campi esistono nel pattern generico perché quasi tutte le entità di Tama sono paginate lato server, ma Extra non lo è (Passaggio 2). Copiare comunque quei campi "per abitudine" da `customers.store.ts` sarebbe un errore: introdurrebbero stato morto, mai letto da nessun componente.
- `create`/`update` rifanno `fetchAll()` (round-trip in più ma stato sempre coerente col server), `remove` aggiorna `items` localmente senza rifetch: stessa doppia strategia già documentata per `customers.store.ts` in [../architecture/frontend-03-nel-progetto.md](../architecture/frontend-03-nel-progetto.md) — per una lista corta come Extra, il costo del round-trip extra su create/update è trascurabile, quindi non vale la pena dell'update ottimistico più complesso lì dove basta rileggere tutto.
- `selectedItem` è presente anche se, con la pagina lista + dialog descritta al Passaggio 11 (non una pagina di dettaglio separata come `UserDetailPage.vue`), il suo uso è minimo: serve solo a passare l'item corrente al dialog di modifica. Resta comunque nello store (non uno stato locale del componente) per coerenza con lo schema standard e perché è comunque dato "di dominio", non UI-only.

## Passaggio 10 — Pagina lista

**File nuovo**: `apps/frontend/src/pages/plus/ExtrasPage.vue`

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useI18n } from 'vue-i18n'
import { useDisplay } from 'vuetify'
import { useExtrasStore } from '@/stores/extras.store'
import { usePermission } from '@/composables/usePermission'
import { useToastStore } from '@/stores/toast.store'
import ExtraFormDialog from '@/components/plus/ExtraFormDialog.vue'
import type { Extra } from '@/types/api.types'

const { t, locale } = useI18n()
const { mobile } = useDisplay()
const store = useExtrasStore()
const { can } = usePermission()
const toast = useToastStore()

const search = ref('')
const formDialog = ref(false)
const editingExtra = ref<Extra | null>(null)
const deleteDialog = ref(false)
const deleteTarget = ref<string | null>(null)
const deleteLoading = ref(false)

const filtered = computed(() => {
  if (!search.value) return store.items
  const s = search.value.toLowerCase()
  return store.items.filter(x => x.description.toLowerCase().includes(s))
})

const headers = computed(() => [
  { title: t('extras.columnDescription'), key: 'description' },
  { title: t('extras.columnCost'), key: 'cost' },
  { title: t('common.actions'), key: 'actions', sortable: false },
])

function formatCost(value: number) {
  return new Intl.NumberFormat(locale.value, { style: 'currency', currency: 'EUR' }).format(value)
}

function openCreateDialog() {
  editingExtra.value = null
  formDialog.value = true
}

function openEditDialog(extra: Extra) {
  editingExtra.value = extra
  formDialog.value = true
}

function openDeleteDialog(id: string) {
  deleteTarget.value = id
  deleteDialog.value = true
}

async function confirmDelete() {
  if (!deleteTarget.value) return
  deleteLoading.value = true
  try {
    await store.remove(deleteTarget.value)
    toast.success(t('extras.deleted'))
    deleteDialog.value = false
  } catch {
    toast.error(t('common.error'))
  } finally {
    deleteLoading.value = false
  }
}

onMounted(() => store.fetchAll())
</script>

<template>
  <v-container>
    <div class="d-flex justify-space-between align-center mb-4">
      <h1>{{ t('extras.title') }}</h1>
      <v-btn v-if="can('extras.write')" color="primary" @click="openCreateDialog">
        {{ t('common.new') }}
      </v-btn>
    </div>

    <v-text-field v-model="search" :label="t('common.search')" prepend-inner-icon="mdi-magnify" class="mb-4" />

    <v-data-table v-if="!mobile" :headers="headers" :items="filtered" :loading="store.loading" item-value="id">
      <template #item.cost="{ item }">{{ formatCost(item.cost) }}</template>
      <template #item.actions="{ item }">
        <v-btn v-if="can('extras.write')" icon="mdi-pencil" variant="text" @click="openEditDialog(item)" />
        <v-btn v-if="can('extras.delete')" icon="mdi-delete" variant="text" @click="openDeleteDialog(item.id)" />
      </template>
    </v-data-table>

    <v-list v-else>
      <v-list-item v-for="item in filtered" :key="item.id" :title="item.description" :subtitle="formatCost(item.cost)">
        <template #append>
          <v-btn v-if="can('extras.write')" icon="mdi-pencil" variant="text" @click="openEditDialog(item)" />
          <v-btn v-if="can('extras.delete')" icon="mdi-delete" variant="text" @click="openDeleteDialog(item.id)" />
        </template>
      </v-list-item>
    </v-list>

    <ExtraFormDialog v-model="formDialog" :extra="editingExtra" />

    <v-dialog v-model="deleteDialog" max-width="420">
      <v-card>
        <v-card-title>{{ t('extras.confirmDeleteTitle') }}</v-card-title>
        <v-card-actions>
          <v-spacer />
          <v-btn @click="deleteDialog = false">{{ t('common.cancel') }}</v-btn>
          <v-btn color="error" :loading="deleteLoading" @click="confirmDelete">{{ t('common.delete') }}</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>
</template>
```

**Perché è fatto così**

- `v-data-table` semplice, non `v-data-table-server`: quest'ultimo esiste in `CustomersPage.vue` perché la paginazione/ricerca/ordinamento avvengono **lato server** (parametri `page`/`search` inviati nella richiesta). Qui la lista intera è già in memoria nello store (Passaggio 9 non pagina), quindi filtro e tabella lavorano **lato client** su `filtered` — usare `v-data-table-server` qui sarebbe sbagliato: non ci sono parametri server da passargli.
- `filtered` come `computed` che legge `store.items` e `search.value`, non una copia locale filtrata "a mano" con un watcher: si ricalcola automaticamente ogni volta che uno dei due cambia, senza codice esplicito di sincronizzazione — stesso principio di reattività "single source of truth" descritto in [../architecture/frontend-03-nel-progetto.md](../architecture/frontend-03-nel-progetto.md).
- Pattern responsive duplicato (`v-data-table` desktop + `v-list` mobile, scelto in base a `mobile` da `useDisplay()`): non è una scelta di questo documento, è la convenzione osservata in ogni pagina lista del progetto — vedi la stessa nota in [../technical/frontend-03-dettaglio.md](../technical/frontend-03-dettaglio.md).
- `formatCost` con `Intl.NumberFormat` legato a `locale` di vue-i18n, non `item.cost.toFixed(2)`: `toFixed` produce sempre il punto decimale (`50.00`) qualunque sia la lingua, mentre in italiano il costo va mostrato come `50,00 €` — `Intl.NumberFormat` (API standard del browser, nessuna libreria da aggiungere) applica separatori e simbolo di valuta corretti per la lingua attiva. E siccome `locale` è un ref reattivo, cambiando lingua dall'app la colonna si riformatta da sola.
- **Nessun testo hardcoded**: ogni etichetta passa da `t('...')`. Le chiavi usate qui (`extras.title`, `extras.columnDescription`, ecc.) sono definite al Passaggio 12.

## Passaggio 11 — Dialog di creazione/modifica

**File nuovo**: `apps/frontend/src/components/plus/ExtraFormDialog.vue`

```vue
<script setup lang="ts">
import { ref, reactive, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { useExtrasStore } from '@/stores/extras.store'
import { useToastStore } from '@/stores/toast.store'
import type { Extra } from '@/types/api.types'

const props = defineProps<{ modelValue: boolean; extra: Extra | null }>()
const emit = defineEmits<{ 'update:modelValue': [boolean] }>()

const { t } = useI18n()
const store = useExtrasStore()
const toast = useToastStore()

const formRef = ref()
const form = reactive({ description: '', cost: 0 })
const saving = ref(false)

const rules = {
  required: (v: string) => !!v?.trim() || t('validation.required'),
  requiredNumber: (v: number | string) => (v !== '' && v !== null) || t('validation.required'),
  positive: (v: number) => v >= 0 || t('validation.positive'),
}

watch(() => props.modelValue, (open) => {
  if (open) {
    form.description = props.extra?.description ?? ''
    form.cost = props.extra?.cost ?? 0
  }
})

async function save() {
  const { valid } = await formRef.value.validate()
  if (!valid) return

  saving.value = true
  try {
    if (props.extra) await store.update(props.extra.id, { ...form })
    else await store.create({ ...form })
    toast.success(t('common.saved'))
    emit('update:modelValue', false)
  } catch {
    toast.error(t('common.error'))
  } finally {
    saving.value = false
  }
}
</script>

<template>
  <v-dialog :model-value="modelValue" max-width="480" @update:model-value="emit('update:modelValue', $event)">
    <v-card>
      <v-card-title>{{ extra ? t('extras.editTitle') : t('extras.createTitle') }}</v-card-title>
      <v-card-text>
        <v-form ref="formRef">
          <v-text-field v-model="form.description" :label="t('extras.columnDescription')" :rules="[rules.required]" />
          <v-text-field v-model.number="form.cost" type="number" step="0.01" :label="t('extras.columnCost')" :rules="[rules.requiredNumber, rules.positive]" />
        </v-form>
      </v-card-text>
      <v-card-actions>
        <v-spacer />
        <v-btn @click="emit('update:modelValue', false)">{{ t('common.cancel') }}</v-btn>
        <v-btn color="primary" :loading="saving" @click="save">{{ t('common.save') }}</v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>
```

**Perché è fatto così**

- Un solo dialog per create **e** edit, distinto dalla prop `extra` (`null` = creazione, oggetto = modifica): stesso principio del pattern "stesso componente per create/edit" già visto in `UserDetailPage.vue` ([../technical/frontend-03-dettaglio.md](../technical/frontend-03-dettaglio.md)), applicato qui a un **dialog** invece che a una pagina intera — scelta corretta per un'entità con soli due campi, dove aprire una pagina dedicata sarebbe sproporzionato (è la stessa distinzione fatta nella skill di progetto `vuetify-dialog-form` citata in quel documento).
- `watch(() => props.modelValue, ...)` ripopola il form **ogni volta che il dialog si apre**, non una volta sola al mount: il componente dialog resta montato tra un'apertura e l'altra (è dentro `ExtrasPage.vue`, non instanziato/distrutto ogni volta), quindi senza questo watcher aprire "Modifica" su un secondo extra dopo aver già modificato il primo mostrerebbe ancora i valori vecchi.
- `v-model.number` su `form.cost`: senza `.number`, `v-model` su un `<input type="number">` restituirebbe comunque una stringa (`"50.00"` invece di `50.00`), che poi verrebbe inviata così al backend — .NET la deserializzerebbe comunque in `decimal` (conversione implicita nel binding JSON), ma è più corretto e prevedibile avere già un `number` lato TypeScript prima di spedirlo.
- `formRef.value.validate()` **prima** di procedere, esattamente come in `UserDetailPage.vue`: non ci si affida alle sole regole inline `:rules`, che vengono valutate solo alla perdita di focus di un campo — un utente che compila il form e preme subito "Salva" senza mai uscire dall'ultimo campo bypasserebbe la validazione visiva se non ci fosse questa chiamata esplicita.
- `requiredNumber` separata da `required`, per il campo costo: con `v-model.number`, un campo numerico **svuotato** dall'utente non vale `0` ma **stringa vuota** (è ciò che il modificatore `.number` restituisce quando l'input non è interpretabile come numero). E poiché in JavaScript `'' >= 0` è `true` (coercizione implicita), la sola regola `positive` lascerebbe passare il campo vuoto — che arriverebbe al backend come `""` facendo fallire la deserializzazione JSON a `decimal`, con un 400 generico poco leggibile per l'utente. Il controllo esplicito `v !== ''` intercetta il caso a monte, con il messaggio di validazione giusto sul campo giusto.

## Passaggio 12 — Route, sezione di menu, traduzioni

**MODIFICA — `apps/frontend/src/router/index.ts`** (aggiungi tra le altre route):

```typescript
{
  path: '/plus/extras',
  name: 'extras',
  component: () => import('@/pages/plus/ExtrasPage.vue'),
  meta: { requiresAuth: true, permission: 'extras.read', title: 'extras.title', section: 'plus' },
},
```

**MODIFICA — `apps/frontend/src/config/sections.config.ts`** (nuova sezione, allo stesso livello di quella "DATI" esistente):

```typescript
{
  key: 'plus',
  labelKey: 'sections.plus',
  icon: 'mdi-plus-box-outline',
  items: [
    { routeName: 'extras', labelKey: 'extras.title', permission: 'extras.read' },
  ],
},
```

**MODIFICA — `apps/frontend/src/locales/it.ts`**:

```typescript
sections: {
  // ... voci esistenti
  plus: 'Plus',
},
extras: {
  title: 'Extra',
  columnDescription: 'Descrizione extra',
  columnCost: 'Costo',
  createTitle: 'Nuovo extra',
  editTitle: 'Modifica extra',
  confirmDeleteTitle: 'Eliminare questo extra?',
  deleted: 'Extra eliminato',
},
errors: {
  // ... voci esistenti
  loadExtras: 'Errore nel caricamento degli extra',
},
validation: {
  // ... voci esistenti (required/positive probabilmente già presenti, altrimenti aggiungerle)
  positive: 'Il valore deve essere maggiore o uguale a zero',
},
```

**MODIFICA — `apps/frontend/src/locales/en.ts`** (stesse chiavi, valori inglesi):

```typescript
sections: {
  plus: 'Plus',
},
extras: {
  title: 'Extras',
  columnDescription: 'Extra description',
  columnCost: 'Cost',
  createTitle: 'New extra',
  editTitle: 'Edit extra',
  confirmDeleteTitle: 'Delete this extra?',
  deleted: 'Extra deleted',
},
errors: {
  loadExtras: 'Error loading extras',
},
validation: {
  positive: 'Value must be greater than or equal to zero',
},
```

**Perché è fatto così**

- `meta.title: 'extras.title'` è una **chiave** di traduzione, non il testo "Extra" scritto direttamente: il componente che mostra il titolo di pagina/breadcrumb la passa a `t(...)`, così il titolo cambia lingua insieme al resto dell'app senza codice aggiuntivo.
- `meta.section: 'plus'` deve combaciare esattamente con `key: 'plus'` nella configurazione del menu: è così che il menu laterale sa quale sezione evidenziare come "attiva" quando l'utente si trova sulla pagina `extras` — un valore diverso (es. un refuso `'plu'`) non darebbe errori a compile time (sono entrambe stringhe libere) ma romperebbe silenziosamente l'evidenziazione, un bug tipicamente diagnosticato con `devtools-router.md`.
- Sia `router/index.ts` (`meta.permission`) sia `sections.config.ts` (`items[].permission`) portano **lo stesso** valore `'extras.read'`: sono due controlli con lo stesso permesso ma in due punti diversi (uno impedisce di navigare alla rotta se non autorizzati, l'altro nasconde la voce di menu) — servono entrambi, uno non sostituisce l'altro (un utente potrebbe digitare l'URL a mano bypassando il menu).
- Le chiavi `it.ts`/`en.ts` vanno aggiunte **in coppia**, nello stesso commit: il progetto non ha un fallback silenzioso pulito per chiavi mancanti in una sola lingua (tipicamente `vue-i18n` stampa la chiave grezza a schermo, es. `extras.title` letterale, se manca la traduzione per la lingua attiva) — è un bug visibile solo cambiando lingua, quindi facile da dimenticare se si testa sempre in italiano.
- Le chiavi `common.*` e `validation.required` usate nella pagina e nel dialog (`common.new`, `common.search`, `common.actions`, `common.cancel`, `common.delete`, `common.save`, `common.saved`, `common.error`) **esistono già** in entrambi i locale, usate da ogni altra pagina del progetto: non vanno ricreate. Prima di aggiungere una chiave "generica" nuova, verificare sempre che non esista già — una chiave duplicata nello stesso file non dà errori di compilazione, semplicemente una definizione sovrascrive l'altra in silenzio.

---

## Verifica finale end-to-end

Con tutti e 12 i passaggi completati, il flusso di creazione di un extra attraversa questi punti, nell'ordine — utile per isolare dove cercare se qualcosa non funziona:

```
Click "Nuovo" (ExtrasPage.vue)
        ↓
ExtraFormDialog si apre, utente compila e clicca "Salva"
        ↓
formRef.validate() → store.create({ description, cost })
        ↓
extras.store.ts → extrasService.create(data)
        ↓
services/extras.service.ts → api.post('/api/extras', data)   [axios, JWT allegato dall'interceptor]
        ↓
POST /api/extras   → ExtraEndpoints.cs
        ↓
RequireClaim("permissions", "extras.write")   → 403 se assente
        ↓
CreateExtraCommand → ValidationBehavior → CreateExtraValidator
        ↓
CreateExtraHandler → repo.InsertAsync → MongoDB collection "extras"
        ↓
Results.Created(...) → 201 + ExtraResponse
        ↓
store.create() → await fetchAll()   → GET /api/extras rilegge la lista intera
        ↓
store.items aggiornato → ExtrasPage.vue si ridisegna da sola (reattività Pinia)
```

Se un passaggio della guida è stato saltato, questo è tipicamente il flusso in cui si manifesta l'errore: endpoint non registrato → 404 al passo `POST /api/extras`; permesso non seminato/assegnato (o assegnato ma **senza rifare il login**: il vecchio JWT non lo contiene) → 403 subito dopo; validator con regola dimenticata → 400 con `errors` popolato ma nessun record salvato; chiave i18n mancante → nessun errore, ma testo grezzo `extras.xxx` visibile a schermo.
