# Mockup schermate (versione ad alta fedeltà) — Compositore preventivi TAMA

Questa è la versione **ad alta fedeltà** del percorso a schermate: 6 passaggi rappresentati con campi, pulsanti, tabelle e colori simili a come apparirebbero nell'app reale (palette neutra grigio/blu, senza branding specifico).

**I mockup grafici veri e propri sono nel file HTML gemello: [`spec-schermate-mockup.html`](spec-schermate-mockup.html)** — apri quello per vedere l'aspetto reale delle schermate. Questo file `.md` riporta solo la descrizione testuale di ogni schermata, per chi consulta la documentazione fuori da un browser.

Per il dettaglio di ogni singolo campo, vedi [`spec-semplificata.md`](spec-semplificata.md).

## Indice

1. Elenco preventivi
2. Intestazione preventivo (con calcolo automatico prezzo totale)
3. Compilazione di una posizione (con prezzi per componente)
4. Tabella delle posizioni (con prezzo totale a riga)
5. Totale e generazione documento (selezione template Word)
6. Libreria immagini per tipologia (gestione anagrafica)
7. Documento finale stampato
8. Riepilogo del flusso completo

---

## 1. Elenco preventivi

Schermata di apertura: barra superiore con titolo app, pulsante blu in evidenza "+ Nuovo preventivo" in alto a destra, sotto una tabella con colonne Cliente / Data / Riferimento / Totale e un pulsante "Apri" per riga.

## 2. Intestazione preventivo

Un pannello (card) con i campi di testata disposti su più righe:
- **Riga 1**: Cliente (select da anagrafica) / Data (default: data del giorno) / Riferimento (readonly, default: contatore automatico formato NNNN-XXX-YYYY, es. 1001-A00-2026)
- **Riga 2**: Sistema (select da dati-app.xlsx) / Serie (select, dipendente da Sistema)
- **Riga 3**: Colore RAL / Bicolore (radio button No / RAL su RAL / RAL eff. legno)
- **Riga 4**: Contributo spese trasporto € (campo numerico)
- **Riga 5**: Prezzo totale preventivo € (readonly calcolato: Contributo spese trasporto € + somma di tutti i Prezzo totale posizione € delle posizioni compilate)
- **Riga 6**: Note a tutta larghezza

Pulsante primario "Continua → Posizioni" in basso a destra.

## 3. Compilazione di una posizione

