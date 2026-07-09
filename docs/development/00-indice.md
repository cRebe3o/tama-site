# Tama — Guida allo sviluppo di una nuova feature

Questa documentazione è diversa da [../technical/00-indice.md](../technical/00-indice.md) (che spiega **come è fatto** il codice esistente) e da [../architecture/00-indice.md](../architecture/00-indice.md) (che spiega **perché** è fatto così). Qui l'obiettivo è opposto e più pratico: **partire da zero e arrivare ad avere una feature nuova funzionante**, con ogni file da creare, il suo contenuto reale e il motivo di ogni scelta.

Chi legge questa guida non deve necessariamente conoscere già Tama: deve conoscere le basi di .NET/C# e di Vue/TypeScript. Tutto il resto — le convenzioni specifiche di *questo* progetto — è spiegato qui.

## Esempio guida usato in tutta la documentazione

Per rendere concreta ogni spiegazione, tutti e tre i livelli seguono **lo stesso esempio end-to-end**: aggiungere una sezione di menu **"PLUS"** (allo stesso livello della sezione esistente **"DATI"**) che contiene la gestione degli **EXTRA**, una tabella semplice con due soli campi:

| Descrizione extra | Costo |
|---|---|
| Sopralluogo aggiuntivo | 50,00 |
| Rilievo misure fuori zona | 80,00 |
| Trasporto entro 30 km | 40,00 |
| Pratica detrazione fiscale | 120,00 |
| Certificazione posa | 80,00 |

È deliberatamente il caso più semplice possibile (due campi, nessuna relazione con altre entità, CRUD puro) proprio per poter concentrare l'attenzione sul **processo** invece che sulla complessità del dominio. Una volta capito questo schema, si applica identico a qualunque nuova entità dati di Tama (con l'aggiunta delle parti specifiche — relazioni, calcoli, stampa — se il dominio reale le richiede).

> ⚠️ Questo è un esempio **didattico**. "PLUS"/"EXTRA" non esiste nel codice reale di Tama al momento della stesura di questa guida: è scelto come caso di scuola, non come specifica di una feature da consegnare. Se in futuro si deciderà davvero di costruirla, questa guida è comunque un buon punto di partenza, ma andrà verificata contro lo stato del codice in quel momento (nomi di file, convenzioni, permessi già esistenti possono essere cambiati).

## I tre livelli

| Livello | Documento | A cosa serve |
|---|---|---|
| **1 — Panoramica** | [01-panoramica](01-panoramica.md) | La lista dei passaggi da compiere, in ordine, **senza codice**: cosa si crea, dove, e perché in quell'ordine. Da leggere per primo per avere la mappa mentale completa prima di scrivere una riga. |
| **2 — Dove si inserisce nel progetto** | [02-nel-progetto](02-nel-progetto.md) | Come "PLUS"/"EXTRA" si aggancia alle parti **esistenti** di Tama che non vengono create da zero: il menu laterale, il sistema permessi, il modello di navigazione, le convenzioni di naming — usando la sezione "DATI" e il dominio `accessories` (struttura più simile a EXTRA nel progetto reale) come riferimento concreto. |
| **3 — Dettaglio codice** | [03-dettaglio](03-dettaglio.md) | Ogni file, per intero, con il codice reale (backend .NET + MongoDB, frontend Vue + Pinia), riga per riga dove serve, con una sezione "**perché è fatto così**" per ogni scelta non ovvia. |

Si consiglia di leggerli **in ordine**: il livello 3 dà per scontato tutto quello che è già stato spiegato nei livelli 1 e 2, e non ripete le motivazioni architetturali generali già coperte da [../technical/00-indice.md](../technical/00-indice.md) — quando serve, rimanda lì.

> Ogni documento è disponibile anche in versione `.html` autonoma (stesso nome), con lo stesso stile del resto del sito di documentazione.

## Prerequisiti per seguire questa guida

- Aver letto almeno [../technical/backend-01-panoramica.md](../technical/backend-01-panoramica.md) e [../technical/frontend-01-panoramica.md](../technical/frontend-01-panoramica.md) (elenco tecnologie), per non trovarsi davanti a nomi come "MediatR" o "Pinia" senza sapere cosa sono.
- Ambiente di sviluppo Tama avviato in locale: backend (`dotnet run` su `apps/backend/Tama.Api`) e frontend (`pnpm --filter frontend dev`), con MongoDB raggiungibile (locale o istanza di sviluppo condivisa).
- Un editor con supporto C# e TypeScript (il progetto non richiede altro: nessun tool di scaffolding automatico è usato per aggiungere una feature, tutto è scritto a mano seguendo le convenzioni).

## Come è strutturato ogni "passaggio" nei livelli 2 e 3

Per coerenza, ogni sezione dei documenti segue lo stesso schema, che è anche quello con cui vale la pena leggerli:

1. **Cosa fare** — l'azione concreta (creare un file, modificare una riga).
2. **File coinvolti** — percorso esatto, e se è `NEW` (file nuovo) o `MODIFICA` (file esistente da toccare).
3. **Codice** — il contenuto reale, non pseudocodice.
4. **Perché è fatto così** — la motivazione, con riferimento alla documentazione architetturale quando la spiegazione è generale e non specifica di questo passaggio.
