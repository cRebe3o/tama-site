# Deploy su Render da GitHub

Guida per pubblicare backend (.NET) e frontend (Vue/Vite) su [Render](https://render.com) collegando il repository GitHub.

---

## Prerequisiti

- Repository GitHub pushato e aggiornato
- Account Render collegato a GitHub
- Accesso alla dashboard MongoDB Atlas (per aggiornare credenziali)

---

## 1. Backend — Web Service (Docker)

### Creazione servizio

1. Dalla dashboard Render: **New → Web Service**
2. Connetti il repository GitHub `tama`
3. Configura:

| Campo              | Valore                            |
| ------------------ | --------------------------------- |
| **Name**           | `tama-backend`              |
| **Root Directory** | `apps/backend/Tama.Api`     |
| **Environment**    | `Docker`                          |

Render rileverà automaticamente il `Dockerfile` presente nella root directory.

### Variabili d'ambiente

Nella sezione **Environment** del servizio, aggiungi:

| Variabile                       | Valore                                                        | Note                                        |
| ------------------------------- | ------------------------------------------------------------- | ------------------------------------------- |
| `ASPNETCORE_ENVIRONMENT`        | `Production`                                                  | Carica `appsettings.Production.json`        |
| `ConnectionStrings__MongoDB`    | `mongodb+srv://user:pass@cluster.mongodb.net/tama?...`  | Stringa di connessione MongoDB di produzione |
| `Auth__Jwt__Secret`             | *(un secret robusto, min 32 caratteri)*                       | Chiave firma JWT                            |
| `Seed__AdminPassword`           | *(password admin iniziale)*                                   | Usata al primo seed del DB                  |
| `Cors__AllowedOrigins__0`       | `https://tama-frontend.onrender.com`                    | URL del frontend su Render                  |

> **Nota**: In .NET le variabili d'ambiente usano `__` (doppio underscore) al posto di `:` per la navigazione gerarchica. Ad esempio `ConnectionStrings__MongoDB` corrisponde a `ConnectionStrings:MongoDB` in `appsettings.json`.

### Porta

Il `Dockerfile` espone la porta **8080** (`EXPOSE 8080`). Render la rileva automaticamente dal Dockerfile; se richiesto, impostala manualmente nella configurazione del servizio.

---

## 2. Frontend — Static Site

### Creazione servizio

1. Dalla dashboard Render: **New → Static Site**
2. Connetti lo stesso repository GitHub
3. Configura:

| Campo                  | Valore                                       |
| ---------------------- | -------------------------------------------- |
| **Name**               | `tama-frontend`                        |
| **Root Directory**     | `apps/frontend`                              |
| **Build Command**      | `pnpm install && pnpm run build`             |
| **Publish Directory**  | `dist`                                       |

### Abilitare pnpm

Il progetto utilizza pnpm. Aggiungi la variabile d'ambiente:

| Variabile      | Valore |
| -------------- | ------ |
| `ENABLE_PNPM`  | `true` |

In alternativa, se vuoi usare npm:

- **Build Command**: `npm install && npm run build`

### Variabili d'ambiente

| Variabile            | Valore                                            | Note                 |
| -------------------- | ------------------------------------------------- | -------------------- |
| `VITE_API_BASE_URL`  | `https://tama-backend.onrender.com`         | URL del backend API  |

> **Importante**: Le variabili `VITE_*` vengono iniettate a build time da Vite, non a runtime. Ogni modifica richiede un nuovo build.

### Rewrite rule per SPA

Vue Router usa la history mode: tutti i percorsi devono essere reindirizzati a `index.html`.

Nella sezione **Redirects/Rewrites** del static site, aggiungi:

| Source | Destination    | Action    |
| ------ | -------------- | --------- |
| `/*`   | `/index.html`  | `Rewrite` |

---

## 3. CORS

Il file `appsettings.Production.json` contiene già una configurazione CORS:

```json
{
  "Cors": {
    "AllowedOrigins": ["https://tama-dev-frontend.onrender.com"]
  }
}
```

Puoi gestire le origini consentite in due modi:

- **Via file**: modifica `appsettings.Production.json` con l'URL corretto del frontend
- **Via env var**: imposta `Cors__AllowedOrigins__0` nelle variabili d'ambiente di Render (sovrascrive il file)

---

## 4. Sicurezza — Credenziali nel repository

⚠️ Il file `appsettings.json` contiene la connection string MongoDB con username e password **in chiaro**. Azioni necessarie:

1. **Rimuovi la connection string** da `appsettings.json` — lascia un valore vuoto o placeholder:
   ```json
   "ConnectionStrings": {
     "MongoDB": ""
   }
   ```
2. **Cambia la password** dell'utente MongoDB su Atlas
3. **Usa solo variabili d'ambiente** su Render per i segreti (`ConnectionStrings__MongoDB`, `Auth__Jwt__Secret`, `Seed__AdminPassword`)
4. *(Opzionale)* Pulisci la history Git con `git filter-repo` o `BFG Repo-Cleaner` per rimuovere le credenziali dai commit precedenti

---

## 5. Deploy automatico

Una volta collegato il repo, Render effettua il deploy automatico ad ogni push sul branch configurato (di default `main`).

Per disabilitare l'auto-deploy:

- Dashboard → Servizio → **Settings → Build & Deploy → Auto-Deploy** → `No`

---

## 6. Troubleshooting

### Il backend non si avvia

- Controlla i log nella dashboard Render (sezione **Logs**)
- Verifica che tutte le variabili d'ambiente siano impostate
- Assicurati che MongoDB Atlas accetti connessioni da qualsiasi IP (`0.0.0.0/0`) nella Network Access

### Il frontend mostra pagina bianca

- Verifica che `VITE_API_BASE_URL` sia impostata correttamente **prima** del build
- Controlla che la rewrite rule `/* → /index.html` sia attiva

### Errori CORS

- Verifica che `Cors__AllowedOrigins__0` corrisponda **esattamente** all'URL del frontend (protocollo incluso, senza trailing slash)
- Esempio corretto: `https://tama-frontend.onrender.com`
- Esempio errato: `https://tama-frontend.onrender.com/`

### Spin-down su free tier

I servizi gratuiti su Render vanno in sleep dopo 15 minuti di inattività. La prima richiesta dopo lo sleep richiede ~30-60 secondi. Per evitarlo:

- Passa a un piano a pagamento
- Oppure usa un servizio di ping periodico (es. UptimeRobot)

---

## Riepilogo

| Servizio  | Tipo Render      | Root Directory                  | Build Command                       | Output |
| --------- | ---------------- | ------------------------------- | ----------------------------------- | ------ |
| Backend   | **Web Service**  | `apps/backend/Tama.Api`   | Docker (automatico)                 | —      |
| Frontend  | **Static Site**  | `apps/frontend`                 | `pnpm install && pnpm run build`    | `dist` |
