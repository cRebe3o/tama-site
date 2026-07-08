# Backend — Livello 3: Dettaglio con codice reale

## Esempio end-to-end: creazione di un cliente (`POST /api/customers`)

### 1. Endpoint — `Features/Customers/CustomerEndpoints.cs`

```csharp
public static class CustomerEndpoints
{
    public static IEndpointRouteBuilder MapCustomerEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/customers").WithTags("Customers").RequireAuthorization();

        group.MapPost("/", async (CreateCustomerCommand cmd, IMediator mediator, CancellationToken ct) =>
        {
            var result = await mediator.Send(cmd, ct);
            return Results.Created($"/api/customers/{result.Id}", result);
        })
            .RequireAuthorization(p => p.RequireClaim("permissions", "customers.write"))
            .WithName("CreateCustomer")
            .WithSummary("Crea un nuovo cliente")
            .Produces(StatusCodes.Status201Created)
            .ProducesProblem(StatusCodes.Status400BadRequest);

        return app;
    }
}
```
Il gruppo `/api/customers` richiede autenticazione (`RequireAuthorization()` senza argomenti = JWT valido), e ogni singolo endpoint aggiunge il permesso specifico via `RequireClaim("permissions", "...")`. `CreateCustomerCommand` è bindato **direttamente dal body JSON** — con Minimal API, se il primo parametro è un tipo complesso non è necessario `[FromBody]` esplicito.

### 2. Command — `Features/Customers/CreateCustomer/CreateCustomerCommand.cs`

```csharp
public record CreateCustomerCommand(
    string Name,
    string Email,
    string Phone,
    string Address,
    string? VatOrFiscalCode
) : IRequest<CreateCustomerResponse>;
```
Un `record` immutabile che implementa `IRequest<TResponse>` di MediatR. Fa da DTO di input **e** da payload spedito al mediatore: non c'è un livello di mapping separato tra "richiesta HTTP" e "comando applicativo".

### 3. Validator — `Features/Customers/CreateCustomer/CreateCustomerValidator.cs`

```csharp
public class CreateCustomerValidator : AbstractValidator<CreateCustomerCommand>
{
    public CreateCustomerValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(200);
        RuleFor(x => x.Email).NotEmpty().EmailAddress().MaximumLength(200);
        RuleFor(x => x.Phone).NotEmpty().MaximumLength(50);
        RuleFor(x => x.Address).NotEmpty().MaximumLength(500);
        RuleFor(x => x.VatOrFiscalCode).MaximumLength(32);
    }
}
```
Non viene invocato esplicitamente da nessuna parte: è scoperto per assembly-scan (`AddValidatorsFromAssemblyContaining<Program>()`) ed eseguito automaticamente da `ValidationBehavior` prima che l'Handler venga chiamato. Se una regola fallisce, l'Handler **non viene mai eseguito**.

### 4. Handler — `Features/Customers/CreateCustomer/CreateCustomerHandler.cs`

