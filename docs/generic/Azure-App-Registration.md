# ☁️ Configurazione Azure — App Registration per MSAL

> **Documento**: Guida step-by-step per configurare Azure AD (Entra ID)
> **Lingua**: Italiano
> **Ultimo aggiornamento**: 2026-03-29
> **Versione**: 1.0
> **Prerequisiti**: Un account Azure con permessi per gestire App Registration

---

## 📋 Indice

1. [Panoramica](#panoramica) — cosa serve e perché
2. [Step 1: Accedere al portale Azure](#step-1-accedere-al-portale-azure)
3. [Step 2: Creare una App Registration](#step-2-creare-una-app-registration)
4. [Step 3: Configurare la piattaforma SPA](#step-3-configurare-la-piattaforma-spa)
5. [Step 4: Copiare TenantId e ClientId](#step-4-copiare-tenantid-e-clientid)
6. [Step 5: Configurare i permessi API](#step-5-configurare-i-permessi-api)
7. [Step 6: Configurare il progetto Tama](#step-6-configurare-il-progetto-tama)
8. [Step 7: Verificare il funzionamento](#step-7-verificare-il-funzionamento)
9. [Configurazione per Produzione](#configurazione-per-produzione)
10. [Troubleshooting](#troubleshooting)

---

# Panoramica

## Cosa serve?

Per far funzionare l'autenticazione MSAL nel progetto Tama servono due valori che si ottengono da Azure:

| Valore | Cos'è | Dove si usa |
|---|---|---|
| **TenantId** | Identificativo dell'organizzazione Azure AD (la tua azienda) | Backend + Frontend |
| **ClientId** | Identificativo dell'app registrata in Azure | Backend + Frontend |

## Come funziona il flusso

```
┌─────────────────────────────────┐
│   Azure AD (Entra ID)           │
│                                 │
│  App Registration               │
│  ├─ ClientId: edf643a0-...     │
│  ├─ TenantId: b04fdd3d-...     │
│  ├─ Redirect URI: /login        │
│  └─ Piattaforma: SPA            │
│                                 │
│  Utenti autorizzati:            │
│  ├─ mario@tuaazienda.com        │
│  └─ giulia@tuaazienda.com       │
└────────────────┬────────────────┘
                 │
                 │  Token JWT firmato da Microsoft
                 │  (contiene email dell'utente)
                 │
                 ▼
┌─────────────────────────────────┐
│   Tama Frontend           │
│   @azure/msal-browser           │
│   → Invia token al backend      │
└────────────────┬────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│   Tama Backend            │
│   Legge email dal token Azure   │
│   Cerca utente locale (MongoDB) │
│   Genera JWT interno            │
└─────────────────────────────────┘
```

> **Importante**: L'utente **deve esistere sia in Azure AD che nel database locale** di Tama (con la stessa email). Azure AD autentica l'identità, ma i permessi vengono dal database locale.

---

# Step 1: Accedere al portale Azure

1. Vai su **https://portal.azure.com**
2. Accedi con un account che ha il ruolo **Amministratore globale** o **Amministratore applicazioni**
3. Nella barra di ricerca in alto, cerca **"Microsoft Entra ID"** (ex Azure Active Directory)

```
┌─────────────────────────────────────────────────────┐
│  🔍 Cerca: "Microsoft Entra ID"                     │
│                                                     │
│  Risultati:                                         │
│  ► Microsoft Entra ID ← Clicca qui                  │
└─────────────────────────────────────────────────────┘
```

> **Nota**: Microsoft ha rinominato "Azure Active Directory" in "Microsoft Entra ID". Sono la stessa cosa.

---

# Step 2: Creare una App Registration

1. Nel pannello di Microsoft Entra ID, clicca **"Registrazioni app"** nel menu laterale (sotto "Gestisci")
2. Clicca **"+ Nuova registrazione"** in alto

```
┌─────────────────────────────────────────────────────┐
│  Registra un'applicazione                           │
│                                                     │
│  Nome: Tama                                   │
│        (o un nome descrittivo a tua scelta)         │
│                                                     │
│  Tipi di account supportati:                        │
│  ○ Account solo in questa directory organizzativa   │
│    (Single tenant) ← SELEZIONA QUESTO               │
│  ○ Account in qualsiasi directory organizzativa     │
│  ○ Account in qualsiasi directory + account         │
│    Microsoft personali                              │
│                                                     │
│  URI di reindirizzamento (facoltativo):             │
│  → Configureremo dopo nello Step 3                  │
│                                                     │
│  [ Registra ]                                       │
└─────────────────────────────────────────────────────┘
```

### Quale tipo di account scegliere?

| Opzione | Quando usarla |
|---|---|
| **Solo questa directory (Single tenant)** | ✅ **Consigliato** — se tutti gli utenti appartengono alla stessa organizzazione Azure AD |
| **Qualsiasi directory (Multi-tenant)** | Se utenti di organizzazioni Azure diverse devono accedere |
| **Qualsiasi directory + personali** | Se anche account Microsoft personali (outlook.com, hotmail) devono accedere |

> Per la maggior parte dei casi aziendali, **Single tenant** è la scelta corretta.

3. Clicca **"Registra"**
4. Verrai reindirizzato alla pagina di panoramica della nuova app

---

# Step 3: Configurare la piattaforma SPA

Questo è il passaggio **più importante** — senza questa configurazione, MSAL non funziona.

1. Nella pagina della tua app, vai a **"Autenticazione"** nel menu laterale
2. Clicca **"+ Aggiungi una piattaforma"**
3. Seleziona **"Applicazione a pagina singola (SPA)"**

```
┌─────────────────────────────────────────────────────┐
│  Configura applicazione a pagina singola (SPA)      │
│                                                     │
│  URI di reindirizzamento:                           │
│  ┌─────────────────────────────────────────────┐    │
│  │ http://localhost:5173/login                  │    │ ← Sviluppo
│  └─────────────────────────────────────────────┘    │
│                                                     │
│  [ Configura ]                                      │
└─────────────────────────────────────────────────────┘
```

4. Clicca **"Configura"**

### Aggiungere URI aggiuntivi

Dopo aver configurato la piattaforma, puoi aggiungere altri URI di reindirizzamento:

- Torna su **"Autenticazione"**
- Nella sezione **"Applicazione a pagina singola (SPA)"** → **"URI di reindirizzamento"**
- Clicca **"Aggiungi URI"** per aggiungere altri ambienti:

```
URI di reindirizzamento:
┌─────────────────────────────────────────────────────┐
│ ✓ http://localhost:5173/login        ← Sviluppo     │
│ ✓ http://localhost:5174/login        ← Porta alt.   │
│ ✓ https://tuodominio.com/login       ← Produzione   │
└─────────────────────────────────────────────────────┘
```

> **⚠️ Attenzione**: L'URI di reindirizzamento **deve corrispondere esattamente** a `window.location.origin + '/login'` configurato nel file `plugins/msal.ts` del frontend. Ogni mismatch causa un errore `redirect_uri_mismatch`.

### Impostazioni avanzate (nella stessa pagina Autenticazione)

Scorri verso il basso nella pagina "Autenticazione":

```
┌─────────────────────────────────────────────────────┐
│  Concessione implicita e flussi ibridi               │
│                                                     │
│  ☐ Token di accesso                                 │
│  ☐ Token ID                                         │
│                                                     │
│  → Lascia ENTRAMBI DESELEZIONATI                    │
│  (Il progetto usa il flusso Authorization Code       │
│   con PKCE, non il flusso implicito)                │
└─────────────────────────────────────────────────────┘
```

> Il flusso **Authorization Code con PKCE** è più sicuro del flusso implicito e viene usato automaticamente dalla libreria `@azure/msal-browser`.

---

# Step 4: Copiare TenantId e ClientId

### ClientId (ID applicazione)

1. Vai alla pagina **"Panoramica"** della tua app registration
2. Copia il valore **"ID applicazione (client)"**

```
┌─────────────────────────────────────────────────────┐
│  Panoramica                                         │
│                                                     │
│  Nome visualizzato:  Tama                     │
│  ID applicazione (client):                          │
│    edf643a0................9ed9  📋 Copia  │
│                                                     │
│  ID della directory (tenant):                       │
│    b04fdd3d................1b5f07  📋 Copia  │
└─────────────────────────────────────────────────────┘
```

### TenantId (ID directory)

Si trova nella stessa pagina **"Panoramica"**, campo **"ID della directory (tenant)"**.

> **Conserva entrambi i valori**: ti serviranno nello Step 6 per configurare backend e frontend.

---

# Step 5: Configurare i permessi API

Configura i permessi API su Azure esponendo la tua app come API e definendo uno scope custom `access_as_user`.

> **Nota**: il backend di Tama **valida la firma (JWKS), l'issuer e l'audience** del token Azure. Configurare "Expose an API" con lo scope `access_as_user` fa sì che il frontend ottenga un **Access Token** (anziché un semplice ID Token). Il backend accetta come audience sia `api://{clientId}` sia il `clientId` nudo, quindi funziona anche se lo scope non è ancora stato esposto — ma è comunque consigliato configurarlo per ottenere un Access Token corretto.

## Expose an API + Scope custom `access_as_user`

### Passo 1: Expose an API

1. Vai alla sezione **"Esponi un'API"** nel menu laterale
2. Clicca **"Aggiungi"** accanto a "URI ID applicazione". Azure proporrà automaticamente:
   ```
   api://edf643a0................9ed9
   ```
   (dove `edf643a0-...` è il tuo ClientId)
3. Conferma cliccando **"Salva"**

### Passo 2: Aggiungere lo scope

1. Clicca **"+ Aggiungi un ambito"**
2. Compila i campi:

```
┌─────────────────────────────────────────────────────┐
│  Aggiungi un ambito                                 │
│                                                     │
│  Nome ambito:      access_as_user                   │
│                                                     │
│  Autorizzazione:   Amministratori e utenti          │
│                    (Admins and users)                │
│                                                     │
│  Nome visualizzato consenso amministratore:         │
│    Accedi a Tama come utente                  │
│                                                     │
│  Descrizione consenso amministratore:               │
│    Consente all'app di accedere a Tama        │
│    per conto dell'utente connesso.                  │
│                                                     │
│  Nome visualizzato consenso utente:                 │
│    Accedi a Tama come utente                  │
│                                                     │
│  Descrizione consenso utente:                       │
│    Consente all'app di accedere a Tama        │
│    per conto tuo.                                   │
│                                                     │
│  Stato: Abilitato                                   │
│                                                     │
│  [ Aggiungi ambito ]                                │
└─────────────────────────────────────────────────────┘
```

3. Lo scope risultante sarà: `api://edf643a0-.../access_as_user`

### Passo 3: Aggiungere il permesso API

1. Vai alla sezione **"Autorizzazioni API"** nel menu laterale
2. Clicca **"+ Aggiungi un'autorizzazione"**
3. Seleziona il tab **"Le mie API"** (My APIs)
4. Seleziona la tua App Registration (es. "Tama")
5. Vedrai solo **"Autorizzazioni delegate"** — selezionalo
6. Spunta il check **`access_as_user`**
7. Clicca **"Aggiungi autorizzazioni"**

```
┌─────────────────────────────────────────────────────┐
│  Autorizzazioni API                                 │
│                                                     │
│  API / Nome permesso        Tipo          Stato     │
│  ─────────────────────────────────────────────────  │
│  Tama (api://edf643a0-...)                    │
│    └─ access_as_user        Delegato    ✅ Concesso  │
└─────────────────────────────────────────────────────┘
```

**Frontend — `services/msal.service.ts`** (configurazione corrispondente):
```typescript
const loginRequest = {
  scopes: ['api://edf643a0................9ed9/access_as_user'],
}
```

> Sostituisci il ClientId nell'URI con il tuo.

> **Nota**: MSAL aggiunge automaticamente `openid` e `profile` alla richiesta anche se non li specifichi esplicitamente negli scope. Il frontend invia l'**Access Token** (non l'ID Token) al backend.

---

## Concedere il consenso amministratore

Indipendentemente dall'opzione scelta, se vedi lo stato "Non concesso" accanto ai permessi:

1. Clicca **"Concedi consenso amministratore per [tua organizzazione]"**
2. Conferma cliccando **"Sì"**

> Senza il consenso amministratore, ogni utente dovrà accettare individualmente i permessi al primo login. Con il consenso amministratore, il processo è trasparente.

---

# Step 6: Configurare il progetto Tama

Ora che hai `TenantId` e `ClientId`, configura il progetto.

### Backend — `appsettings.local.json`

```json
{
  "Auth": {
    "Strategy": "Msal",
    "Jwt": {
      "Secret": "CAMBIA_QUESTO_SEGRETO_IN_PRODUZIONE_MIN_32_CHARS"
    },
    "Msal": {
      "TenantId": "b04fdd3d................1b5f07",
      "ClientId": "edf643a0................9ed9"
    }
  }
}
```

> Sostituisci i valori di `TenantId` e `ClientId` con quelli copiati dallo Step 4.

### Frontend — `.env.local`

Crea (o modifica) il file `.env.local` nella cartella `apps/frontend/`:

```env
VITE_API_BASE_URL=http://localhost:5001
VITE_AUTH_STRATEGY=Msal
VITE_MSAL_CLIENT_ID=edf643a0................9ed9
VITE_MSAL_TENANT_ID=b04fdd3d................1b5f07
```

> Sostituisci i valori con quelli copiati dallo Step 4.

### Corrispondenza valori

Assicurati che i valori coincidano in tutti i file:

```
Azure Portal                    Backend                     Frontend
─────────────────────────────────────────────────────────────────────
ID applicazione (client)   →   Auth:Msal:ClientId      →   VITE_MSAL_CLIENT_ID
ID directory (tenant)      →   Auth:Msal:TenantId      →   VITE_MSAL_TENANT_ID
```

---

# Step 7: Verificare il funzionamento

### Checklist pre-test

- [ ] App Registration creata su Azure
- [ ] Piattaforma SPA configurata con redirect URI `http://localhost:5173/login`
- [ ] Expose an API configurato con scope `access_as_user` e consenso amministratore concesso
- [ ] `TenantId` e `ClientId` configurati nel backend (`appsettings.local.json`)
- [ ] `VITE_AUTH_STRATEGY=Msal` nel frontend (`.env.local`)
- [ ] `VITE_MSAL_CLIENT_ID` e `VITE_MSAL_TENANT_ID` configurati nel frontend
- [ ] L'utente Azure AD esiste anche nel database locale di Tama (stessa email)

### Test

1. Avvia il backend:
   ```bash
   pnpm serve:backend
   ```

2. Avvia il frontend:
   ```bash
   pnpm serve:frontend
   ```

3. Vai su `http://localhost:5173`
4. Dovresti vedere il bottone **"Login con Microsoft"**
5. Clicca → verrai reindirizzato a `login.microsoftonline.com`
6. Inserisci le credenziali dell'utente Azure AD
7. Dopo il login, verrai reindirizzato a `http://localhost:5173/login`
8. Il frontend invia il token Azure al backend → il backend genera il JWT interno → sei loggato

### Verifica nel browser (DevTools → Network)

```
1. POST http://localhost:5001/api/auth/msal-login
   Body: { "azureToken": "eyJ..." }
   Response: { "accessToken": "eyJ...", "expiresAt": "...", ... }

2. Da questo punto, tutte le richieste usano il JWT interno:
   GET http://localhost:5001/api/users
   Headers: Authorization: Bearer eyJ...
```

---

# Configurazione per Produzione

### Aggiungere il dominio di produzione

1. Torna su Azure Portal → App Registration → **Autenticazione**
2. Nella sezione SPA, aggiungi l'URI di produzione:

```
URI di reindirizzamento:
┌─────────────────────────────────────────────────────┐
│ ✓ http://localhost:5173/login        ← Sviluppo     │
│ ✓ https://app.tuodominio.com/login   ← Produzione   │
└─────────────────────────────────────────────────────┘
```

> In produzione è **obbligatorio** usare HTTPS.

### App Registration separata (consigliato)

Per ambienti production è consigliato creare una **App Registration separata**:

| Ambiente | App Registration | Redirect URI |
|---|---|---|
| Development | Tama-Dev | `http://localhost:5173/login` |
| Production | Tama-Prod | `https://app.tuodominio.com/login` |

Questo garantisce isolamento completo tra gli ambienti e permette policy diverse.

### Variabili d'ambiente in produzione

Non committare mai `appsettings.local.json` o `.env.local`. In produzione, usa:

- **Backend**: variabili d'ambiente di sistema o Azure App Configuration
  ```
  Auth__Msal__TenantId=your-tenant-id
  Auth__Msal__ClientId=your-client-id
  Auth__Jwt__Secret=your-production-secret-min-32-chars
  ```

- **Frontend**: variabili d'ambiente nel processo di build (CI/CD)
  ```
  VITE_MSAL_CLIENT_ID=your-client-id
  VITE_MSAL_TENANT_ID=your-tenant-id
  VITE_AUTH_STRATEGY=Msal
  ```

---

# Troubleshooting

## Errori Comuni

### `redirect_uri_mismatch`

**Causa**: L'URI di reindirizzamento nel browser non corrisponde a quello registrato su Azure.

**Soluzione**:
1. Verifica l'URI esatto nel browser (es. `http://localhost:5173/login`)
2. Vai su Azure → App Registration → Autenticazione
3. Assicurati che l'URI sia **identico** (attenzione a: protocollo, porta, path, trailing slash)

```
✅ http://localhost:5173/login
❌ http://localhost:5173/login/     ← trailing slash
❌ https://localhost:5173/login     ← https vs http
❌ http://localhost:5174/login      ← porta diversa
```

### `AADSTS700054: response_type 'id_token' is not enabled`

**Causa**: Hai selezionato la piattaforma **"Web"** invece di **"SPA"**.

**Soluzione**:
1. Azure → App Registration → Autenticazione
2. Rimuovi la piattaforma "Web" se presente
3. Aggiungi la piattaforma **"Applicazione a pagina singola (SPA)"**

> La differenza è fondamentale: "Web" usa il flusso implicito (token nell'URL), "SPA" usa Authorization Code con PKCE (più sicuro).

### `AADSTS50011: The redirect URI specified in the request does not match`

Stesso problema di `redirect_uri_mismatch`. Vedi sopra.

### `AADSTS65001: The user or administrator has not consented to use the application`

**Causa**: I permessi API non hanno il consenso dell'amministratore.

**Soluzione**:
1. Azure → App Registration → Autorizzazioni API
2. Clicca **"Concedi consenso amministratore"**

### `Nessun utente locale con email 'xxx@yyy.com'`

**Causa**: L'utente esiste su Azure AD ma **non** nel database locale di Tama.

**Soluzione**:
1. Crea l'utente nel pannello admin di Tama con la **stessa email** dell'account Azure AD
2. L'email deve corrispondere al campo `preferred_username` del token Azure (solitamente è l'email UPN dell'utente)

> **Ricorda**: Azure AD verifica l'identità, ma l'utente deve esistere nel database locale per avere ruoli e permessi nel sistema.

### `IDX10214: Audience validation failed`

**Causa**: Il claim `aud` nel token Azure non corrisponde all'audience atteso dal backend. Succede tipicamente quando lo scope `access_as_user` non è stato ancora esposto su Azure (in quel caso `aud` contiene il ClientId nudo anziché `api://{clientId}`), oppure quando si usa una versione recente (7.x+) di `Microsoft.IdentityModel` che ha una validazione audience più restrittiva.

**Soluzione**:
1. Decodifica il token su https://jwt.ms e controlla il valore esatto del claim `aud`
2. Verifica che lo Step 5 ("Expose an API" + scope `access_as_user`) sia stato completato correttamente
3. Se il problema persiste, il backend di Tama usa un `AudienceValidator` custom che accetta sia `clientId` sia `api://{clientId}` — verifica che il codice in `MsalLoginHandler.cs` sia aggiornato

### `Token Azure non valido`

**Causa**: Il frontend ha inviato un token malformato o scaduto.

**Soluzione**:
1. Verifica che `VITE_MSAL_CLIENT_ID` e `VITE_MSAL_TENANT_ID` nel frontend siano corretti
2. Pulisci il localStorage del browser e riprova
3. Controlla la console del browser per errori MSAL

### Login funziona ma l'utente non ha permessi

**Causa**: L'utente locale nel database non ha ruoli/permessi assegnati.

**Soluzione**:
1. Accedi con un account admin
2. Vai a Gestione Utenti
3. Assegna ruoli all'utente (direttamente o tramite gruppi)

---

## FAQ

### Serve un Client Secret (segreto client)?

**No**. Il progetto Tama usa la libreria `@azure/msal-browser` che è un client **pubblico** (SPA). I client pubblici non possono custodire segreti in modo sicuro, quindi usano il flusso **Authorization Code con PKCE** che non richiede un client secret.

> Non generare un Client Secret per questa App Registration — non serve e non andrebbe mai esposto nel frontend.

### Posso usare account Microsoft personali (outlook.com)?

Solo se hai selezionato **"Account in qualsiasi directory + account Microsoft personali"** come tipo di account nello Step 2. Per uso aziendale, **Single tenant** è consigliato.

### Serve configurare i "Certificati e segreti"?

**No**. Questa sezione serve per applicazioni backend (confidential clients). La nostra app è un SPA (public client) — non serve nessun certificato o segreto.

### Posso limitare quali utenti Azure AD possono accedere?

Sì, in due modi:

1. **Su Azure**: App Registration → Proprietà enterprise → Assegnazione utenti obbligatoria → Sì. Poi assegna solo gli utenti/gruppi desiderati.
2. **Su Tama**: L'utente deve esistere nel database locale. Se l'utente Azure AD non ha un corrispondente locale, il backend rifiuta il login con errore 401.

### Dove trovo l'authority URL?

L'authority è composta così:
```
https://login.microsoftonline.com/{TenantId}
```

Il frontend la costruisce automaticamente nel file `plugins/msal.ts`:
```typescript
authority: `https://login.microsoftonline.com/${import.meta.env.VITE_MSAL_TENANT_ID}`
```

---

## Riferimenti

| Risorsa | Link |
|---|---|
| Portale Azure | https://portal.azure.com |
| Documentazione Entra ID | https://learn.microsoft.com/it-it/entra/identity/ |
| MSAL.js (libreria frontend) | https://github.com/AzureAD/microsoft-authentication-library-for-js |
| Flusso Auth Code + PKCE | https://learn.microsoft.com/it-it/entra/identity-platform/v2-oauth2-auth-code-flow |
| Documentazione autenticazione Tama | `Authentication.md` |
| Documentazione strategia JWT/MSAL | `Authentication-Strategy.md` |
