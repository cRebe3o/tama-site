# Concetti architetturali — Backend — Livello 3: dove si vedono nel codice di Tama

Questo documento ancora ogni concetto spiegato nei livelli precedenti a un file/snippet reale del repository. Per la spiegazione teorica, vedi [backend-02-confronto](backend-02-confronto.md); per il dettaglio implementativo completo, vedi [../technical/backend-03-dettaglio.md](../technical/backend-03-dettaglio.md).

## Minimal API

File: `apps/backend/Tama.Api/Features/Customers/CustomerEndpoints.cs`
```csharp
var group = app.MapGroup("/api/customers").WithTags("Customers").RequireAuthorization();
group.MapPost("/", async (CreateCustomerCommand cmd, IMediator mediator, CancellationToken ct) => { ... })
     .RequireAuthorization(p => p.RequireClaim("permissions", "customers.write"));
```
Nessun `[ApiController]`, nessuna classe `CustomersController`. La registrazione dell'intera "famiglia" di endpoint (`GET/POST/PUT/DELETE /api/customers`) è tutta in un unico metodo di estensione. Il fatto che tutti gli endpoint di `MapAllEndpoints()` (`apps/backend/Tama.Api/Features/_Shared/Extensions/EndpointExtensions.cs`) siano registrati **esplicitamente uno per uno** è la conseguenza diretta di non avere convenzioni automatiche: è il prezzo pagato per la semplicità del modello.

## Vertical Slice Architecture

Confronta la struttura reale:
```
Features/Customers/
├── CustomerEndpoints.cs
├── CreateCustomer/   (Command, Handler, Validator, Response)
├── UpdateCustomer/
├── DeleteCustomer/
├── GetCustomerById/
└── GetCustomers/
```
con come sarebbe in Layered Architecture: `Controllers/CustomersController.cs`, `Services/CustomerService.cs` (con dentro `CreateAsync`, `UpdateAsync`, `DeleteAsync`, `GetByIdAsync`, `GetAllAsync` tutti insieme), `Repositories/CustomerRepository.cs`. In Tama, invece, ogni operazione ha la propria cartella isolata — **non esiste** una classe `CustomerService` con 5 metodi: esistono 5 Handler distinti, ciascuno con una singola responsabilità (Single Responsibility Principle applicato al livello della singola operazione, non della classe "servizio").

Il piccolo strato condiviso `Features/_Shared/` (repository, documenti Mongo, behavior, middleware) è l'unica parte "orizzontale" del progetto — usalo come bussola: se una cosa è specifica di **una** feature, va nella cartella della feature; se serve a **più** feature, va in `_Shared/`.

## CQRS + Mediator

`CreateCustomerCommand` (muta stato) e `GetCustomersQuery` (legge) sono entrambi `IRequest<TResponse>`, ma concettualmente distinti per convenzione di naming (`...Command` vs `...Query`) — nulla lo impone a livello di tipo C#, è una convenzione osservata in ogni feature del progetto.

L'endpoint non conosce mai l'Handler direttamente:
```csharp
var result = await mediator.Send(cmd, ct);   // non "new CreateCustomerHandler(...).Handle(cmd)"
```
`mediator.Send` risolve `CreateCustomerHandler` automaticamente perché implementa `IRequestHandler<CreateCustomerCommand, CreateCustomerResponse>` ed è registrato in DI da `AddMediatR(cfg => cfg.RegisterServicesFromAssemblyContaining<Program>())` (`Features/_Shared/Extensions/MediatRExtensions.cs`).

## Pipeline behavior (Decorator)

```csharp
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
```
Questa riga in `MediatRExtensions.cs` è dove il pattern Decorator viene "assemblato": ogni `Send()` passa attraverso `LoggingBehavior` (esterno) che avvolge `ValidationBehavior` (interno) che avvolge l'Handler. Nessuno dei tre sa dell'esistenza degli altri due — puoi aggiungere un nuovo behavior (es. un futuro `CachingBehavior`) senza toccare nessuna riga degli Handler esistenti: è esattamente il beneficio "open/closed" del pattern.

