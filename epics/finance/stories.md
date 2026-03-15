# Stories — Epic 06 — Finance Projection

## Sequenza di implementazione

```
story-06-01 → Layout Finance con grafico storico (linea continua)
story-06-02 → Linea di proiezione tratteggiata (regressione lineare 30gg)
story-06-03 → Empty states (nessuna area Finance / nessun check-in)
```

> **Dipendenze:** Richiede Epic 03 (Check-in) e Epic 05 (Add/Edit Area) completati. La schermata usa i dati di `score_daily` filtrati per l'area di tipo `finance`.

---

## story-06-01 — Layout Finance con grafico storico

opad.me è un'app di osservazione del benessere. Costruisci la schermata Finance (route `/finance`).

**Cosa mostra:**
- Header: `"Finance"`
- Selettore time range: 30d / 90d / 365d (stesso stile della Dashboard)
- Grafico Recharts `<LineChart>` con `type="monotone"` — altezza prominente
- Label sotto il grafico: `"30-day estimate based on your current trend."`

**Behavior:**
- La schermata mostra i dati dell'area di tipo `finance` dell'utente
- Se l'utente ha più aree di tipo finance, viene mostrata la prima creata (in MVP non è prevista selezione multipla)
- Il time range selector aggiorna il grafico
- Loading: skeleton grafico animate-pulse

**Note:**
- Nessun campo per inserire obiettivi o target di risparmio
- Nessun testo del tipo "You're behind on your savings goal"
- La proiezione (linea tratteggiata) viene aggiunta nella story-06-02

---

## story-06-02 — Linea di proiezione tratteggiata

Continua Epic 06 di opad.me. Aggiungi la proiezione lineare al grafico Finance.

**Behavior:**
- Calcola una regressione lineare semplice sugli ultimi 30 punti dati di `cumulative_score` disponibili
- Estendi la retta per i prossimi 30 giorni come serie separata nel grafico
- La proiezione usa `strokeDasharray="4 4"` (linea tratteggiata)
- Il colore della linea tratteggiata è lo stesso della linea storica (determinato dallo slope)

**Quando non mostrare la proiezione:**
- Meno di 3 check-in disponibili → solo il grafico storico, nessuna linea tratteggiata
- Nessun messaggio di spiegazione in questo caso

**Note:**
- Mai usare il rosso per una proiezione negativa — sempre `#BFA37A` per il trend in declino
- La proiezione è puramente matematica, senza giudizi o suggerimenti

---

## story-06-03 — Empty states Finance

Continua Epic 06 di opad.me. Aggiungi gli empty state per la schermata Finance.

**Empty state 1 — Nessuna area Finance creata:**
- Icona BarChart2 (Lucide), 48px, colore muted
- Testo: `"Log your first finance check-in to see your trend."`
- CTA: `"Add Finance area"` → naviga al form Add Area (`/areas/new`) con il tipo `finance` pre-selezionato

**Empty state 2 — Area Finance esiste ma nessun check-in:**
- Icona BarChart2 (Lucide), 48px, colore muted
- Testo: `"Log your first finance check-in to see your trend."`
- Nessuna CTA (l'utente deve tornare alla Dashboard per fare il check-in)
