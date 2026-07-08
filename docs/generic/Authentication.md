# 🔐 Autenticazione — Documentazione Completa (JWT + MSAL)

> **Documento**: Autenticazione basata su JWT e MSAL (Azure AD)
> **Lingua**: Italiano
> **Ultimo aggiornamento**: 2026-03-29
> **Versione**: 2.0

---

## 📋 Indice

1. [Livello Base](#livello-base) — cos'è, come funziona, esempi
2. [Flussi Principali](#flussi-principali) — diagrammi e spiegazioni
3. [Livello Tecnico Avanzato](#livello-tecnico-avanzato) — implementazione dettagliata
4. [Configurazione](#configurazione) — setup e parametri
5. [Esempi di Request/Response](#esempi-di-requestresponse) — curl e payload JSON
6. [Sicurezza](#sicurezza) — considerazioni e best practices
7. [Troubleshooting](#troubleshooting) — problemi comuni e soluzioni

---

# Livello Base

## Cos'è l'autenticazione JWT?

JWT (JSON Web Token) è un sistema per autenticare gli utenti. Funziona così:

1. **L'utente accede** con email e password
2. **Il server verifica** le credenziali
3. **Il server crea un JWT** — un "biglietto" firmato che contiene l'identità dell'utente
4. **Il client riceve il JWT** e lo usa per fare richieste autenticate
5. **Il server verifica il JWT** su ogni richiesta — se valido, consente l'accesso

### Perché JWT?

- ✅ **Stateless**: il server non deve ricordare nulla sui client (niente sessioni in database)
- ✅ **Scalabile**: funziona bene anche con molti server
- ✅ **Sicuro**: il JWT è firmato digitalmente, quindi non può essere falsificato
- ✅ **Mobile-friendly**: facile da usare nelle app mobili

### Cosa contiene un JWT?

Un JWT ha 3 parti (separate da punti):

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJzdWIiOiI2NWZkMzBhYjEyMzQ1YzAwMDFhYmNkZWYiLCJlbWFpbCI6InVzZXJAZXhhbXBsZS5jb20ifQ
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

1. **Header** (algoritmo di firma): `HS256`
2. **Payload** (i dati): `sub` (ID utente), `email`, `permissions` (permessi)
3. **Signature** (firma digitale): garantisce che il token non sia stato modificato

### Esempio semplice

Immagina di accedere a un hotel:

```
┌─────────────┐
│   Hotel     │
│  (Server)   │
└─────────────┘
      ↑
      │ "Ho la prenotazione a nome Mario Rossi"
      │
   ┌──────────────────────┐
   │ "Ecco la tua chiave"  │  ← JWT (biglietto)
   │ Numero stanza: 205    │
   │ Validità: 3 giorni    │
   │ (Firmato dal nostro   │
   │  sistema di booking)  │
   └──────────────────────┘
      │
      ↓
   ┌─────────┐
   │  Mario  │
   │(Client) │
   └─────────┘
```

Quando Mario vuole accedere alla stanza, mostra il biglietto all'hotel. L'hotel verifica che è autentico (firmato) e lo lascia entrare.

---

## Flussi Principali

### 1️⃣ Flusso di Login

**Cosa accade**: l'utente invia email e password, e riceve un JWT.

```
PASSO 1: INVIO CREDENZIALI
┌─────────────────────┐
│   BROWSER / APP     │
├─────────────────────┤
│ Email: mario@...    │
│ Password: ****      │
└──────────┬──────────┘
           │
           │  POST /api/auth/login
           │  { email, password }
           │
           ▼
┌─────────────────────────────────────┐
│       SERVER - LoginHandler.cs      │
├─────────────────────────────────────┤
│ PASSO 2: VALIDA CREDENZIALI         │
│  ✓ Cerca utente per email           │
│  ✓ Verifica password (BCrypt)       │
│  ✓ Verifica account attivo          │
│                                     │
│ PASSO 3: RISOLVE PERMESSI           │
│  ✓ Ruoli diretti dell'utente        │
│  ✓ Ruoli ereditati da gruppi        │
│  ✓ Tutti i permessi associati       │
│                                     │
│ PASSO 4: GENERA TOKEN               │
│  ✓ JWT (AccessToken) — scade 60min  │
│  ✓ RefreshToken — scade 7 giorni    │
└──────────┬──────────────────────────┘
           │
           │  200 OK
           │  {
           │    accessToken: "eyJ...",
           │    expiresAt: "2026-03-29T11:30:00Z",
           │    userId: "65fd30ab...",
           │    email: "mario@...",
           │    displayName: "Mario Rossi",
           │    language: "it",
           │    permissions: ["users.read", "roles.manage"]
           │  }
           │
           ▼
┌─────────────────────────────────────┐
│   BROWSER / APP                     │
├─────────────────────────────────────┤
│ ✓ Salva accessToken in localStorage │
│ ✓ Riceve refreshToken in cookie     │
│   (HttpOnly, non leggibile da JS)   │
│ ✓ Utente LOGGATO ✓                  │
└─────────────────────────────────────┘
```

**Cosa riceve il client**:
- `accessToken` — JWT da usare per le richieste autenticate
- `expiresAt` — quando scade il token (es. tra 60 minuti)
- `userId`, `email`, `displayName`, `language`
- `permissions` — array di permessi (es. `["users.read", "roles.manage"]`)
- **Cookie `refreshToken`** — salvato automaticamente dal browser (HttpOnly, non accessibile da JavaScript)

---

### 2️⃣ Flusso di Accesso Protetto

Una volta che il client ha il JWT, lo usa su tutte le richieste autenticate.

```
PASSO 1: INVIO RICHIESTA CON JWT
┌─────────────────────────────────────┐
│       BROWSER / APP                 │
├─────────────────────────────────────┤
│ GET /api/users                      │
│ Authorization: Bearer eyJ...        │
└──────────┬──────────────────────────┘
           │
           │
           ▼
┌─────────────────────────────────────┐
│    SERVER - JWT Middleware          │
├─────────────────────────────────────┤
│ PASSO 2: VERIFICA JWT               │
│                                     │
│ ✓ Parse il token                    │
│ ✓ Verifica firma (HMAC-SHA256)      │
│ ✓ Non è scaduto? (expiry check)     │
│ ✓ Issuer e Audience corretti?       │
│                                     │
│ RISULTATO:                          │
│  JWT VALIDO ✓                       │
│  → Estrai claims (sub, email,       │
│    permissions)                     │
│  → Consenti accesso                 │
└──────────┬──────────────────────────┘
           │
           │  ✓ JWT Valid
           │  → Passa al handler
           │
           ▼
┌─────────────────────────────────────┐
│   Endpoint Handler /api/users       │
├─────────────────────────────────────┤
│ Accesso ai dati del JWT:            │
│  • user.Id                          │
│  • user.Email                       │
│  • user.Permissions                 │
│                                     │
│ Esegue la logica endpoint           │
└──────────┬──────────────────────────┘
           │
           │  200 OK
           │  { data }
           │
           ▼
┌─────────────────────────────────────┐
│   BROWSER / APP                     │
├─────────────────────────────────────┤
│ ✓ Riceve i dati richiesti           │
└─────────────────────────────────────┘
```

---

### 3️⃣ Flusso di Refresh Token

Quando il JWT scade (dopo 60 minuti), il client lo rinnova usando il refresh token.

```
SITUAZIONE: JWT è scaduto (oltre 60 minuti)
┌─────────────────────────────────────┐
│       BROWSER / APP                 │
├─────────────────────────────────────┤
│ Tenta: GET /api/users               │
│ Authorization: Bearer (scaduto)     │
│                                     │
│ RISPOSTA: 401 Unauthorized          │
│ → Recupera refreshToken dal cookie  │
└──────────┬──────────────────────────┘
           │
           │  PASSO 1: RINNOVA TOKEN
           │  POST /api/auth/refresh
           │  Cookie: refreshToken=abcd1234...
           │
           ▼
┌─────────────────────────────────────┐
│  SERVER - RefreshTokenHandler.cs    │
├─────────────────────────────────────┤
│ PASSO 2: VALIDA REFRESH TOKEN       │
│                                     │
│ ✓ Token esiste in MongoDB?          │
│ ✓ Non è scaduto?                    │
│ ✓ Non è stato revocato?             │
│ ✓ Utente esiste e è attivo?         │
│                                     │
│ ❌ FALLISCE? → 401 Unauthorized     │
│                                     │
│ PASSO 3: TOKEN ROTATION             │
│                                     │
│ ✓ Revoca vecchio token (RT-old)     │
│   → marcato come revoked in DB      │
│   → non può più essere usato        │
│                                     │
│ ✓ Genera nuovo JWT (AT-new)         │
│   → valido per 60 minuti            │
│                                     │
│ ✓ Genera nuovo RefreshToken (RT-new)│
│   → salvato in MongoDB              │
│   → valido per 7 giorni             │
└──────────┬──────────────────────────┘
           │
           │  200 OK
           │  {
           │    accessToken: "eyJ..." (NUOVO),
           │    expiresAt: "2026-03-29T12:30:00Z",
           │    permissions: [...]
           │  }
           │
           │  Set-Cookie: refreshToken=wxyz5678...
           │  (nuovo token, RUOTATO)
           │
           ▼
┌─────────────────────────────────────┐
│   BROWSER / APP                     │
├─────────────────────────────────────┤
│ ✓ Salva nuovo accessToken           │
│ ✓ Cookie aggiornato con nuovo token │
│ ✓ Può continuare a fare richieste   │
└─────────────────────────────────────┘

TIMELINE DELLA ROTATION (Sicurezza):
═════════════════════════════════════════
1️⃣  Login        → Crea RT-1
2️⃣  Refresh      → Revoca RT-1, crea RT-2
3️⃣  Refresh      → Revoca RT-2, crea RT-3
4️⃣  Se attacker ruba RT-1 e tenta refresh
    → "Token not found" (è revocato)
    → ❌ Fallisce istantaneamente
```

**Perché il refresh token viene "ruotato"?**

Ogni volta che lo usi, ricevi uno nuovo e il vecchio viene disabilitato. Così se qualcuno ruba il refresh token, può usarlo solo una volta prima che sia inutilizzabile.

---

### 4️⃣ Flusso di Logout

```
PASSO 1: LOGOUT REQUEST
┌─────────────────────────────────────┐
│       BROWSER / APP                 │
├─────────────────────────────────────┤
│ Utente clicca "Logout"              │
└──────────┬──────────────────────────┘
           │
           │  POST /api/auth/logout
           │  Cookie: refreshToken=abcd1234...
           │
           ▼
┌─────────────────────────────────────┐
│  SERVER - LogoutHandler.cs          │
├─────────────────────────────────────┤
│ PASSO 2: REVOCA REFRESH TOKEN       │
│                                     │
│ ✓ Cerca token in MongoDB            │
│ ✓ Se trovato → marca come revoked   │
│   (revokedAt = now)                 │
│                                     │
│ ℹ️  Non lancia errore se non trovato │
│    (idempotente - safe da duplicati)│
└──────────┬──────────────────────────┘
           │
           │  204 No Content
           │  (successo, body vuoto)
           │
           │  Set-Cookie: refreshToken=
           │  (cancella il cookie)
           │
           ▼
┌─────────────────────────────────────┐
│   BROWSER / APP                     │
├─────────────────────────────────────┤
│ ✓ Pulisce localStorage (accessToken)│
│ ✓ Cookie refreshToken eliminato     │
│ ✓ Utente DISCONNESSO ✓              │
│                                     │
│ Per accedere di nuovo: login        │
└─────────────────────────────────────┘
```

Dopo il logout, il refresh token non può più essere usato (è revocato). L'utente deve accedere di nuovo.

---

### 5️⃣ Flusso di MSAL Login (Azure AD)

Per gli utenti che accedono con Azure AD (Microsoft/Office 365).

```
PASSO 1: REDIRECT A AZURE
┌─────────────────────────────────────┐
│       BROWSER / APP                 │
├─────────────────────────────────────┤
│ Utente clicca: "Login con Azure"    │
└──────────┬──────────────────────────┘
           │
           │  Reindirizzamento
           │  → https://login.microsoftonline.com
           │
           ▼
       ┌─────────────────────────┐
       │    AZURE AD (Cloud)     │
       ├─────────────────────────┤
       │ • Mostra form login     │
       │ • Utente inserisce cred │
       │ • Verifica 2FA se abili │
       │ • Genera JWT di Azure   │
       └──────────┬──────────────┘
                  │
                  │ PASSO 2: RICEVE TOKEN AZURE
                  │ Reindirizza indietro
                  │ con token nella URL
                  │
           ▼
┌─────────────────────────────────────┐
│       BROWSER / APP                 │
├─────────────────────────────────────┤
│ Ha token Azure:                     │
│ eyJhbGciOiJSUzI1NiIsInR5cCI... (JWT│
│ di Azure)                           │
└──────────┬──────────────────────────┘
           │
           │  PASSO 3: INVIA A BACKEND
           │  POST /api/auth/msal-login
           │  { azureToken: "eyJ..." }
           │
           ▼
┌─────────────────────────────────────┐
│ SERVER - MsalLoginHandler.cs        │
├─────────────────────────────────────┤
│ PASSO 4: LEGGE TOKEN AZURE           │
│                                     │
│ ✓ Parse il JWT di Azure             │
│ ✓ Estrae email dal claim            │
│   (preferred_username or email)     │
│ ℹ️  Token già validato da MSAL       │
│   lato frontend (firma RSA)         │
│                                     │
│ PASSO 5: TROVA UTENTE LOCALE        │
│                                     │
│ ✓ Cerca in MongoDB per email        │
│ ✓ Utente DEVE essere creato in      │
│   anticipo nel sistema locale       │
│ ✓ Verifica sia attivo               │
│                                     │
│ PASSO 6: GENERA JWT INTERNO         │
│                                     │
│ ✓ Risolve permessi dell'utente      │
│ ✓ Crea JWT interno (stesso formato  │
│   del login classico)               │
│ ✓ Crea RefreshToken                 │
│                                     │
└──────────┬──────────────────────────┘
           │
           │  200 OK
           │  {
           │    accessToken: "eyJ..." (INTERNO),
           │    expiresAt: "...",
           │    userId: "65fd30ab...",
           │    email: "mario@company.com",
           │    displayName: "Mario Rossi",
           │    language: "it",
           │    permissions: [...]
           │  }
           │
           │  Set-Cookie: refreshToken=...
           │
           ▼
┌─────────────────────────────────────┐
│   BROWSER / APP                     │
├─────────────────────────────────────┤
│ ✓ Riceve JWT interno (non Azure)    │
│ ✓ Usa come qualsiasi JWT normale    │
│ ✓ Utente LOGGATO ✓                  │
└─────────────────────────────────────┘
```

Il server legge il token Azure (ricevuto dal frontend tramite la libreria `@azure/msal-browser`, che ha già validato la firma RSA di Microsoft), estrae l'email, trova l'utente locale corrispondente, e genera un JWT interno per accedere all'API.

---

# Flussi Principali

## Overview Architetturale

```
┌──────────────────────────────────────────────────────────┐
│                     TAMA API                       │
├──────────────────────────────────────────────────────────┤
│                                                            │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              AuthEndpoints.cs                       │ │
│  │  - POST /api/auth/login                            │ │
│  │  - POST /api/auth/msal-login                       │ │
│  │  - POST /api/auth/refresh                          │ │
│  │  - POST /api/auth/logout                           │ │
│  └─────────────────────────────────────────────────────┘ │
│                        │                                  │
│                        ↓                                  │
│  ┌─────────────────────────────────────────────────────┐ │
│  │   MediatR Handlers (Vertical Slice Pattern)        │ │
│  │                                                     │ │
│  │  • LoginHandler → verifica password, genera token  │ │
│  │  • MsalLoginHandler → valida token Azure, genera   │ │
│  │  • RefreshTokenHandler → rinnova JWT, ruota token  │ │
│  │  • LogoutHandler → revoca refresh token            │ │
│  │                                                     │ │
│  └─────────────────────────────────────────────────────┘ │
│                        │                                  │
│                        ↓                                  │
│  ┌─────────────────────────────────────────────────────┐ │
│  │         JwtTokenHelper.cs (Shared)                  │ │
│  │                                                     │ │
│  │  • GenerateAccessToken() → crea JWT                │ │
│  │  • CreateRefreshTokenAsync() → salva nel DB        │ │
│  │                                                     │ │
│  └─────────────────────────────────────────────────────┘ │
│                        │                                  │
│                        ↓                                  │
│  ┌─────────────────────────────────────────────────────┐ │
│  │    MongoDB Repositories (Accesso Dati)             │ │
│  │                                                     │ │
│  │  • IUserRepository → lettura/scrittura utenti      │ │
│  │  • IRefreshTokenRepository → gestione token        │ │
│  │  • IRoleRepository, IGroupRepository →             │ │
│  │    risolvere permessi                              │ │
│  │                                                     │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │             MONGODB DATABASE                         │ │
│  │  Collections: users, refreshTokens, roles, groups   │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                            │
└──────────────────────────────────────────────────────────┘
```

---

## Ciclo Completo: dal Login all'Accesso Protetto

```
1. USER LOGS IN
   Email + Password
         │
         ↓
┌─────────────────────────────────────┐
│ LoginHandler.Handle()               │
│ [File: LoginHandler.cs:17-43]       │
│                                     │
│ 1. Cerca utente per email           │
│    [userRepo.GetByEmailAsync]       │
│ 2. Verifica password con BCrypt     │
│    [BCrypt.Verify]                  │
│ 3. Risolve permessi                 │
│    [ResolvePermissionsAsync]        │
│ 4. Genera Access Token (JWT)        │
│    [JwtTokenHelper.GenerateAccessToken] │
│ 5. Crea Refresh Token (DB)          │
│    [JwtTokenHelper.CreateRefreshTokenAsync] │
└─────────────────────────────────────┘
         │
         ↓ Returns LoginResponse
    ACCESS TOKEN SENT TO FRONTEND
    REFRESH TOKEN SET IN COOKIE
         │
         ↓
2. CLIENT MAKES PROTECTED REQUEST
   GET /api/users
   Header: Authorization: Bearer {accessToken}
         │
         ↓
┌─────────────────────────────────────┐
│ ASP.NET Core Middleware             │
│ (JWT Validation)                    │
│                                     │
│ • Parse JWT                         │
│ • Verify Signature                  │
│ • Check Expiration                  │
│ • Extract Claims (sub, permissions) │
│                                     │
│ ✅ Valid → Allow Request            │
│ ❌ Invalid → 401 Unauthorized       │
└─────────────────────────────────────┘
         │
         ↓
  ENDPOINT HANDLER EXECUTES
  Has access to user info from JWT
         │
         ↓
  RESPONSE SENT TO CLIENT
         │
         ↓
3. TOKEN EXPIRES (60 min)
   Client tries to use old token
         │
         ↓
   401 Unauthorized response
         │
         ↓
4. CLIENT REFRESHES TOKEN
   POST /api/auth/refresh
   Cookie: refreshToken={refreshToken}
         │
         ↓
┌─────────────────────────────────────┐
│ RefreshTokenHandler.Handle()        │
│ [File: RefreshTokenHandler.cs:16-57]│
│                                     │
│ 1. Trova refresh token nel DB       │
│    [refreshRepo.GetByTokenAsync]    │
│ 2. Verifica non sia scaduto         │
│    [ExpiresAt < DateTime.UtcNow]    │
│ 3. Verifica utente esista e attivo  │
│    [userRepo.GetByIdAsync]          │
│ 4. REVOCA vecchio token (rotation)  │
│    [refreshRepo.RevokeAsync]        │
│ 5. Genera nuovo JWT                 │
│    [JwtTokenHelper.GenerateAccessToken] │
│ 6. Crea nuovo Refresh Token         │
│    [JwtTokenHelper.CreateRefreshTokenAsync] │
└─────────────────────────────────────┘
         │
         ↓
  NEW ACCESS TOKEN SENT
  NEW REFRESH TOKEN IN COOKIE
         │
         ↓
5. USER LOGS OUT
   POST /api/auth/logout
   Cookie: refreshToken={refreshToken}
         │
         ↓
┌─────────────────────────────────────┐
│ LogoutHandler.Handle()              │
│ [File: LogoutHandler.cs:8-15]       │
│                                     │
│ • Revoca refresh token              │
│   [refreshRepo.RevokeAsync]         │
│ • Cancella cookie (frontend)        │
└─────────────────────────────────────┘
         │
         ↓
  REFRESH TOKEN INVALID
  USER MUST LOGIN AGAIN
```

---

# Livello Tecnico Avanzato

## 🏗️ Architettura Implementativa

### Pattern: Vertical Slice

L'autenticazione segue il pattern **Vertical Slice**: ogni feature (Login, RefreshToken, Logout, MsalLogin) è autocontenuta nella propria cartella.

```
Features/Auth/
├── Login/
│   ├── LoginCommand.cs              # Input
│   ├── LoginHandler.cs              # Logica
│   ├── LoginValidator.cs            # Validazione (FluentValidation)
│   └── LoginResponse.cs             # Output
│
├── RefreshToken/
│   ├── RefreshTokenCommand.cs
│   ├── RefreshTokenHandler.cs
│   ├── RefreshTokenResponse.cs
│   └── (nessun validator — logica nel handler)
│
├── MsalLogin/
│   ├── MsalLoginCommand.cs
│   ├── MsalLoginHandler.cs
│   └── MsalLoginResponse.cs
│
├── Logout/
│   ├── LogoutCommand.cs
│   └── LogoutHandler.cs
│
├── Shared/
│   └── JwtTokenHelper.cs            # Utility condivisa
│
└── AuthEndpoints.cs                 # Routing (Minimal API)
```

### Tecnologie Utilizzate

| Componente | Libreria | Uso |
|---|---|---|
| **Framework Web** | ASP.NET Core 10 (Minimal API) | Routing, middleware, cookie handling |
| **CQRS / Mediator** | MediatR v11 | Command/Query pattern, pipeline behaviors |
| **JWT** | `System.IdentityModel.Tokens.Jwt` | Parsing, creazione, validazione token |
| **Password Hashing** | BCrypt.Net | Hash sicuro delle password |
| **Database** | MongoDB | Persistenza refresh token, utenti, ruoli |
| **Validation** | FluentValidation | Validazione input (LoginCommand) |
| **Configuration** | `IConfiguration` | Lettura impostazioni da appsettings.json |

---

## 📝 Dettaglio dei File e Metodi

### 1. **AuthEndpoints.cs** — Il Routing

```csharp
// File: Features/Auth/AuthEndpoints.cs
public static class AuthEndpoints
{
    public static IEndpointRouteBuilder MapAuthEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/auth").WithTags("Auth");

        // 1. LOGIN ENDPOINT
        group.MapPost("/login", async (LoginCommand cmd, IMediator mediator, HttpContext ctx, ...) =>
        {
            var res = await mediator.Send(cmd, ct);
            SetRefreshCookie(ctx, res.RefreshToken, config);  // Set-Cookie header
            return Results.Ok(new { res.AccessToken, res.ExpiresAt, ... });
        })
        .AllowAnonymous()
        .RequireRateLimiting("auth-policy");  // Rate limit (protezione brute-force)

        // 2. REFRESH ENDPOINT
        group.MapPost("/refresh", async (IMediator mediator, HttpContext ctx, ...) =>
        {
            var token = ctx.Request.Cookies[RefreshTokenCookie];  // Legge cookie
            if (string.IsNullOrEmpty(token))
                return Results.Unauthorized();

            var res = await mediator.Send(new RefreshTokenCommand(token), ct);
            SetRefreshCookie(ctx, res.RefreshToken, config);  // Ruota cookie
            return Results.Ok(new { res.AccessToken, res.ExpiresAt, ... });
        })
        .AllowAnonymous()
        .RequireRateLimiting("auth-policy");

        // ... altri endpoint
    }

    private static void SetRefreshCookie(HttpContext ctx, string refreshToken, IConfiguration config)
    {
        var days = int.Parse(config["Auth:Jwt:RefreshExpiresInDays"] ?? "7");
        ctx.Response.Cookies.Append(RefreshTokenCookie, refreshToken, new CookieOptions
        {
            HttpOnly = true,           // Non accessibile da JavaScript (XSS protection)
            Secure = !isDevelopment,   // HTTPS only in production
            SameSite = SameSiteMode.Lax, // CSRF protection
            Path = "/api/auth",
            MaxAge = TimeSpan.FromDays(days)
        });
    }
}
```

**Flusso**:
1. Riceve comando (email + password)
2. Invia a MediatR → pipeline (logging, validazione, handler)
3. Handler ritorna response con token
4. Endpoint setta il cookie (`SetRefreshCookie`)
5. Restituisce JSON con accessToken

---

### 2. **LoginHandler.cs** — La Logica di Autenticazione

```csharp
// File: Features/Auth/Login/LoginHandler.cs
public class LoginHandler(
    IUserRepository userRepo,
    IGroupRepository groupRepo,
    IRoleRepository roleRepo,
    IPermissionRepository permRepo,
    IRefreshTokenRepository refreshRepo,
    IConfiguration configuration
) : IRequestHandler<LoginCommand, LoginResponse>
{
    public async Task<LoginResponse> Handle(LoginCommand request, CancellationToken cancellationToken)
    {
        // STEP 1: Cerca utente per email
        var user = await userRepo.GetByEmailAsync(request.Email, cancellationToken)
            ?? throw new UnauthorizedException("Credenziali non valide.");

        // STEP 2: Verifica che l'account sia attivo
        if (!user.IsActive)
            throw new UnauthorizedException("Account disabilitato.");

        // STEP 3: Verifica password (BCrypt — one-way hash)
        if (!BCrypt.Net.BCrypt.Verify(request.Password, user.PasswordHash))
            throw new UnauthorizedException("Credenziali non valide.");
            // Nota: messaggio di errore generico per security (no enumeration attack)

        // STEP 4: Risolvi permessi (diretti + ereditati da gruppi)
        var permissions = await ResolvePermissionsAsync(user, cancellationToken);

        // STEP 5: Genera JWT
        var (accessToken, expiresAt) = JwtTokenHelper.GenerateAccessToken(
            user, permissions, configuration
        );

        // STEP 6: Crea refresh token (salva in DB)
        var refreshToken = await JwtTokenHelper.CreateRefreshTokenAsync(
            user.Id, refreshRepo, configuration, cancellationToken
        );

        // STEP 7: Ritorna response
        return new LoginResponse(
            accessToken, refreshToken, expiresAt,
            user.Id.ToString(), user.Email, user.DisplayName,
            user.Language, permissions
        );
    }

    private async Task<List<string>> ResolvePermissionsAsync(UserDocument user, CancellationToken ct)
    {
        // Raccoglie ruoli DIRETTI (assegnati all'utente)
        var directRoleIds = user.RoleIds.ToList();

        // Raccoglie ruoli EREDITATI dai gruppi
        var groupRoleIds = new List<ObjectId>();
        if (user.GroupIds.Any())
        {
            var groups = await groupRepo.GetByIdsAsync(user.GroupIds, ct);
            groupRoleIds = groups.SelectMany(g => g.RoleIds).ToList();
        }

        // Unisce e deduplica
        var allRoleIds = directRoleIds.Concat(groupRoleIds).Distinct().ToList();

        // Risolve i ruoli → permessi
        var roles = await roleRepo.GetByIdsAsync(allRoleIds, ct);
        var permissionIds = roles.SelectMany(r => r.PermissionIds).Distinct().ToList();
        var permissions = await permRepo.GetByIdsAsync(permissionIds, ct);

        // Ritorna i KEY dei permessi (es. "users.read", "roles.manage")
        return permissions.Select(p => p.Key).Distinct().ToList();
    }
}
```

**Sicurezza**:
- ✅ Password verificata con **BCrypt** (one-way hash, slow)
- ✅ Messaggio di errore generico ("Credenziali non valide") — no info leakage
- ✅ Verifica `IsActive` — previene accesso account disabilitati
- ✅ Permessi risolti in real-time — cambamenti immediati dopo login

---

### 3. **JwtTokenHelper.cs** — Generazione Token

```csharp
// File: Features/Auth/Shared/JwtTokenHelper.cs
public static class JwtTokenHelper
{
    public static (string Token, DateTime ExpiresAt) GenerateAccessToken(
        UserDocument user,
        List<string> permissions,
        IConfiguration configuration)
    {
        // STEP 1: Leggi configurazione
        var jwtSection = configuration.GetSection("Auth:Jwt");
        var secret = jwtSection["Secret"]!;           // Secret key (min 256 bit)
        var issuer = jwtSection["Issuer"] ?? "Tama";
        var audience = jwtSection["Audience"] ?? "TamaClients";
        var expiresInMinutes = int.Parse(jwtSection["ExpiresInMinutes"] ?? "60");

        // STEP 2: Crea la chiave di firma (HS256)
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secret));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        // STEP 3: Calcola scadenza
        var expiresAt = DateTime.UtcNow.AddMinutes(expiresInMinutes);

        // STEP 4: Crea i CLAIMS (i dati nel token)
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id.ToString()),  // Subject: ID utente
            new(JwtRegisteredClaimNames.Email, user.Email),        // Email
            new("name", user.DisplayName),                         // Display name
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),  // Unique ID
        };

        // Aggiunge ogni permesso come claim separato
        claims.AddRange(permissions.Select(p => new Claim("permissions", p)));
        // Risultato: { "permissions": ["users.read", "roles.manage"] }

        // STEP 5: Crea il JWT
        var token = new JwtSecurityToken(
            issuer: issuer,
            audience: audience,
            claims: claims,
            expires: expiresAt,
            signingCredentials: creds);

        // STEP 6: Serializza in string
        return (new JwtSecurityTokenHandler().WriteToken(token), expiresAt);
        // Ritorna: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWI..."
    }

    public static async Task<string> CreateRefreshTokenAsync(
        ObjectId userId,
        IRefreshTokenRepository refreshRepo,
        IConfiguration configuration,
        CancellationToken ct)
    {
        // STEP 1: Leggi scadenza dal config
        var daysValid = int.Parse(configuration["Auth:Jwt:RefreshExpiresInDays"] ?? "7");

        // STEP 2: Genera token casuale (64 byte = 512 bit, molto sicuro)
        var tokenValue = Convert.ToBase64String(RandomNumberGenerator.GetBytes(64));

        // STEP 3: Crea documento MongoDB
        var doc = new RefreshTokenDocument
        {
            Id = ObjectId.GenerateNewId(),
            UserId = userId,
            Token = tokenValue,  // Il valore casuale
            ExpiresAt = DateTime.UtcNow.AddDays(daysValid),
            CreatedAt = DateTime.UtcNow
        };

        // STEP 4: Salva nel database
        await refreshRepo.InsertAsync(doc, ct);

        // STEP 5: Ritorna il token (il client lo riceverà nel cookie)
        return tokenValue;
    }
}
```

**Dettagli tecnici**:

#### Claims nel JWT

Quando il JWT viene decodificato, il payload contiene questi claims:

```json
{
  "sub": "65fd30ab12345c0001abcdef",        // ObjectId dell'utente
  "email": "mario@example.com",
  "name": "Mario Rossi",
  "permissions": ["users.read", "roles.manage"],
  "jti": "550e8400-e29b-41d4-a716-446655440000",  // Unique ID
  "iat": 1695312000,                         // Issued at
  "exp": 1695315600,                         // Expiration (60 min dopo)
  "iss": "Tama",
  "aud": "TamaClients"
}
```

#### Firma (HMAC-SHA256)

```
HMAC-SHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret_key
)
```

Il server conosce il `secret_key` → può verificare che nessuno ha modificato il token.

---

### 4. **RefreshTokenHandler.cs** — Rinnovamento Token

```csharp
// File: Features/Auth/RefreshToken/RefreshTokenHandler.cs
public class RefreshTokenHandler(
    IRefreshTokenRepository refreshRepo,
    IUserRepository userRepo,
    IGroupRepository groupRepo,
    IRoleRepository roleRepo,
    IPermissionRepository permRepo,
    IConfiguration configuration
) : IRequestHandler<RefreshTokenCommand, RefreshTokenResponse>
{
    public async Task<RefreshTokenResponse> Handle(
        RefreshTokenCommand request, CancellationToken cancellationToken)
    {
        // STEP 1: Trova il refresh token nel DB
        var existing = await refreshRepo.GetByTokenAsync(request.Token, cancellationToken)
            ?? throw new UnauthorizedException("Refresh token non valido o scaduto.");

        // STEP 2: Verifica che non sia scaduto
        if (existing.ExpiresAt < DateTime.UtcNow)
            throw new UnauthorizedException("Refresh token scaduto.");

        // STEP 3: Trova l'utente
        var user = await userRepo.GetByIdAsync(existing.UserId, cancellationToken)
            ?? throw new UnauthorizedException("Utente non trovato.");

        // STEP 4: Verifica che sia attivo
        if (!user.IsActive)
            throw new UnauthorizedException("Account disabilitato.");

        // STEP 5: REVOCA il vecchio token (Token Rotation — security best practice)
        await refreshRepo.RevokeAsync(existing.Id, cancellationToken);
        // Nota: il vecchio token nel cookie del client diventa inutile

        // STEP 6-8: Risolvi permessi (stesso codice di LoginHandler)
        var directRoleIds = user.RoleIds.ToList();
        var groupRoleIds = new List<ObjectId>();
        if (user.GroupIds.Any())
        {
            var groups = await groupRepo.GetByIdsAsync(user.GroupIds, cancellationToken);
            groupRoleIds = groups.SelectMany(g => g.RoleIds).ToList();
        }

        var allRoleIds = directRoleIds.Concat(groupRoleIds).Distinct().ToList();
        var roles = await roleRepo.GetByIdsAsync(allRoleIds, cancellationToken);
        var permIds = roles.SelectMany(r => r.PermissionIds).Distinct().ToList();
        var perms = await permRepo.GetByIdsAsync(permIds, cancellationToken);
        var permKeys = perms.Select(p => p.Key).ToList();

        // STEP 9: Genera nuovo JWT
        var (accessToken, expiresAt) = JwtTokenHelper.GenerateAccessToken(
            user, permKeys, configuration);

        // STEP 10: Crea nuovo refresh token
        var newRefresh = await JwtTokenHelper.CreateRefreshTokenAsync(
            user.Id, refreshRepo, configuration, cancellationToken);

        // STEP 11: Ritorna
        return new RefreshTokenResponse(accessToken, newRefresh, expiresAt, permKeys);
    }
}
```

**Token Rotation**:

```
Timeline:

1. Login
   ├─ Create RT-1 → Send to client
   └─ RT-1 valid ✓

2. Time passes...

3. Client calls /refresh
   ├─ Revoke RT-1 → RT-1 invalid ✗
   ├─ Create RT-2 → Send to client
   └─ RT-2 valid ✓

4. If attacker steals and tries to use RT-1
   → "Refresh token not found" (revoked)
   → Request fails
```

---

### 5. **MsalLoginHandler.cs** — Azure AD Integration

```csharp
// File: Features/Auth/MsalLogin/MsalLoginHandler.cs
public class MsalLoginHandler(...) : IRequestHandler<MsalLoginCommand, MsalLoginResponse>
{
    public async Task<MsalLoginResponse> Handle(
        MsalLoginCommand request, CancellationToken cancellationToken)
    {
        // STEP 1: Parsa il token Azure AD
        var handler = new JwtSecurityTokenHandler();
        if (!handler.CanReadToken(request.AzureToken))
            throw new UnauthorizedException("Token Azure non valido.");

        var jwt = handler.ReadJwtToken(request.AzureToken);

        // STEP 2: Estrai l'email dal token Azure
        // Nota: Azure usa "preferred_username" o "email"
        var email = jwt.Claims.FirstOrDefault(c =>
            c.Type == "preferred_username" || c.Type == "email")?.Value
            ?? throw new UnauthorizedException("Claim 'email' assente nel token Azure.");

        // STEP 3: Trova l'utente LOCALE per email
        var user = await userRepo.GetByEmailAsync(email, cancellationToken)
            ?? throw new UnauthorizedException($"Nessun utente locale con email '{email}'.");
        // Nota: L'utente deve essere creato preventivamente nel sistema

        // STEP 4: Verifica che sia attivo
        if (!user.IsActive)
            throw new UnauthorizedException("Account disabilitato.");

        // STEP 5-7: Risolvi permessi (stesso di Login e Refresh)
        var directRoleIds = user.RoleIds.ToList();
        var groupRoleIds = new List<ObjectId>();
        if (user.GroupIds.Any())
        {
            var groups = await groupRepo.GetByIdsAsync(user.GroupIds, cancellationToken);
            groupRoleIds = groups.SelectMany(g => g.RoleIds).ToList();
        }

        var allRoleIds = directRoleIds.Concat(groupRoleIds).Distinct().ToList();
        var roles = await roleRepo.GetByIdsAsync(allRoleIds, cancellationToken);
        var permIds = roles.SelectMany(r => r.PermissionIds).Distinct().ToList();
        var perms = await permRepo.GetByIdsAsync(permIds, cancellationToken);
        var permKeys = perms.Select(p => p.Key).ToList();

        // STEP 8-9: Genera JWT interno + refresh token
        var (accessToken, expiresAt) = JwtTokenHelper.GenerateAccessToken(
            user, permKeys, configuration);
        var refreshToken = await JwtTokenHelper.CreateRefreshTokenAsync(
            user.Id, refreshRepo, configuration, cancellationToken);

        return new MsalLoginResponse(
            accessToken, refreshToken, expiresAt,
            user.Id.ToString(), user.Email, user.DisplayName,
            user.Language, permKeys);
    }
}
```

**Flusso**:
1. Frontend riceve token da Azure AD (validato dalla libreria `@azure/msal-browser`)
2. Invia idToken al backend
3. Backend legge il token Azure (parsing JWT, senza rivalidare la firma)
4. Backend estrae email dal claim `preferred_username` o `email`
5. Backend cerca utente locale per email
6. Genera JWT interno (nello stesso formato del login classico)

---

### 6. **LogoutHandler.cs** — Revoca Token

```csharp
// File: Features/Auth/Logout/LogoutHandler.cs
public class LogoutHandler(IRefreshTokenRepository refreshRepo)
    : IRequestHandler<LogoutCommand>
{
    public async Task Handle(LogoutCommand request, CancellationToken cancellationToken)
    {
        // Cerca il token nel DB
        var token = await refreshRepo.GetByTokenAsync(request.RefreshToken, cancellationToken);

        // Se trovato, revoca (non lancia errore se non trovato)
        if (token is not null)
            await refreshRepo.RevokeAsync(token.Id, cancellationToken);
        // Nota: implementazione idempotente (safe da chiamare più volte)
    }
}
```

---

## 🗄️ MongoDB Documents

### RefreshTokenDocument

```csharp
public class RefreshTokenDocument
{
    public ObjectId Id { get; set; }              // ID MongoDB
    public ObjectId UserId { get; set; }          // Riferimento all'utente
    public string Token { get; set; } = string.Empty;  // Token casuale (64 byte)
    public DateTime ExpiresAt { get; set; }       // Scadenza
    public DateTime CreatedAt { get; set; }       // Quando creato
    public DateTime? RevokedAt { get; set; }      // Null = valido, Date = revocato
}
```

**Collection MongoDB**: `refreshTokens`

**Indici suggeriti**:
- `Token` (unique, sparse) — ricerca rapida
- `UserId` — find all tokens di un utente
- `ExpiresAt` — cleanup token scaduti (TTL index)

---

# Configurazione

## appsettings.json

```json
{
  "Auth": {
    "Jwt": {
      "Secret": "your-super-secret-key-at-least-32-characters-long-!!!",
      "Issuer": "Tama",
      "Audience": "TamaClients",
      "ExpiresInMinutes": 60,
      "RefreshExpiresInDays": 7
    }
  }
}
```

| Parametro | Significato | Note |
|---|---|---|
| **Secret** | Chiave per firmare JWT | Min 32 caratteri. Generare casuale. **Non mettere in git!** |
| **Issuer** | Chi ha emesso il token | Valore logico, non importante |
| **Audience** | A chi è destinato il token | Identifica i client autorizzati |
| **ExpiresInMinutes** | Scadenza del JWT | Default 60. Valori comuni: 15-60 minuti |
| **RefreshExpiresInDays** | Scadenza del refresh token | Default 7. Controllo quanto a lungo l'utente rimane loggato |

### Generare un Secret Sicuro

**PowerShell**:
```powershell
[Convert]::ToBase64String((1..32 | ForEach-Object { [byte](Get-Random -Max 256) }))
```

**Bash**:
```bash
openssl rand -base64 32
```

### appsettings.local.json (Development)

```json
{
  "Auth": {
    "Jwt": {
      "Secret": "dev-secret-key-not-for-production-12345"
    }
  }
}
```

**IMPORTANTE**: `.local.json` è in `.gitignore` → secrets locali non finiscono in git.

---

## Program.cs — Registrazione Servizi

```csharp
// Nel Program.cs:

// Carica appsettings.local.json (gitignored) per i segreti locali
builder.Configuration.AddJsonFile("appsettings.local.json", optional: true, reloadOnChange: true);

// Autenticazione: JWT oppure MSAL in base a appsettings.json Auth:Strategy
builder.Services.AddAppAuthentication(builder.Configuration);

// Autorizzazione con policy per permessi
builder.Services.AddAppAuthorization();

// MediatR + FluentValidation + pipeline behaviors
builder.Services.AddMediatRWithBehaviors();

// Rate limiting — protezione brute-force sugli endpoint di autenticazione
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("auth-policy", limiter =>
    {
        limiter.Window = TimeSpan.FromMinutes(15);
        limiter.PermitLimit = 10;
    });
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

// CORS con AllowCredentials (necessario per cookie refresh token)
builder.Services.AddCors(options =>
{
    policy.AllowAnyHeader()
          .AllowAnyMethod()
          .AllowCredentials(); // Necessario per cookie HttpOnly
});
```

**Dettaglio registrazione autenticazione** (`Features/_Shared/Extensions/AuthExtensions.cs`):

```csharp
public static class AuthExtensions
{
    public static IServiceCollection AddAppAuthentication(
        this IServiceCollection services, IConfiguration configuration)
    {
        var jwtSection = configuration.GetSection("Auth:Jwt");
        var secret = jwtSection["Secret"]
            ?? throw new InvalidOperationException(
                "JWT Secret is missing. Set it in appsettings.local.json.");

        services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(options =>
            {
                options.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuerSigningKey = true,
                    IssuerSigningKey = new SymmetricSecurityKey(
                        Encoding.UTF8.GetBytes(secret)),
                    ValidateIssuer = true,
                    ValidIssuer = jwtSection["Issuer"] ?? "Tama",
                    ValidateAudience = true,
                    ValidAudience = jwtSection["Audience"] ?? "TamaClients",
                    ValidateLifetime = true,
                    ClockSkew = TimeSpan.Zero  // Nessuna tolleranza sulla scadenza
                };
            });
        return services;
    }

    public static IServiceCollection AddAppAuthorization(
        this IServiceCollection services)
    {
        services.AddAuthorization();
        return services;
    }
}
```

---

# Esempi di Request/Response

## 1. Login

### Request
```http
POST /api/auth/login HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "email": "mario@example.com",
  "password": "SecurePassword123!"
}
```

### Response (200 OK)
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI2NWZkMzBhYjEyMzQ1YzAwMDFhYmNkZWYiLCJlbWFpbCI6Im1hcmlvQGV4YW1wbGUuY29tIiwibmFtZSI6Ik1hcmlvIFJvc3NpIiwicGVybWlzc2lvbnMiOlsidXNlcnMucmVhZCIsInJvbGVzLm1hbmFnZSJdLCJpYXQiOjE2OTUzMTIwMDAsImV4cCI6MTY5NTMxNTYwMH0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c",
  "expiresAt": "2026-03-29T11:30:00Z",
  "userId": "65fd30ab12345c0001abcdef",
  "email": "mario@example.com",
  "displayName": "Mario Rossi",
  "language": "it",
  "permissions": ["users.read", "roles.manage"]
}
```

Set-Cookie Header:
```
Set-Cookie: refreshToken=abcd1234efgh5678ijkl9012mnop3456qrst7890uvwx=; Path=/api/auth; Max-Age=604800; HttpOnly; Secure; SameSite=Lax
```

### Response (401 Unauthorized)
```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "Unauthorized",
  "status": 401,
  "detail": "Credenziali non valide."
}
```

---

## 2. Accesso Protetto

### Request
```http
GET /api/users HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWI...
```

### Response (200 OK)
```json
[
  {
    "id": "65fd30ab12345c0001abcdef",
    "email": "mario@example.com",
    "displayName": "Mario Rossi",
    "createdAt": "2025-01-15T10:00:00Z"
  }
]
```

### Response (401 Unauthorized) — Token scaduto
```json
{
  "type": "https://tools.ietf.org/html/rfc7235#section-3.1",
  "title": "Unauthorized",
  "status": 401,
  "detail": "Token expired"
}
```

---

## 3. Refresh Token

### Request
```http
POST /api/auth/refresh HTTP/1.1
Host: api.example.com
Cookie: refreshToken=abcd1234efgh5678ijkl9012mnop3456qrst7890uvwx=
```

### Response (200 OK)
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI2NWZkMzBhYjEyMzQ1YzAwMDFhYmNkZWYiLCJlbWFpbCI6Im1hcmlvQGV4YW1wbGUuY29tIiwibmFtZSI6Ik1hcmlvIFJvc3NpIiwicGVybWlzc2lvbnMiOlsidXNlcnMucmVhZCIsInJvbGVzLm1hbmFnZSJdLCJpYXQiOjE2OTUzMTU2MDAsImV4cCI6MTY5NTMxOTIwMH0.xxxxx",
  "expiresAt": "2026-03-29T12:30:00Z",
  "permissions": ["users.read", "roles.manage"]
}
```

Set-Cookie Header (nuovo token):
```
Set-Cookie: refreshToken=new_abcd1234efgh5678ijkl9012mnop3456qrst7890uvwx=; Path=/api/auth; Max-Age=604800; HttpOnly; Secure; SameSite=Lax
```

### Response (401 Unauthorized) — Token scaduto/revocato
```json
{
  "type": "https://tools.ietf.org/html/rfc7235#section-3.1",
  "title": "Unauthorized",
  "status": 401,
  "detail": "Refresh token non valido o scaduto."
}
```

---

## 4. Logout

### Request
```http
POST /api/auth/logout HTTP/1.1
Host: api.example.com
Cookie: refreshToken=abcd1234efgh5678ijkl9012mnop3456qrst7890uvwx=
```

### Response (204 No Content)
```
(empty body)
```

Set-Cookie Header (elimina cookie):
```
Set-Cookie: refreshToken=; Path=/api/auth; Max-Age=0; HttpOnly; Secure; SameSite=Lax
```

---

## 5. MSAL Login

### Request
```http
POST /api/auth/msal-login HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "azureToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6Ik1pMnZ..."
}
```

### Response (200 OK)
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresAt": "2026-03-29T11:30:00Z",
  "userId": "65fd30ab12345c0001abcdef",
  "email": "mario@example.com",
  "displayName": "Mario Rossi",
  "language": "it",
  "permissions": ["users.read", "roles.manage"]
}
```

---

## Decodificare un JWT (online)

Usa **jwt.io**:

1. Copia il token da `accessToken`
2. Incolla su https://jwt.io
3. Vedi il payload decodificato
4. Verifica che sia valido

**Attenzione**: Non mettere mai token con dati sensibili su siti pubblici!

---

# Sicurezza

## ✅ Cosa è Implementato

### 1. **HTTPS Only (Production)**
```csharp
// AuthEndpoints.cs:92
Secure = !ctx.RequestServices.GetRequiredService<IWebHostEnvironment>().IsDevelopment(),
```
Il cookie è inviato solo su HTTPS in produzione.

---

### 2. **HttpOnly Cookie**
```csharp
HttpOnly = true,  // JavaScript NON può accedere (XSS protection)
```
Il refresh token è in un cookie HttpOnly → JavaScript non può rubarlo neanche se il sito è hackerato (XSS).

---

### 3. **SameSite=Lax**
```csharp
SameSite = SameSiteMode.Lax,  // CSRF protection
```
Il browser non invia il cookie in richieste cross-site → protegge da CSRF.

---

### 4. **Token Rotation**
```csharp
// RefreshTokenHandler.cs:31
await refreshRepo.RevokeAsync(existing.Id, cancellationToken);
```
Ogni volta che il token viene usato, ricevi uno nuovo e quello vecchio viene revocato.

**Beneficio**: Se il token viene rubato, l'attacker può usarlo solo una volta.

---

### 5. **BCrypt Password Hashing**
```csharp
// LoginHandler.cs:26
if (!BCrypt.Net.BCrypt.Verify(request.Password, user.PasswordHash))
```
Le password sono hashate con BCrypt (slow, con salt) → impossibile recovarle da un hash.

---

### 6. **JWT Signature Verification**
```csharp
// JwtTokenHelper.cs:25-26
var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secret));
var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
```
Ogni JWT è firmato con HMAC-SHA256 → il server può verificare che non sia stato modificato.

---

### 7. **Rate Limiting**
```csharp
// AuthEndpoints.cs:24, 40, 59
.RequireRateLimiting("auth-policy")
```
Login, msal-login, e refresh hanno rate limiting → protezione da brute-force.

---

### 8. **Messaggi di Errore Generici**
```csharp
// LoginHandler.cs:20, 27
throw new UnauthorizedException("Credenziali non valide.");
```
Non si specifica se l'email non esiste o la password è sbagliata → protegge da enumeration attack.

---

### 9. **Account Disabilitati**
```csharp
// LoginHandler.cs:22-23
if (!user.IsActive)
    throw new UnauthorizedException("Account disabilitato.");
```
Se un account viene disabilitato, l'utente non può più accedere neanche con credenziali valide.

---

### 10. **Token Expiration**
```csharp
// JwtTokenHelper.cs:27
var expiresAt = DateTime.UtcNow.AddMinutes(expiresInMinutes);
```
Il JWT ha una scadenza breve (default 60 minuti) → minimizza il danno se rubato.

---

## ⚠️ Considerazioni e Best Practices

### 1. **Secret Management**
```
❌ SBAGLIATO: "secret123" in appsettings.json
✅ GIUSTO: Generare casuale, min 32 char, conservare in Azure Key Vault / Secrets Manager
```

### 2. **HTTPS in Produzione**
```
❌ SBAGLIATO: HTTP
✅ GIUSTO: HTTPS sempre (i cookie insicuri vengono bloccati)
```

### 3. **Scadenza del JWT**
```
❌ SBAGLIATO: ExpiresInMinutes = 86400 (24 ore, troppo lungo)
✅ GIUSTO: 15-60 minuti (usa refresh token per rinnovare)
```

### 4. **Logging Sicuro**
```csharp
❌ SBAGLIATO: logger.LogInformation($"Login per {user.Password}");
✅ GIUSTO: logger.LogInformation("Login successful");
```
Non loggare mai dati sensibili (password, token).

### 5. **Validazione Input**
```csharp
// LoginCommand usa FluentValidation (vedi LoginValidator.cs)
// Email deve essere valida
// Password deve avere lunghezza minima
```

---

# Troubleshooting

## ❌ Errore: "401 Unauthorized — Token expired"

**Causa**: Il JWT è scaduto (oltre 60 minuti).

**Soluzione**:
Il frontend (Axios interceptor in `plugins/axios.ts`) intercetta automaticamente le risposte 401 e chiama `authStore.logout()` per disconnettere l'utente. Per un'esperienza più fluida, il client può chiamare `authStore.refresh()` preventivamente prima della scadenza del token.

```typescript
// plugins/axios.ts — già implementato
api.interceptors.response.use(
  res => res,
  async error => {
    if (error.response?.status === 401 && !isLoggingOut) {
      isLoggingOut = true
      try { await authStore.logout() }
      finally { isLoggingOut = false }
    }
    return Promise.reject(error)
  }
)
```

---

## ❌ Errore: "401 Unauthorized — Token invalid"

**Cause possibili**:
1. Token modificato
2. Algoritmo/Secret diverso tra client e server
3. Token da server diverso

**Soluzione**:
- Verificare che il secret sia lo stesso su tutti i server
- Verificare che l'algoritmo sia HS256
- Controllare i log del server

---

## ❌ Errore: "401 Unauthorized — Refresh token not found"

**Cause**:
1. Cookie non inviato (verifica SameSite, Secure)
2. Token revocato (logout, rotation)
3. Token scaduto

**Soluzione**:
```bash
# Verifica che il cookie sia presente
curl -v http://localhost:5000/api/auth/refresh
# Cerca: "Cookie:" negli header
```

---

## ❌ Errore: "400 Bad Request — Validation failed"

**Causa**: Email o password non valida.

**Soluzione**:
```json
// Verificare il payload
{
  "email": "mario@example.com",  // deve essere email valida
  "password": "SecurePassword123!"  // minimo 8 char, numero, maiuscola?
}
```

---

## ❌ Errore: "401 Unauthorized — Account disabilitato"

**Causa**: L'account è stato disabilitato (IsActive = false).

**Soluzione**:
- Contattare un admin per riabilitare l'account
- Oppure: verificare il documento User in MongoDB

```javascript
db.users.updateOne(
  { email: "mario@example.com" },
  { $set: { isActive: true } }
)
```

---

## ❌ Errore: "No refresh token in cookie"

**Causa**: Il browser non sta inviando il cookie.

**Controllo**: Browser DevTools → Application → Cookies

```
Path: /api/auth
HttpOnly: ✓ (non visibile in console)
Secure: ✓ (solo HTTPS)
SameSite: Lax
```

**Soluzione**:
```javascript
// Frontend: assicurati di impostare credentials
fetch('/api/auth/refresh', {
  credentials: 'include'  // Invia cookie automaticamente
})
```

---

## ❌ Errore: "JWT signature is invalid"

**Causa**: Secret diverso tra signing e verification.

**Soluzione**:
```csharp
// Verificare che il secret sia identico
var secret = configuration["Auth:Jwt:Secret"];
// Deve essere lo stesso in GenerateAccessToken e JwtBearerOptions
```

---

## ✅ Debug Checklist

```
□ Token è presente in response?
□ Token è valido (decodifica su jwt.io)?
□ Token è nel header Authorization: Bearer {token}?
□ Secret è lo stesso su tutti i server?
□ Server clock è sincronizzato (exp check)?
□ Scadenza è nel futuro?
□ Issuer e Audience sono corretti?
□ Refresh token è nel cookie?
□ Cookie è HttpOnly?
□ Sito è su HTTPS (o localhost development)?
```

---

## Comandi MongoDB Utili

```javascript
// Vedere tutti i refresh token di un utente
db.refreshTokens.find({
  userId: ObjectId("65fd30ab12345c0001abcdef")
})

// Revocare manualmente un token
db.refreshTokens.updateOne(
  { _id: ObjectId("...") },
  { $set: { revokedAt: new Date() } }
)

// Elimina token scaduti (cleanup)
db.refreshTokens.deleteMany({
  expiresAt: { $lt: new Date() }
})

// Vedi il token più recente
db.refreshTokens
  .find({ userId: ObjectId("...") })
  .sort({ createdAt: -1 })
  .limit(1)
```

---

## Log di Debug (abilitare livello Trace)

```json
// appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Tama.Api.Features.Auth": "Debug",
      "Microsoft.IdentityModel.Tokens": "Debug"
    }
  }
}
```

Poi controllare i log:
```
[14:32:00 INF] Handling login for mario@example.com
[14:32:00 DBG] User found: 65fd30ab12345c0001abcdef
[14:32:00 DBG] Password verified: OK
[14:32:00 DBG] Permissions resolved: ["users.read", "roles.manage"]
[14:32:00 DBG] Access token generated
[14:32:00 DBG] Refresh token saved
```

---

## Implementazione Frontend (Effettiva)

Il frontend utilizza **Axios** con interceptor automatici, **Pinia** per lo stato, e i servizi dedicati.

### Auth Service (`services/auth.service.ts`)

```typescript
import { api } from '@/plugins/axios'
import type { LoginRequest, LoginResponse } from '@/types/auth.types'