## Repository pattern (non generico)

`ICustomerRepository` (`Features/_Shared/Repositories/`) espone `GetPagedAsync`, `GetByIdAsync`, `InsertAsync`, `UpdateAsync`, `DeleteAsync` — metodi pensati per come i Customer vengono effettivamente interrogati in questo progetto, non un `IRepository<T>` generico riusato per ogni entità. Ogni feature ha la propria interfaccia repository con la propria "forma".

## Document-oriented modeling

`CustomerDocument` (`Features/_Shared/Documents/`) è una classe POCO semplice, senza attributi `[Key]`/`[ForeignKey]` di EF Core:
```csharp
public class CustomerDocument
{
    public ObjectId Id { get; set; }
    public string Name { get; set; } = string.Empty;
    // ...
}
```
L'esempio migliore di modellazione "a documento" nel progetto è `DraftDocument` (il preventivo), in `Features/_Shared/Documents/DraftDocument.cs` — verificato nel codice:
```csharp
public class DraftDocument
{
    public ObjectId Id { get; set; }
    public ObjectId CustomerId { get; set; }
    public string CustomerName { get; set; } = string.Empty;   // snapshot denormalizzato
    public ObjectId SeriesId { get; set; }
    public string SeriesCode { get; set; } = string.Empty;      // snapshot denormalizzato
    public string SeriesName { get; set; } = string.Empty;      // snapshot denormalizzato
    public List<DraftItemDocument> Items { get; set; } = [];    // righe EMBEDDED, non tabella separata
    // ...
}
```
Qui si vedono **entrambe** le tecniche tipiche di un Document DB:
- **Embedding**: le righe del preventivo (`Items`, `List<DraftItemDocument>`) vivono *dentro* il documento del preventivo — "le cose che si leggono sempre insieme si salvano insieme". Caricare un preventivo con 20 posizioni è una singola query, zero join. In un RDBMS sarebbero una tabella `DraftItems` con FK.
- **Denormalizzazione**: accanto a `CustomerId` (il riferimento) c'è `CustomerName` (una *copia* del nome al momento della creazione); idem `SeriesCode`/`SeriesName` accanto a `SeriesId`, e `WindowType` accanto a `WindowTypeId` nelle righe. Il preventivo mostra così i suoi dati senza fare lookup — ed è anche una scelta di dominio sensata: un preventivo emesso è uno "snapshot storico", non deve cambiare retroattivamente se il cliente viene rinominato.

Il riferimento tra entità diverse (`CustomerId`) resta però un `ObjectId` "morbido": nessun vincolo di integrità referenziale a livello di database. Verificato nel codice: `DeleteCustomerHandler` (`Features/Customers/DeleteCustomer/`) **non controlla** se esistono preventivi collegati prima di eliminare — elimina e basta. I preventivi orfani continuano a funzionare visivamente grazie al `CustomerName` denormalizzato, ma il loro `CustomerId` punta a un documento che non esiste più. È il trade-off documentato in [backend-02](backend-02-confronto.md): con Mongo l'integrità referenziale, se serve, è responsabilità del codice applicativo — e qui (deliberatamente o meno) non è stata implementata. ⚠️ Punto da tenere presente se si aggiunge una feature che naviga da preventivo a cliente.

## Transazioni e cascade delete