```csharp
public class CreateCustomerHandler(
    ICustomerRepository repo,
    IAuditLogRepository auditRepo,
    IHttpContextAccessor httpContextAccessor) : IRequestHandler<CreateCustomerCommand, CreateCustomerResponse>
{
    public async Task<CreateCustomerResponse> Handle(CreateCustomerCommand request, CancellationToken cancellationToken)
    {
        var now = DateTime.UtcNow;
        var doc = new CustomerDocument
        {
            Id = ObjectId.GenerateNewId(),
            Name = request.Name,
            Email = request.Email,
            Phone = request.Phone,
            Address = request.Address,
            VatOrFiscalCode = request.VatOrFiscalCode ?? string.Empty,
            CreatedAt = now,
            UpdatedAt = now
        };

        await repo.InsertAsync(doc, cancellationToken);

        var actorId = httpContextAccessor.HttpContext?.User.FindFirstValue(ClaimTypes.NameIdentifier) ?? "system";
        var actorEmail = httpContextAccessor.HttpContext?.User.FindFirstValue(ClaimTypes.Email) ?? "system";
        await auditRepo.InsertAsync(new AuditLogDocument
        {
            Id = ObjectId.GenerateNewId(),
            ActorId = ObjectId.TryParse(actorId, out var aid) ? aid : ObjectId.Empty,
            ActorEmail = actorEmail,
            EntityType = "Customer",
            EntityId = doc.Id,
            Action = "Created",
            After = doc.ToBsonDocument(),
            Timestamp = now,
            IpAddress = httpContextAccessor.HttpContext?.Connection.RemoteIpAddress?.ToString(),
        }, cancellationToken);

        return new CreateCustomerResponse(doc.Id.ToString());
    }
}
```
Punti da notare per chi scrive un nuovo Handler:
- **Mapping manuale** `Command → Document`: nessun AutoMapper.
- `ObjectId.GenerateNewId()` genera l'id lato applicazione (non delegato a Mongo).
- L'attore per l'audit log è letto dai claim del JWT tramite `IHttpContextAccessor` (`ClaimTypes.NameIdentifier`, `ClaimTypes.Email`), con fallback `"system"` se non presenti (es. seed).
- La scrittura dell'audit log è **esplicita e manuale**, non automatica — va replicata in ogni Handler che deve essere tracciato.

## Gestione delle eccezioni

`Features/_Shared/Middleware/ExceptionHandlingMiddleware.cs` (estratto):
```csharp
public async Task InvokeAsync(HttpContext context)
{
    try { await next(context); }
    catch (Exception ex) { await HandleExceptionAsync(context, ex); }
}

private async Task HandleExceptionAsync(HttpContext context, Exception exception)
{
    var (status, title) = exception switch
    {
        ValidationException      => (StatusCodes.Status400BadRequest,  "Validation failed"),
        NotFoundException        => (StatusCodes.Status404NotFound,    "Resource not found"),
        UnauthorizedException    => (StatusCodes.Status401Unauthorized, "Unauthorized"),
        ForbiddenException       => (StatusCodes.Status403Forbidden,   "Forbidden"),
        ConflictException        => (StatusCodes.Status409Conflict,    "Conflict"),
        MongoWriteException we when we.WriteError?.Category == ServerErrorCategory.DuplicateKey
                                 => (StatusCodes.Status409Conflict,    "Conflict"),
        _                        => (StatusCodes.Status500InternalServerError, "Internal server error")
    };

    if (status == StatusCodes.Status500InternalServerError)
    {
        logger.LogError(exception, "Unhandled exception");
        await SaveErrorLogAsync(context, exception);   // scope dedicato: repository è scoped, middleware è singleton
    }

    var problem = new ProblemDetails { Status = status, Title = title, Detail = exception.Message, Instance = context.Request.Path };
    if (exception is ValidationException validationEx)
    {
        problem.Extensions["errors"] = validationEx.Errors
            .GroupBy(e => e.PropertyName)
            .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage).ToArray());
    }
    // ... scrive problem come JSON camelCase, content-type application/problem+json
}
```
Da notare: il salvataggio dell'`ErrorLogDocument` avviene **solo** per errori 500 non gestiti, dentro un `try/catch` interno che non ripropaga (`SaveErrorLogAsync`) — se il logging stesso fallisce, non deve far cadere la risposta di errore originale.

## Come aggiungere un nuovo endpoint CRUD (convenzione del progetto)

Segui esattamente la struttura di `Features/Customers/`:

