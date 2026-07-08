# Backend — Livello 1: Panoramica tecnologie

> Progetto: `apps/backend/Tama.Api` — .NET **10.0** (Minimal API), verificato in `Tama.Api.csproj` (`<TargetFramework>net10.0</TargetFramework>`).

Questo documento elenca le tecnologie usate nel backend e il ruolo di ciascuna, senza entrare nei dettagli implementativi (per quelli vedi [backend-02-funzionamento](backend-02-funzionamento.md) e [backend-03-dettaglio](backend-03-dettaglio.md)).

## Framework e runtime

| Tecnologia | Ruolo nel progetto |
|---|---|
| **.NET 10 / ASP.NET Core Minimal API** | Runtime e modello di hosting. Non ci sono Controller MVC: ogni endpoint è definito come funzione registrata su un `IEndpointRouteBuilder`. |
| **Nullable reference types abilitati** | `<Nullable>enable</Nullable>` nel csproj: il compilatore segnala riferimenti potenzialmente null. |

## Librerie applicative (NuGet)

| Pacchetto | Versione | Ruolo |
|---|---|---|
| **MediatR** | 12.4.0 | Implementa il pattern CQRS: ogni operazione (Command o Query) è una classe inviata a un mediatore, che la instrada all'Handler corrispondente. Disaccoppia gli endpoint dalla logica di business. |
| **FluentValidation** + **FluentValidation.DependencyInjectionExtensions** | 11.9.2 | Validazione dichiarativa degli input (Command/Query) tramite classi `Validator` separate, eseguite automaticamente prima dell'Handler. |
| **MongoDB.Driver** | 2.29.0 | Driver ufficiale per MongoDB. Non è usato alcun ORM relazionale (niente Entity Framework): l'accesso ai dati è diretto tramite `IMongoCollection<T>`. |
| **Microsoft.AspNetCore.Authentication.JwtBearer** | 10.0.0 | Autenticazione basata su token JWT firmati (HMAC-SHA256), usata per **tutte** le richieste autenticate all'API. |
| **Microsoft.Identity.Web** | 4.6.0 | Utilità Microsoft per l'integrazione con Azure AD, usata nel flusso di login alternativo "MSAL" (vedi sezione Auth in [backend-02](backend-02-funzionamento.md)). |
| **BCrypt.Net-Next** | 4.0.3 | Hashing one-way delle password per il login locale (email + password). |
| **Microsoft.AspNetCore.OpenApi** | 10.0.0 | Generazione dello schema OpenAPI (`/openapi/v1.json`) a partire dagli endpoint Minimal API. |
| **Scalar.AspNetCore** | 2.5.5 | UI di documentazione API interattiva moderna, esposta oltre alla classica Swagger UI. |
| **Swashbuckle.AspNetCore.SwaggerUI** | 7.2.0 | UI Swagger classica, montata su `/swagger`, che consuma lo schema generato da `Microsoft.AspNetCore.OpenApi`. |
| **SkiaSharp** | 3.116.1 | Libreria di grafica 2D. **Dichiarata nel csproj ma nessun utilizzo trovato nel codice** (`grep` su `SkiaSharp\|SKBitmap\|SKImage` non produce risultati). ⚠️ TODO: verificare con il team se è dipendenza residua/non ancora usata o se va rimossa. |

## Persistenza

- **MongoDB** come unico datastore, con collection tipizzate tramite POCO (`*Document`) in `Features/_Shared/Documents/`.
- Nessuna migration nel senso EF Core: la struttura dei documenti è implicita nel codice C#; gli indici vengono creati programmaticamente all'avvio (`EnsureIndexesAsync`, vedi livello 2).
- Contenuti binari (template `.docx`, immagini tipologie serramento `.png`) sono **embedded resource** nell'assembly, non file su disco a runtime.

## Autenticazione e autorizzazione

- **Doppia strategia di login**: email+password locale (BCrypt) oppure Azure AD via MSAL — ma in entrambi i casi il backend emette sempre un proprio **JWT interno**, usato per tutte le chiamate successive. Dettagli in [backend-02](backend-02-funzionamento.md) e nel documento pre-esistente `docs/Authentication-Strategy.md`.
- **Autorizzazione basata su permessi**: ogni endpoint richiede un claim `permissions` con uno specifico valore (es. `customers.read`), non ruoli fissi a livello di policy centralizzata.

## Cross-cutting

| Aspetto | Come è gestito |
|---|---|
| **Logging delle richieste MediatR** | `LoggingBehavior<TRequest,TResponse>`, pipeline behavior custom. |
| **Validazione automatica** | `ValidationBehavior<TRequest,TResponse>`, pipeline behavior custom che esegue i validator FluentValidation registrati. |
| **Gestione errori centralizzata** | `ExceptionHandlingMiddleware`, mappa le eccezioni a risposte `ProblemDetails` (RFC 7807). |
| **Audit trail** | Scrittura manuale (non automatica) di record `AuditLogDocument` dentro ogni Handler Create/Update/Delete. |
| **Rate limiting** | `Microsoft.AspNetCore.RateLimiting` (built-in .NET), due policy: `error-log-policy` e `auth-policy` (anti brute-force). |
| **CORS** | Policy permissiva su `localhost` in sviluppo, whitelist esplicita in produzione. |
| **Localizzazione** | `RequestLocalization` su header `Accept-Language`, culture supportate `it`/`en`. |

## Cosa NON c'è

Per evitare assunzioni errate leggendo altri progetti .NET:

- Nessun **Controller** (`ControllerBase`) — solo Minimal API.
- Nessun **Entity Framework Core** — solo `MongoDB.Driver`.
- Nessun **AutoMapper** — la mappatura tra `Document` (Mongo) e `Response` (DTO) è scritta a mano dentro gli Handler.
- Nessuna **Clean Architecture a progetti separati** (Domain/Application/Infrastructure) — tutto vive in un solo progetto `Tama.Api`, organizzato per feature verticale (vedi livello 2).