export const authService = {
  login: (data: LoginRequest) =>
    api.post<LoginResponse>('/api/auth/login', data).then(r => r.data),

  msalLogin: (azureToken: string) =>
    api.post<LoginResponse>('/api/auth/msal-login', { azureToken }).then(r => r.data),

  refresh: () =>
    api.post<{ accessToken: string; expiresAt: string; permissions: string[] }>(
      '/api/auth/refresh'
    ).then(r => r.data),

  logout: () =>
    api.post('/api/auth/logout'),
}
```

### Axios Interceptor (`plugins/axios.ts`)

```typescript
export const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  headers: { 'Content-Type': 'application/json' },
  withCredentials: true,  // Invia/ricevi cookies automaticamente
})

// Aggiunge Bearer token ad ogni richiesta
api.interceptors.request.use(config => {
  const authStore = useAuthStore()
  if (authStore.token) {
    config.headers.Authorization = `Bearer ${authStore.token}`
  }
  config.headers['Accept-Language'] = authStore.language ?? 'it'
  return config
})

// Gestisce 401 → logout automatico (NON retry con refresh)
let isLoggingOut = false
api.interceptors.response.use(
  res => res,
  async error => {
    if (error.response?.status === 401 && !isLoggingOut) {
      isLoggingOut = true
      try { await useAuthStore().logout() }
      finally { isLoggingOut = false }
    }
    return Promise.reject(error)
  }
)
```

> **Nota**: Su 401, il frontend effettua il logout automatico dell'utente (non il retry con refresh). La funzione `authStore.refresh()` è disponibile per rinnovo preventivo ma non è usata automaticamente su 401.

### Auth Store (`stores/auth.store.ts`)

```typescript
// Stato persistito in localStorage:
// token, expiresAt, userId, email, displayName, language, permissions

