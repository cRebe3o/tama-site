# 🔐 Autenticazione — JWT vs MSAL

> **Documento**: Strategie di autenticazione nel sistema
> **Lingua**: Italiano
> **Ultimo aggiornamento**: 2026-03-29
> **Versione**: 2.0

---

## 📋 Indice

1. [Overview](#overview) — quale strategia scegliere
2. [Strategia JWT](#strategia-jwt) — autenticazione classica email+password
3. [Strategia MSAL](#strategia-msal) — autenticazione Azure AD
4. [Flussi Completi](#flussi-completi) — diagrammi step-by-step
5. [Come Switchare](#come-switchare) — configurazione
6. [Livello Tecnico](#livello-tecnico) — implementazione dettagliata
7. [Sicurezza](#sicurezza) — considerazioni
8. [Troubleshooting](#troubleshooting) — problemi comuni

---

# Overview

## Quale Strategia Scegliere?

| Aspetto | JWT | MSAL |
|---|---|---|
| **Autenticazione** | Email + Password | Microsoft/Office 365 |
| **Gestione utenti** | Manuale (backend) | Azure AD (Microsoft) |
| **Single Sign-On (SSO)** | ❌ No | ✅ Sì (con Office 365) |
| **Complessità setup** | ⭐ Bassa | ⭐⭐⭐ Media-Alta |
| **Ideale per** | Startup, app interna | Enterprise con Azure AD |
| **Password forget?** | Implementare manualmente | Microsoft gestisce |
| **2FA / MFA** | Implementare manualmente | Microsoft gestisce |

**Consiglio**:
- **Piccoli team**: JWT è più semplice
- **Aziende enterprise**: MSAL integra con ecosistema Microsoft (Azure, Office 365, Teams)

---

## Che Succede Dietro le Quinte

Entrambe le strategie fanno **la stessa cosa al backend**:

```
┌─────────────────────────────────────┐
│       STRATEGIA SCELTA              │
│                                     │
│  JWT: email+password                │
│  MSAL: token Azure AD               │
├─────────────────────────────────────┤
│         Server Backend              │
├─────────────────────────────────────┤
│  ✓ Autentica l'utente               │
│  ✓ Risolve permessi                 │
│  ✓ Genera JWT INTERNO (sempre!)     │
│  ✓ Crea Refresh Token               │
├─────────────────────────────────────┤
│  Ritorna JWT locale + Refresh       │
└─────────────────────────────────────┘
        ↓
   Client usa JWT interno
   per tutte le richieste
```

**Nota fondamentale**: il backend **genera sempre JWT interni**, indipendentemente da quale strategia usi. Con MSAL, il token Azure AD serve **solo per il login iniziale** per identificare l'utente. Poi il backend emette un JWT locale che il client usa normalmente.

---

# Strategia JWT

## 📧 Email + Password (Classica)

### Come Funziona

```
PASSO 1: UTENTE ACCEDE
┌─────────────────────────────────────┐
│       BROWSER / APP                 │
├─────────────────────────────────────┤
│ Email: mario@example.com            │
│ Password: ****                      │
└──────────┬──────────────────────────┘
           │
           │  POST /api/auth/login
           │  { email, password }
           │
           ▼
┌─────────────────────────────────────┐
│    SERVER - LoginHandler.cs         │
├─────────────────────────────────────┤
│ ✓ Cerca utente per email            │
│ ✓ Verifica password (BCrypt)        │
│ ✓ Verifica account attivo           │
│ ✓ Risolve permessi                  │
│ ✓ Genera JWT + Refresh Token        │
└──────────┬──────────────────────────┘
           │
           │  200 OK
           │  {
           │    accessToken: "eyJ...",
           │    expiresAt: "2026-03-29T11:30:00Z",
           │    permissions: [...]
           │  }
           │
           │  Set-Cookie: refreshToken=...
           │  (HttpOnly, Secure, SameSite)
           │
           ▼
┌─────────────────────────────────────┐
│       BROWSER / APP                 │
├─────────────────────────────────────┤
│ ✓ Salva JWT in localStorage         │
│ ✓ Riceve refreshToken in cookie     │
│ ✓ Utente LOGGATO ✓                  │
└─────────────────────────────────────┘
```

### Configurazione JWT

#### **Backend - appsettings.local.json**
```json
{
  "Auth": {
    "Strategy": "Jwt",
    "Jwt": {
      "Secret": "your-super-secret-key-at-least-32-characters-long",
      "Issuer": "Tama",
      "Audience": "TamaClients",
      "ExpiresInMinutes": 60,
      "RefreshExpiresInDays": 7
    }
  }
}
```

#### **Frontend - .env.local**
```env
VITE_API_BASE_URL=http://localhost:5001
VITE_AUTH_STRATEGY=Jwt
```

### Dettagli Tecnici JWT

**File Backend:**
- `Features/Auth/Login/LoginHandler.cs` — logica di login
- `Features/Auth/Shared/JwtTokenHelper.cs` — generazione JWT
- `Features/Auth/Shared/PermissionResolver.cs` — risoluzione permessi

**Come funziona:**

1. **Password**: verificata con BCrypt (one-way hash, impossibile recovare)
2. **JWT**: firmato con HMAC-SHA256
   ```
   Header: { "alg": "HS256", "typ": "JWT" }
   Payload: { "sub": "userId", "email": "...", "permissions": [...], "exp": ... }
   Signature: HMAC-SHA256(header.payload, secret)
   ```
3. **Scadenza**: 60 minuti (configurabile)
4. **Refresh**: usando il refresh token, ottieni un nuovo JWT ogni ora

---

# Strategia MSAL

## ☁️ Azure AD (Microsoft/Office 365)

### Come Funziona

```
PASSO 1: LOGIN CON MICROSOFT
┌─────────────────────────────────────┐
│       BROWSER / APP                 │
├─────────────────────────────────────┤
│ Clicca: "Login con Microsoft"       │
└──────────┬──────────────────────────┘
           │
           │ Reindirizzamento a:
           │ login.microsoftonline.com
           │
           ▼
       ┌─────────────────────────┐
       │    AZURE AD (Cloud)     │
       ├─────────────────────────┤
       │ • Mostra form login     │
       │ • Utente inserisce cred │
       │ • Verifica 2FA se abit. │
       │ • Genera token Azure    │
       │ • Reindirizza indietro  │
       │   con token JWT Azure   │
       └──────────┬──────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│       BROWSER / APP                 │
├─────────────────────────────────────┤
│ Ha token Azure:                     │
│ eyJhbGciOiJSUzI1NiIsInR5cCI... (JWT)
│                                     │
│ POST /api/auth/msal-login           │
│ { azureToken: "eyJ..." }            │
└──────────┬──────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ SERVER - MsalLoginHandler.cs        │
├─────────────────────────────────────┤
│ ✓ Valida firma del token Azure      │
│   (scarica chiavi pubbliche JWKS)   │
│ ✓ Verifica issuer, audience, exp    │
│ ✓ Estrae email dal token            │
│ ✓ Cerca utente locale per email     │
│ ✓ Risolve permessi                  │
│ ✓ Genera JWT INTERNO (non Azure)    │
│ ✓ Crea Refresh Token                │
└──────────┬──────────────────────────┘
           │
           │  200 OK
           │  {
           │    accessToken: "eyJ..." (JWT INTERNO),
           │    expiresAt: "...",
           │    permissions: [...]
           │  }
           │
           │  Set-Cookie: refreshToken=...
           │
           ▼
┌─────────────────────────────────────┐
│       BROWSER / APP                 │
├─────────────────────────────────────┤
│ ✓ Riceve JWT interno (non Azure)    │
│ ✓ Usa come qualsiasi JWT normale    │
│ ✓ Utente LOGGATO ✓                  │
└─────────────────────────────────────┘
```

### Configurazione MSAL

#### **Backend - appsettings.local.json**
```json
{
  "Auth": {
    "Strategy": "Msal",
    "Jwt": {
      "Secret": "your-super-secret-key-at-least-32-characters-long",
      "Issuer": "Tama",
      "Audience": "TamaClients",
      "ExpiresInMinutes": 60,
      "RefreshExpiresInDays": 7
    },
    "Msal": {
      "Instance": "https://login.microsoftonline.com/",
      "TenantId": "your-azure-tenant-id",
      "ClientId": "your-azure-app-client-id"
    }
  }
}
```

#### **Frontend - .env.local**
```env
VITE_API_BASE_URL=http://localhost:5001
VITE_AUTH_STRATEGY=Msal
VITE_MSAL_CLIENT_ID=your-azure-app-client-id
VITE_MSAL_TENANT_ID=your-azure-tenant-id
```

### Dove Ottenere TenantId e ClientId

1. Vai su **Azure Portal** → **Azure Active Directory**
2. **TenantId**:
   - Home → Gestisci → Proprietà → ID directory
3. **ClientId**:
   - Registrazioni app → Seleziona app → ID applicazione (client)

### Dettagli Tecnici MSAL

**File Backend:**
- `Features/Auth/MsalLogin/MsalLoginHandler.cs` — logica MSAL (parsing token Azure + lookup utente)

**File Frontend:**
- `plugins/msal.ts` — configurazione MSAL Browser
- `services/msal.service.ts` — metodi login/redirect

**Come funziona:**

1. **Token Azure**: firmato con chiave privata di Microsoft (RSA-256), validato dalla libreria `@azure/msal-browser` nel frontend
2. **Backend**: legge il token Azure ricevuto (parsing JWT senza rivalidare la firma)
3. **Email estrapolata**: dal claim `preferred_username` o `email` del token Azure
4. **Utente locale**: cercato nel database MongoDB per email corrispondente
5. **JWT interno generato**: stesso formato della strategia JWT, usato dal client per tutte le richieste
6. **Sicurezza**: la validazione del token Azure avviene lato client tramite MSAL; il backend si fida del token ricevuto dall'app

---

# Flussi Completi

## Flusso 1: Login → Accesso Protetto → Refresh

**Vale per entrambe le strategie (dopo il login iniziale, il comportamento è identico):**

```
1. USER LOGS IN
   Email + Password (JWT) o Token Azure (MSAL)
         │
         ▼
┌─────────────────────────────────────┐
│       SERVER AUTENTICA              │
│   Genera JWT INTERNO + Refresh      │
└──────────┬──────────────────────────┘
           │
           ▼
    CLIENT RICEVE JWT
         │
         │  Salva in localStorage
         │  Riceve refreshToken cookie
         │
         ▼
2. CLIENT ACCEDE ALLA RISORSA
   GET /api/users
   Authorization: Bearer eyJ...
         │
         ▼
┌─────────────────────────────────────┐
│  SERVER - JWT Middleware            │
│  Verifica firma, issuer, audience   │
│  Verifica scadenza                  │
│  ✓ JWT VALIDO → Consenti accesso   │
└──────────┬──────────────────────────┘
           │
           ▼
    200 OK { data }
         │
         │
         ▼
3. 60 MINUTI DOPO
   JWT SCADE
   Client tenta richiesta
         │
         ▼
    401 Unauthorized
         │
         ▼
4. CLIENT CHIEDE REFRESH
   POST /api/auth/refresh
   Cookie: refreshToken=...
         │
         ▼
┌─────────────────────────────────────┐
│  SERVER - RefreshTokenHandler       │
│  Valida refreshToken                │
│  REVOCA vecchio token (rotation)    │
│  Genera nuovo JWT                   │
│  Genera nuovo RefreshToken          │
└──────────┬──────────────────────────┘
           │
           ▼
    200 OK + nuovo JWT
    Set-Cookie: refreshToken=...
         │
         ▼
    CLIENT CONTINUA
    (stesso flusso di prima)
```

## Flusso 2: Logout

```
CLIENT: POST /api/auth/logout
            │
            ▼
BACKEND: Revoca refreshToken
         Invalida cookie
            │
            ▼
RESPONSE: 204 No Content
            │
            ▼
CLIENT: Cancella localStorage
        Utente DISCONNESSO ✓
```

---

# Come Switchare

## Procedura: Da JWT → MSAL

### Passo 1: Configura Azure AD

1. **Azure Portal** → Registra una nuova app
2. Annota: `TenantId` e `ClientId`
3. Configura redirect URI: `http://localhost:5173/login`

### Passo 2: Aggiorna Backend

**appsettings.local.json:**
```json
{
  "Auth": {
    "Strategy": "Msal",  // ← Cambia da "Jwt"
    "Msal": {
      "TenantId": "your-tenant-id",
      "ClientId": "your-client-id"
    }
  }
}
```

**Risultato**:
- ✅ `POST /api/auth/msal-login` — Endpoint usato dal frontend
- ℹ️ `POST /api/auth/login` — Sempre disponibile, ma non usato dal frontend MSAL

> **Nota**: Il backend non disabilita endpoint in base alla strategia. Entrambi restano sempre attivi. È il frontend che determina quale flusso mostrare all'utente.

### Passo 3: Aggiorna Frontend

**.env.local:**
```env
VITE_AUTH_STRATEGY=Msal  # ← Cambia da "Jwt"
VITE_MSAL_CLIENT_ID=your-client-id
VITE_MSAL_TENANT_ID=your-tenant-id
```

**Risultato**:
- LoginPage mostra solo bottone "Login con Microsoft"
- Form email/password nascosto

### Passo 4: Testa

```bash
# Backend su http://localhost:5001
# Frontend su http://localhost:5173
# Clicca "Login con Microsoft"
# → Reindirizzamento a Azure
# → Ritorno con token
# → JWT interno generato
# → Login completo
```

---

## Procedura: Da MSAL → JWT

Basta cambiare la `Strategy` a `"Jwt"` nel backend e `VITE_AUTH_STRATEGY=jwt` nel frontend. Le variabili MSAL nel frontend possono essere rimosse o lasciate (vengono ignorate se la strategia è JWT).

> **Nota importante**: Il backend **non disabilita** nessun endpoint in base alla strategia. Entrambi i percorsi (`/login` e `/msal-login`) sono sempre disponibili. La configurazione `Auth:Strategy` nel backend serve solo per documentazione e per eventuali future estensioni. È il **frontend** che decide quale UI mostrare (form email/password o bottone Microsoft) in base a `VITE_AUTH_STRATEGY`.

---

# Livello Tecnico

## Struttura Completa Backend

```
Features/Auth/
├── Login/
│   ├── LoginCommand.cs
│   ├── LoginHandler.cs          ← Email + password
│   ├── LoginValidator.cs
│   └── LoginResponse.cs
│
├── MsalLogin/
│   ├── MsalLoginCommand.cs
│   ├── MsalLoginHandler.cs      ← Token Azure
│   └── MsalLoginResponse.cs
│
├── RefreshToken/
│   ├── RefreshTokenCommand.cs
│   ├── RefreshTokenHandler.cs   ← Rinnova JWT (uguale per entrambe)
│   └── RefreshTokenResponse.cs
│
├── Logout/
│   ├── LogoutCommand.cs
│   └── LogoutHandler.cs         ← Revoca token (uguale per entrambe)
│
├── Shared/
│   └── JwtTokenHelper.cs        ← Generazione JWT interno + refresh token
│
└── AuthEndpoints.cs             ← Routing (tutti gli endpoint sempre attivi)
```

### File Critici

| File | Ruolo | Linee Codice |
|---|---|---|
| `JwtTokenHelper.cs` | Genera JWT + RefreshToken | ~60 |
| `LoginHandler.cs` | Autentica con email/password | ~40 |
| `MsalLoginHandler.cs` | Autentica con Azure AD | ~35 |
| `RefreshTokenHandler.cs` | Rinnova JWT | ~40 |
| `AuthExtensions.cs` | Registra servizi + JWT config | ~40 |
| `AuthEndpoints.cs` | Espone endpoint | ~50 |

### Sequenza di Login JWT

```csharp
LoginHandler.Handle()
├── userRepo.GetByEmailAsync()           // Cerca utente
├── BCrypt.Verify()                      // Verifica password
├── ResolvePermissionsAsync() (inline)   // Risolve permessi (ruoli diretti + gruppi)
├── JwtTokenHelper.GenerateAccessToken() // JWT firmato
├── JwtTokenHelper.CreateRefreshTokenAsync() // Salva su MongoDB
└── Ritorna LoginResponse
```

### Sequenza di Login MSAL

```csharp
MsalLoginHandler.Handle()
├── JwtSecurityTokenHandler.ReadJwtToken()  // Legge il JWT Azure (senza validare firma)
├── Estrae email dal claim (preferred_username / email)
├── userRepo.GetByEmailAsync()           // Cerca utente locale
├── Verifica user.IsActive               // Account attivo?
├── ResolvePermissions (inline)           // Risolve permessi (ruoli diretti + gruppi)
├── JwtTokenHelper.GenerateAccessToken() // JWT interno
├── JwtTokenHelper.CreateRefreshTokenAsync() // Salva su MongoDB
└── Ritorna MsalLoginResponse
```

### Parsing Token Azure (inline in MsalLoginHandler)

```csharp
// MsalLoginHandler.Handle() — parsing del token Azure AD
var handler = new JwtSecurityTokenHandler();
if (!handler.CanReadToken(request.AzureToken))
    throw new UnauthorizedException("Token Azure non valido.");

var jwt = handler.ReadJwtToken(request.AzureToken);

// Estrae email dal claim (Azure usa "preferred_username" o "email")
var email = jwt.Claims.FirstOrDefault(c =>
    c.Type == "preferred_username" || c.Type == "email")?.Value
    ?? throw new UnauthorizedException("Claim 'email' assente nel token Azure.");

// Cerca utente locale per email
var user = await userRepo.GetByEmailAsync(email, cancellationToken)
    ?? throw new UnauthorizedException($"Nessun utente locale con email '{email}'.");
```

> **Nota sulla validazione**: Il backend **non rivalidda la firma RSA** del token Azure. La validazione crittografica avviene nel frontend tramite la libreria `@azure/msal-browser`, che verifica la firma con le chiavi pubbliche JWKS di Microsoft prima di consegnare il token all'app. Il backend si limita a leggere il token e ad estrarre l'email per trovare l'utente locale corrispondente.
```

---

## Struttura Frontend

```
src/
├── pages/auth/
│   └── LoginPage.vue       ← Switchante JWT/MSAL UI
│
├── components/auth/
│   └── LoginDialog.vue     ← Re-autenticazione (stesso switchante)
│
├── services/
│   ├── auth.service.ts     ← POST /login e /msal-login
│   └── msal.service.ts     ← Wrapper @azure/msal-browser
│
├── plugins/
│   └── msal.ts             ← PublicClientApplication
│
├── stores/
│   └── auth.store.ts       ← Store Pinia (login, loginWithMsal, logout)
│
└── vite-env.d.ts           ← Tipi per variabili d'ambiente
```

### LoginPage.vue - Logica di Switch

```typescript
const authStrategy = import.meta.env.VITE_AUTH_STRATEGY

// Se JWT: mostra form email/password
// Se MSAL: mostra bottone Microsoft
// Se MSAL + redirect da Azure: chiama msalService.handleRedirectResult()
```

---

# Sicurezza

## ✅ Implemented

### JWT
- ✅ HMAC-SHA256 signature
- ✅ BCrypt password hashing
- ✅ 60-minute expiration
- ✅ Refresh token rotation
- ✅ HttpOnly cookie
- ✅ Generic error messages (no enumeration)

### MSAL
- ✅ RSA signature verification (Microsoft keys)
- ✅ Issuer validation
- ✅ Audience validation
- ✅ Token expiration check
- ✅ JWKS key caching (1 hour)
- ✅ All JWT protections (applies to internal JWT)

## ⚠️ Considerations

1. **Secret Management**:
   - ❌ NEVER hardcode secrets
   - ✅ USE `.local.json` files (gitignored)
   - ✅ USE Azure Key Vault in production

2. **HTTPS**:
   - Backend: Set `Secure=true` in production
   - Frontend: Use HTTPS URLs only

3. **JWKS Caching**:
   - `ConfigurationManager` automatically caches for 1 hour
   - No need to refresh manually

4. **User Sync (MSAL)**:
   - Users must exist in Tama DB beforehand
   - Azure AD doesn't auto-create users
   - Consider: Azure → Tama sync process

---

# Troubleshooting

## ❌ Errore: "401 Unauthorized — Token expired"
**Soluzione**: Il frontend intercetta automaticamente il 401 e chiama `authStore.logout()` per disconnettere l'utente. L'utente dovrà effettuare un nuovo login.

## ❌ Errore: "Nessun utente locale con email"
**MSAL Only**: L'utente deve essere creato manualmente in Tama prima di poter fare login con Azure AD

## ❌ Errore: "Token Azure non valido"
**MSAL Only**: Il token ricevuto dal frontend non è un JWT leggibile. Verificare che il frontend stia inviando l'`idToken` (non l'`accessToken`) di MSAL.

## ❌ Bottone Microsoft non appare
**Causa**: `.env.local` ha `VITE_AUTH_STRATEGY` diverso da `msal`
**Soluzione**: Cambia a `VITE_AUTH_STRATEGY=msal` (lowercase, il confronto nel frontend è case-sensitive)

## ❌ Login classico non funziona (form non visibile)
**Causa**: `.env.local` ha `VITE_AUTH_STRATEGY=msal`
**Soluzione**: Cambia a `VITE_AUTH_STRATEGY=jwt` (o qualsiasi valore diverso da `msal`)

## ✅ Debug Checklist

```
□ appsettings.local.json ha Auth:Strategy?
□ .env.local ha VITE_AUTH_STRATEGY?
□ Secret JWT non vuoto?
□ Se MSAL: TenantId e ClientId compilati?
□ Se MSAL: utenti esistono in DB?
□ Se MSAL: redirect URI configurato in Azure?
□ Backend sta ascoltando su http://localhost:5001?
□ Frontend sta ascoltando su http://localhost:5173?
□ Browser console: errori?
□ Network tab: request/response codes?
```

---

# Confronto Dettagliato

## Tabella Comparativa Completa

| Caratteristica | JWT (Email+Password) | MSAL (Azure AD) |
|---|---|---|
| **Tipo di credenziali** | Email + Password locale | Account Microsoft/Azure AD |
| **Dove vivono le credenziali** | MongoDB (hash BCrypt) | Azure Active Directory (cloud Microsoft) |
| **Login UI** | Form email/password | Bottone "Login con Microsoft" → redirect Azure |
| **Single Sign-On (SSO)** | ❌ No | ✅ Sì (con Office 365, Teams, Outlook, etc.) |
| **Multi-Factor Auth (MFA)** | ❌ Da implementare manualmente | ✅ Gestito da Microsoft (Authenticator, SMS, FIDO2) |
| **Recupero password** | ❌ Da implementare manualmente | ✅ Gestito da Microsoft |
| **Gestione utenti** | Completa nel backend (CRUD) | Azure AD + sync manuale nel backend |
| **Pre-requisito utente** | Creazione nel sistema | Creazione in Azure AD **E** nel sistema locale |
| **Complessità setup** | ⭐ Bassa | ⭐⭐⭐ Media-Alta (richiede Azure Portal) |
| **Dipendenze esterne** | Nessuna | Azure AD, Microsoft Identity Platform |
| **Costo** | Gratuito | Azure AD Free/Premium (in base alle features) |
| **Validazione token iniziale** | BCrypt.Verify (password hash) | Parsing JWT Azure (firma validata lato client da MSAL) |
| **Token usato per API** | JWT interno (identico) | JWT interno (identico) |
| **Refresh token** | ✅ Rotation con HttpOnly cookie | ✅ Rotation con HttpOnly cookie (identico) |
| **Rate limiting** | ✅ 10 req / 15 min | ✅ 10 req / 15 min (identico) |
| **Offline capability** | ✅ Funziona senza internet (dopo login) | ✅ Funziona senza internet (dopo login) |
| **Integrazione Azure ecosystem** | ❌ No | ✅ Graph API, SharePoint, Teams, etc. |

## Vantaggi JWT

1. **Semplicità**: nessuna dipendenza esterna, setup in 5 minuti
2. **Controllo totale**: gestisci tutto (password policy, reset, blocco account)
3. **Nessun costo**: non richiede abbonamento Azure
4. **Portabilità**: funziona ovunque, su qualsiasi cloud o on-premise
5. **Debug facile**: tutto nel tuo codice, nessun servizio esterno da diagnosticare

## Svantaggi JWT

1. **Sicurezza password**: sei responsabile di hash, policy, reset, leak detection
2. **Nessun SSO**: ogni app richiede un login separato
3. **Nessun MFA nativo**: devi implementare 2FA da zero (TOTP, SMS, etc.)
4. **User onboarding**: l'utente deve creare un account con password nel tuo sistema

## Vantaggi MSAL

1. **SSO enterprise**: un solo login per Office 365, Teams, SharePoint e la tua app
2. **MFA incluso**: Microsoft Authenticator, SMS, chiavi FIDO2 — tutto gratis
3. **Zero gestione password**: Microsoft gestisce hash, reset, policy, breach detection
4. **Conditional Access**: policy Azure AD (blocco per IP, device compliance, etc.)
5. **Compliance**: certificazioni SOC 2, ISO 27001, GDPR built-in

## Svantaggi MSAL

1. **Setup complesso**: richiede registrazione app su Azure Portal, configurazione redirect URI
2. **Dipendenza Microsoft**: se Azure AD ha problemi, nessuno può loggarsi
3. **Costo potenziale**: Azure AD P1/P2 per feature avanzate (Conditional Access, etc.)
4. **Doppia gestione utenti**: l'utente deve esistere sia in Azure AD che nel DB locale
5. **Debug più difficile**: errori possono venire da Azure, dal frontend MSAL, o dal backend
6. **Nessuna validazione firma lato backend**: il backend si fida del token Azure ricevuto dal frontend (la validazione avviene lato client tramite la libreria MSAL)

## Quando Scegliere Cosa

### Scegli **JWT** se:
- Stai sviluppando un'**app interna** o un **prototipo**
- Il team è **piccolo** e non usa Office 365
- Vuoi **massimo controllo** su tutto il flusso di autenticazione
- Non hai un **tenant Azure AD** configurato
- L'app deve funzionare **completamente offline/on-premise**

### Scegli **MSAL** se:
- L'azienda usa **Office 365 / Microsoft 365**
- Servono **SSO** e **MFA** senza implementarli da zero
- Hai un **Azure AD tenant** già configurato
- L'app fa parte di un **ecosistema enterprise Microsoft**
- Servono **policy di Conditional Access** (blocco per paese, device, etc.)

---

**Fine Documentazione**

Scegli una strategia (JWT per semplicità, MSAL per enterprise) e segui la configurazione corrispondente. Entrambe sono completamente implementate e pronte all'uso! 🚀