Un pannello "Posizione N" con pulsante rosso secondario "Azzera riga" in alto a destra del titolo. Campi disposti in righe:
- **Riga 1**: Riferimento / Quantità / Tipologia
- **Riga 2**: Larghezza / Altezza
- **Riga 3**: Telaio - 4 lati (SX/SUP/DX/INF, quest'ultimo con opzione extra "RIB")
- **Riga 4**: Vetro (select, mostra colonna "Sovrapprezzo (EUR/mq)" nel dropdown) / Sovrapprezzo vetro € (campo numerico)
- **Riga 5**: Traverso / Zoccolo ripiegato (radio Sì/No)
- **Riga 6**: Ferramenta / Serrature / Accessori vari
- **Riga 7**: Prezzo singolo € (campo numerico, obbligatorio)
- **Riga 8**: Profilo aggiuntivo 1 (select) / Prezzo profilo aggiuntivo 1 € (campo numerico, attivo solo se Profilo aggiuntivo 1 ≠ "NO")
- **Riga 9**: Profilo aggiuntivo 2 (select) / Prezzo profilo aggiuntivo 2 € (campo numerico, attivo solo se Profilo aggiuntivo 2 ≠ "NO")
- **Riga 10**: Cambia colore (toggle/checkbox) - se attivato mostra i campi sottostanti:
  - Colore RAL (select, sovrascrive il valore di testata)
  - Bicolore (radio button, sovrascrive il valore di testata)
- **Riga 11**: Prezzo totale € (readonly calcolato: (Prezzo singolo € × Quantità) + Sovrapprezzo vetro € + Prezzo profilo aggiuntivo 1 € + Prezzo profilo aggiuntivo 2 €)

In fondo due pulsanti: "Annulla" (secondario) e "Salva posizione" (primario).

## 4. Tabella delle posizioni

Tabella con colonne Pos. / Tipologia / Qtà / Largh. / Altez. / Vetro / Prezzo totale € / azioni. Le righe compilate mostrano i dati e i pulsanti "Modifica" (secondario) e "Elimina" (rosso); le righe non ancora compilate sono in corsivo grigio con un solo pulsante "Compila". In fondo, pulsante rosso "Azzera tutto" allineato a destra. Nota: il numero di posizioni è dinamico e dipende da quante ne vengono aggiunte dall'utente.

## 5. Totale e generazione documento

Un pannello "Riepilogo" con un'etichetta (badge) che indica quante posizioni sono state compilate. 

**Nota**: Il "Prezzo totale preventivo €" è già visibile e calcolato automaticamente nella testata (Schermata 2) e non va ripetuto qui.

Sotto il badge: 
- **Selezione template Word**: select che mostra una lista di template disponibili (nome, descrizione). Il template con `isdefault=TRUE` viene mostrato come valore di default. La tabella sottostante mostra i nomi dei template e un campo bool che identifica il default.

Pulsante primario "Genera documento" in basso a destra.

## 6. Libreria immagini per tipologia

Gestione anagrafica (CRUD) per associare immagini alle tipologie di prodotto. Tabella con colonne Tipologia / Anteprima / azioni:
- L'anteprima è un riquadro tratteggiato (segnaposto grafico) con etichetta del file, o la scritta "nessuna immagine" se non ancora caricata
- Pulsante "Cambia" (se già presente un'immagine) o "Carica" (se assente), entrambi in evidenza primaria
- Azioni di modifica/eliminazione per gestire le associazioni tipologia-immagine

## 7. Documento finale stampato

Impaginato come una vera pagina di documento (bordo, ombra leggera, intestazione con doppia riga separatrice blu): a sinistra titolo "PREVENTIVO" e cliente, a destra i metadati (data, colore RAL, bicolore, coperture). Segue un blocco per ciascuna posizione compilata: a sinistra un riquadro con l'immagine associata alla tipologia, a destra l'elenco dei campi compilati in formato etichetta/valore (quantità, dimensioni, telaio, vetro, opzioni) — **nessun prezzo di riga**. In fondo alla pagina, allineato a destra e evidenziato con bordo superiore blu, il **Totale preventivo** inserito manualmente.

## 8. Riepilogo del flusso completo

Sequenza di "pillole" collegate da frecce: Elenco preventivi → + Nuovo → Intestazione → Compila posizioni → Tabella e selezione template → Genera documento.

1. L'utente crea un nuovo preventivo. Sistema, Data e Riferimento si compilano automaticamente (con default intelligenti). Cliente si seleziona dall'anagrafica.
2. L'utente sceglie Sistema e Serie (dipendente), definisce i colori (RAL/bicolore) e aggiunge l'importo per spese di trasporto. Il **Prezzo totale preventivo** si calcola automaticamente sommando tutte le posizioni compilate.
3. Compila N posizioni (numero dinamico), compilando per ogni posizione: quantità, dimensioni, componenti (telaio, vetro, traverso, zoccolo, ferramenta, serrature, accessori). Per ogni componente prezzo aggiunge un prezzo singolo e facoltativamente prezzi aggiuntivi (sovrapprezzo vetro, profili).
4. Ogni posizione calcola automaticamente il suo "Prezzo totale" in base ai componenti inseriti.
5. Nella tabella posizioni vede l'elenco con tutti i dati e il prezzo totale per riga.
6. Può azzerare singole posizioni o l'intero preventivo in qualsiasi momento.
7. Seleziona il template Word per l'esportazione (il default è quello con `isdefault=TRUE`).
8. L'app genera il documento con intestazione, posizioni (con relative immagini per tipologia) e totale preventivo finale.
9. Il preventivo resta salvato e accessibile dall'elenco iniziale per essere riaperto, modificato o ristampato.