// Azioni principali:
login(credentials)        // POST /api/auth/login → salva sessione
loginWithMsal(azureToken) // POST /api/auth/msal-login → salva sessione
logout()                  // POST /api/auth/logout → pulisce sessione
refresh()                 // POST /api/auth/refresh → aggiorna token
applyLanguage(lang)       // Cambia lingua i18n in tempo reale
```

### LoginPage.vue — Logica di Switch Strategia

```typescript
const authStrategy = import.meta.env.VITE_AUTH_STRATEGY

// Se authStrategy !== 'msal': mostra form email/password
// Se authStrategy === 'msal': mostra bottone "Login con Microsoft"
// Se ritorno da redirect MSAL: chiama msalService.handleRedirectResult()
```

### Router Guard (`router/index.ts`)

```typescript
router.beforeEach(async (to) => {
  const authStore = useAuthStore()

  // Rotta protetta senza autenticazione → redirect a /login?returnUrl=...
  if (to.meta.requiresAuth && !authStore.isAuthenticated) {
    return { name: 'login', query: { returnUrl: to.fullPath } }
  }

  // Permesso mancante → redirect a dashboard con toast warning
  if (to.meta.permission) {
    if (!authStore.permissions.includes(to.meta.permission as string)) {
      toastStore.warning(i18n.global.t('common.forbidden'))
      return { name: 'dashboard' }
    }
  }
})
```

---

**End of Documentation**

---

## 📚 File Correlati nel Progetto

| File | Scopo |
|---|---|
| [AuthEndpoints.cs](apps/backend/Tama.Api/Features/Auth/AuthEndpoints.cs) | Routing e endpoint |
| [LoginHandler.cs](apps/backend/Tama.Api/Features/Auth/Login/LoginHandler.cs) | Logica login |
| [LoginCommand.cs](apps/backend/Tama.Api/Features/Auth/Login/LoginCommand.cs) | Input login |
| [LoginResponse.cs](apps/backend/Tama.Api/Features/Auth/Login/LoginResponse.cs) | Output login |
| [RefreshTokenHandler.cs](apps/backend/Tama.Api/Features/Auth/RefreshToken/RefreshTokenHandler.cs) | Logica refresh |
| [RefreshTokenCommand.cs](apps/backend/Tama.Api/Features/Auth/RefreshToken/RefreshTokenCommand.cs) | Input refresh |
| [RefreshTokenResponse.cs](apps/backend/Tama.Api/Features/Auth/RefreshToken/RefreshTokenResponse.cs) | Output refresh |
| [MsalLoginHandler.cs](apps/backend/Tama.Api/Features/Auth/MsalLogin/MsalLoginHandler.cs) | Logica Azure AD |
| [LogoutHandler.cs](apps/backend/Tama.Api/Features/Auth/Logout/LogoutHandler.cs) | Logica logout |
| [JwtTokenHelper.cs](apps/backend/Tama.Api/Features/Auth/Shared/JwtTokenHelper.cs) | Utility JWT |
| [AuthExtensions.cs](apps/backend/Tama.Api/Features/_Shared/Extensions/AuthExtensions.cs) | Registrazione JWT + configurazione auth |