1. Crea `Features/<Entità>/` con una sottocartella per ogni operazione: `Create<Entità>/`, `Update<Entità>/`, `Delete<Entità>/`, `Get<Entità>ById/`, `Get<Entità>/`.
2. In ciascuna, crea `Command`/`Query` (record `IRequest<TResponse>`), `Handler`, `Response`, e — solo per Create/Update — un `Validator`.
3. Aggiungi il documento Mongo in `Features/_Shared/Documents/<Entità>Document.cs` e la coppia repository (`I<Entità>Repository` + `<Entità>Repository`) in `Features/_Shared/Repositories/`.
4. Registra il repository in `Features/_Shared/Extensions/RepositoryExtensions.cs::AddRepositories()`.
5. Crea `Features/<Entità>/<Entità>Endpoints.cs` con `MapGroup("/api/<entità>")` e i vari `Map{Get,Post,Put,Delete}` con `RequireAuthorization(p => p.RequireClaim("permissions", "<entità>.<azione>"))`.
6. Aggiungi la chiamata `app.Map<Entità>Endpoints();` in `Features/_Shared/Extensions/EndpointExtensions.cs::MapAllEndpoints()` — **passaggio facile da dimenticare**, senza di esso l'endpoint non esiste a runtime.
7. Se serve tracciamento, aggiungi manualmente la scrittura di `AuditLogDocument` negli Handler Create/Update/Delete (vedi skill di progetto `audit-log`).
8. Se servono permessi nuovi, seguili end-to-end con la skill di progetto `permissions` (seed permesso, assegnazione a ruolo, claim, guardia frontend).

Skill già presenti nel repo che automatizzano parte di questo scaffolding: `vertical-slice-backend`, `mongo-entity`, `audit-log`, `permissions`, `pagination`, `search-filter`.

## Stampa preventivo (.docx)

Flusso: `Features/Drafts/PrintDraft/PrintDraftHandler.cs` carica il `DraftDocument`, il `CustomerDocument`, sceglie il `TemplateDocument` (di default o per id esplicito), costruisce i token e chiama `TemplateRenderer.Render`:

```csharp
var globalTokens = BuildGlobalTokens(draft, customer);   // es. NUMERO_PREVENTIVO, CLIENTE_NOME, TOTALE_PREVENTIVO
var positions = draft.Items.Select(item =>
{
    imageFileByType.TryGetValue(item.WindowTypeId, out var fileName);
    var imageBytes = WindowTypeAssets.Load(fileName);
    return new RenderPosition(BuildItemTokens(item, draft), fileName, imageBytes);
}).ToList();
var rendered = TemplateRenderer.Render(template.FileContent, globalTokens, positions);
```

Il motore `Features/Drafts/PrintDraft/TemplateRenderer.cs` **non usa una libreria OOXML** (niente OpenXML SDK): apre il `.docx` come `ZipArchive`, individua `word/document.xml` e lo tratta come XML puro (`XDocument`):

```csharp
private static readonly Regex TokenRegex = new(@"\{\{\s*([a-zA-Z0-9_.]+)\s*\}\}", RegexOptions.Compiled);
private static readonly Regex ItemTokenRegex = new(@"\{\{\s*POS_(?:N|\d+)_([A-Za-z0-9_]+)\s*\}\}", RegexOptions.Compiled);
private static readonly Regex MarkerRegex = new(@"\{\{\s*POS_(?:START|END)\s*\}\}", RegexOptions.Compiled);
private static readonly Regex PosNumberRegex = new(@"\{\{\s*POS_N\s*\}\}", RegexOptions.Compiled);
private static readonly Regex ImageMarkerRegex = new(@"\{\{\s*POS_(?:N|\d+)_IMMAGINE\s*\}\}", RegexOptions.Compiled);
```

Convenzioni token (già annotate in memoria di progetto, confermate nel codice):
- **Token globali** `{{NOME_TOKEN}}` — sostituiti ovunque nel documento (testata, footer, ecc.).
- **Blocco ripetibile posizioni**: delimitato dai marcatori `{{POS_START}}` / `{{POS_END}}` nel template Word. Il renderer individua il più piccolo elemento OOXML che contiene entrambi i marcatori (riga di tabella `<w:tr>`, tabella `<w:tbl>` o paragrafo `<w:p>`) e lo **clona una volta per ogni voce** del preventivo (`ExpandItems`).
- Dentro il blocco clonato: `{{POS_N_CAMPO}}` (es. `{{POS_N_TIPOLOGIA}}`, `{{POS_N_DIMENSIONI}}`, `{{POS_N_VETRO}}`, `{{POS_N_FERRAMENTA}}`) e `{{POS_N}}` per il numero progressivo di posizione.
- **Immagini tipologia**: un'immagine segnaposto nel template con Alt Text `{{POS_N_IMMAGINE}}` viene, per ogni posizione clonata, ricollegata (`r:embed`) all'immagine reale della tipologia serramento — aggiunta come nuova "media part" nel pacchetto ZIP, con relationship e content-type registrati (`AddMediaParts`, `AddImageRelationships`, `EnsurePngContentType`).
- Gestisce anche il caso (frequente con Word) in cui un token è spezzato su più `<w:r>` (run) a causa di correzioni automatiche — `NormalizeSplitTokens` ricompone il testo prima di applicare le regex.

