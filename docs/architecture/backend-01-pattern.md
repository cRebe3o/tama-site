# Concetti architetturali — Backend — Livello 1: Panoramica dei pattern

> Questo documento è **tecnologia-agnostico**: spiega i concetti generali. Per come sono applicati concretamente in Tama, vedi [backend-03-nel-progetto](backend-03-nel-progetto.md). Per il confronto approfondito con le alternative, vedi [backend-02-confronto](backend-02-confronto.md).

Elenco dei pattern/paradigmi architetturali rilevanti per il backend, con una definizione minima ciascuno.

| Pattern / concetto | Definizione in una frase |
|---|---|
| **Minimal API** | Stile di ASP.NET Core in cui gli endpoint HTTP sono funzioni registrate direttamente su route (`app.MapGet(...)`), senza classi Controller né attributi di routing. |
| **Vertical Slice Architecture** | Organizzazione del codice per *funzionalità* (una cartella = un'operazione completa, dall'input all'accesso dati) invece che per *layer tecnico* (una cartella = un tipo di responsabilità condivisa da tutta l'app). |
| **Layered Architecture (N-Tier)** | Organizzazione classica a strati orizzontali: Controller → Service → Repository → Database, ognuno usato da tutte le feature dell'app. |
| **Clean Architecture / Onion Architecture** | Variante della Layered Architecture con una regola di dipendenza esplicita: gli strati interni (Domain) non conoscono gli strati esterni (Infrastructure), invertita tramite interfacce (Dependency Inversion). |
| **CQRS (Command Query Responsibility Segregation)** | Separazione esplicita tra operazioni che *modificano* stato (Command) e operazioni che *leggono* stato (Query), spesso con modelli/percorsi di codice diversi per le due cose. |
| **Mediator pattern** | Un oggetto centrale (il mediatore) a cui i chiamanti inviano una richiesta, senza conoscere direttamente chi la gestirà — disaccoppia mittente e destinatario. |
| **Pipeline / Decorator pattern** | Una richiesta attraversa una catena di "involucri" (behavior) che possono eseguire codice prima/dopo l'elaborazione vera, senza che l'elaborazione stessa ne sappia nulla. |
| **Repository pattern** | Astrazione che nasconde i dettagli di accesso ai dati dietro un'interfaccia orientata al dominio (es. `GetByEmailAsync`), invece di esporre query grezze ovunque serva. |
| **Dependency Injection (DI)** | Le dipendenze di una classe (es. un repository) le vengono "iniettate" dall'esterno (di solito dal framework) invece di essere create internamente — abilita testabilità e disaccoppiamento. |
| **Middleware pipeline** | Catena di componenti che processano ogni richiesta HTTP in sequenza (autenticazione, CORS, gestione errori, ecc.) prima che arrivi all'endpoint applicativo. |
| **Document-oriented database** | Modello dati dove ogni record è un documento semi-strutturato (tipicamente JSON/BSON) invece di righe in tabelle relazionali con foreign key rigide. |
| **Embedding e denormalizzazione** | Tecniche di modellazione tipiche dei Document DB: salvare i dati figli *dentro* il documento padre (embedding) e copiare snapshot di campi correlati (denormalizzazione) per leggere tutto con una query sola. |
| **Counter pattern** | Tecnica per generare numeri progressivi in un DB privo di `IDENTITY`/`SEQUENCE`: una collection di contatori aggiornata con un incremento atomico. |
| **DTO (Data Transfer Object)** | Oggetto usato solo per trasportare dati tra livelli/confini (es. tra API e client), distinto dal modello di dominio o dal modello di persistenza. |
| **Service lifetime (DI)** | Durata di vita di un servizio registrato nella DI: Singleton (uno per app), Scoped (uno per richiesta), Transient (uno per risoluzione) — con la regola che un servizio a vita lunga non può catturare uno a vita corta. |
| **Claims-based authorization** | Autorizzazione basata su "affermazioni" (claim) contenute nel token dell'utente (es. `permissions: customers.read`), verificate a runtime, invece di ruoli fissi codificati negli attributi. |

## Come si combinano in un backend moderno

Questi pattern non sono alternative mutuamente esclusive: si combinano. Ad esempio, un'app può usare **Clean Architecture** (organizzazione a strati con regola di dipendenza) **e** **CQRS** al suo interno (nell'Application layer, separando Command e Query) **e** **Mediator** (per instradare Command/Query agli handler) **e** **Repository pattern** (nell'Infrastructure layer). Sono scelte ortogonali che si sommano.

Quello che distingue davvero un'architettura dall'altra, in pratica, è **come è organizzato il filesystem del codice** — ovvero se aprendo la cartella dei sorgenti la prima suddivisione che vedi è per *tipo tecnico* (Controllers/, Services/, Repositories/) o per *funzionalità* (Customers/, Orders/, Users/). Questo è il tema centrale del prossimo documento.