`Features/Roles/DeleteRole/DeleteRoleHandler.cs` mostra come il progetto gestisce le eliminazioni con effetti a cascata (un ruolo va rimosso anche da tutti gli utenti e gruppi che lo referenziano):
```csharp
using var session = await mongoClient.StartSessionAsync(cancellationToken: cancellationToken);
//session.StartTransaction();
try
{
    await userRepo.RemoveRoleFromAllUsersAsync(id, session, cancellationToken);   // Fase 2
    await groupRepo.RemoveRoleFromAllGroupsAsync(id, session, cancellationToken); // Fase 3
    await roleRepo.DeleteAsync(id, session, cancellationToken);                   // Fase 4
    //await session.CommitTransactionAsync(cancellationToken);
}
catch
{
    //await session.AbortTransactionAsync(cancellationToken);
    throw;
}
```
Da notare: la sessione viene creata e passata ai repository, ma le chiamate `StartTransaction`/`Commit`/`Abort` sono **commentate**. Il motivo (non documentato nel codice, ma quasi certamente questo): le transazioni multi-documento di MongoDB richiedono un **replica set** — su un'istanza standalone (tipica in sviluppo locale e su hosting economici) lanciano un errore a runtime. Il codice è quindi "pronto" per diventare transazionale scommentando tre righe quando l'infrastruttura lo permette; nel frattempo il cascade è sequenziale: se l'app crashasse tra la Fase 2 e la Fase 4, il DB resterebbe in uno stato intermedio (ruolo rimosso dagli utenti ma ancora esistente). Stesso pattern in `DeleteGroupHandler` e `DeletePermissionHandler`. ⚠️ Non scommentare quelle righe senza prima verificare che il MongoDB di destinazione sia un replica set.

## Counter pattern (progressivi senza IDENTITY)

Il numero preventivo (`1001-A00-2026`) contiene un progressivo per anno (`Sequence`). MongoDB non ha `IDENTITY`/`SEQUENCE` nativi, quindi il progetto usa il counter pattern canonico — `Features/_Shared/Repositories/DraftRepository.cs`:
```csharp
public async Task<int> NextAsync(string key, CancellationToken ct)
{
    var update = Builders<CounterDocument>.Update
        .Inc(x => x.Value, 1)
        .SetOnInsert(x => x.Key, key);
    var opts = new FindOneAndUpdateOptions<CounterDocument>
    {
        IsUpsert = true,
        ReturnDocument = ReturnDocument.After
    };
    var result = await _col.FindOneAndUpdateAsync<CounterDocument>(...);
}
```
`FindOneAndUpdate` con `$inc` è **atomico a livello di singolo documento** (garanzia che MongoDB dà sempre, anche senza transazioni): due richieste concorrenti non otterranno mai lo stesso numero. `IsUpsert = true` crea il contatore al primo uso (es. al primo preventivo di un nuovo anno), `ReturnDocument.After` restituisce il valore già incrementato. È l'equivalente Mongo di `SELECT NEXT VALUE FOR sequence` in SQL Server.

## DI lifetimes: la trappola singleton→scoped, risolta

`Features/_Shared/Middleware/ExceptionHandlingMiddleware.cs` è l'ancoraggio perfetto per il vincolo di captive dependency spiegato in [backend-02](backend-02-confronto.md): il middleware è istanziato una volta sola (di fatto singleton), ma per salvare un `ErrorLogDocument` gli serve `IErrorLogRepository`, che è scoped. Non può riceverlo nel costruttore — riceve invece `IServiceScopeFactory` e crea uno scope usa-e-getta al momento del bisogno:
```csharp
private async Task SaveErrorLogAsync(HttpContext context, Exception exception)
{
    // Crea uno scope per risolvere il repository scoped dal middleware singleton
    using var scope = scopeFactory.CreateScope();
    var repo = scope.ServiceProvider.GetRequiredService<IErrorLogRepository>();
    await repo.InsertAsync(new ErrorLogDocument { ... }, CancellationToken.None);
}
```
Se ti capita di dover usare un repository da un contesto a vita lunga (middleware, background service, seed), questo è il pattern da copiare — iniettare il repository direttamente nel costruttore produrrebbe un errore di validazione DI all'avvio in sviluppo.

## Claims-based authorization

```csharp
.RequireAuthorization(p => p.RequireClaim("permissions", "customers.write"))
```
`"permissions"` è il nome del claim, `"customers.write"` è il valore atteso — popolato nel JWT al login risolvendo i permessi di ruoli e gruppi dell'utente (vedi `Features/Auth/`). Non esiste, nel codice, alcun `[Authorize(Roles = "Admin")]`: cerca `RequireClaim("permissions", ...)` per trovare dove un endpoint è protetto e con quale permesso.