Le immagini delle tipologie di serramento sono caricate da `WindowTypeAssets.cs` come **embedded resource** (`Features/_Shared/Assets/WindowTypes/*.png`, dichiarate nel `.csproj`), non da filesystem, con cache statica in `ConcurrentDictionary`.

I 4 template di partenza (`TEMPLATE_Preventivo_BASE/BOZZA/CLASSICO/MINIMAL.docx`) sono anch'essi embedded resource, caricati dal `DataSeeder` al primo avvio se il DB è vuoto.

> ⚠️ **TODO verificato**: il pacchetto **SkiaSharp** è referenziato nel `.csproj` ma non risulta usato in nessun file `.cs` del progetto (nessun `SKBitmap`/`SKImage`/`SkiaSharp` nel codice). Se stai cercando dove viene elaborata la grafica delle immagini, non è qui — probabilmente dipendenza non ancora utilizzata o residuo da funzionalità pianificata. Verifica con il team prima di rimuoverla o di basarti su di essa.

Per modificare un template esistente o crearne uno nuovo, vedi la memoria di progetto su convenzioni token e marcatori (`preventivo-print-templates`), che descrive come ri-creare i file `.docx` mantenendo i marcatori `{{POS_START}}`/`{{POS_END}}` intatti in Word.

## Come intervenire in sicurezza nel backend

**Convenzioni di naming**:
- Cartelle feature in PascalCase plurale (`Customers`, `Series`), sottocartelle operazione in PascalCase verbo+entità singolare (`CreateCustomer`, `GetCustomerById`).
- File `<Operazione><Entità>Command.cs` / `...Query.cs` / `...Handler.cs` / `...Validator.cs` / `...Response.cs`.
- Permessi come stringa `entità_plurale.azione` (`customers.read`, `customers.write`, `customers.delete`) — azione tipicamente `read`/`write`/`delete`, mai verbi liberi.

**Dove aggiungere codice nuovo**:
- Nuova feature CRUD → nuova cartella sotto `Features/`, seguendo i 8 passi sopra.
- Logica cross-cutting riusabile da più feature → `Features/_Shared/` (behaviors, extensions, helpers), non duplicata per feature.
- Nuovo tipo di eccezione applicativa → `Features/_Shared/Exceptions/`, e aggiungi il case nello `switch` di `ExceptionHandlingMiddleware`.

**Cosa NON toccare senza motivo**:
- L'**ordine dei middleware** in `Program.cs` (CORS prima di tutto, ExceptionHandling subito dopo) — invertirlo cambia comportamento di CORS/preflight e di gestione errori.
- L'**ordine di registrazione dei pipeline behavior MediatR** (`LoggingBehavior` prima di `ValidationBehavior`) — se invertito, gli errori di validazione non verrebbero più loggati correttamente.
- Il `ConventionPack` globale Mongo (`AddMongoDb`) — è condiviso da **tutte** le collection; una modifica (es. cambiare naming convention) impatta ogni documento esistente nel DB.
- `EnsureIndexesAsync` — attenzione a non introdurre indici `Unique = true` su collection con dati esistenti non conformi, l'avvio dell'app fallirebbe.
- I file `.docx` in `Features/_Shared/Seed/Templates/` — vanno editati solo in Word preservando i marcatori `{{...}}`, mai come testo semplice (l'XML sottostante si romperebbe).
