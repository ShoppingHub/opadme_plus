# Epic 06 — Finance Projection

## Obiettivo
Mostrare all'utente la traiettoria dell'area Finance con una proiezione lineare a 30 giorni, senza target, obiettivi o valutazioni — solo la direzione attuale.

---

## Behavior

La schermata Finance è una variante dell'Area Detail dedicata esclusivamente all'area di tipo `finance`. La differenza principale è la presenza di una **linea di proiezione tratteggiata** che prolunga la traiettoria storica verso i prossimi 30 giorni.

Non ci sono campi per inserire obiettivi di risparmio. Non ci sono messaggi del tipo "devi risparmiare X". È pura proiezione di traiettoria.

---

## Flussi

### Visualizzazione Finance
1. L'utente tappa il tab "Finance" nel bottom nav
2. Vede la schermata con il grafico storico (linea continua) + proiezione (linea tratteggiata)
3. Sotto il grafico: label `"30-day estimate based on your current trend."`

### Nessun dato Finance
- Se l'utente non ha ancora un'area di tipo `finance` → empty state con CTA per crearla
- Se l'utente ha l'area ma non ha ancora check-in → empty state con copy specifico

---

## Layout

```
┌─────────────────────────────┐
│ ← Finance                   │  ← Header
├─────────────────────────────┤
│  30d  │  90d  │  365d       │  ← TimeRangeSelector
├─────────────────────────────┤
│                             │
│  [Grafico storico]          │  ← Linea continua
│  [+ Proiezione tratteggiata]│  ← strokeDasharray="4 4"
│                             │
├─────────────────────────────┤
│  "30-day estimate based on  │
│   your current trend."      │  ← Label proiezione
├─────────────────────────────┤
│  Home · Areas · Finance · ⚙ │
└─────────────────────────────┘
```

---

## Specifiche Linea di Proiezione

| Proprietà | Valore |
|---|---|
| Stile | `strokeDasharray="4 4"` |
| Colore | Stesso della linea storica (determinato da slope) |
| Calcolo | Regressione lineare semplice sugli ultimi 30 giorni |
| Durata proiezione | 30 giorni nel futuro |
| Distinzione visiva | Solo lo stile tratteggiato (non colore diverso) |

### Calcolo proiezione (semplice linear regression)
```typescript
// Prendi gli ultimi 30 check-in disponibili
// Calcola la retta di regressione y = mx + b
// Estendi per i prossimi 30 giorni
// Mostra come serie dati separata con strokeDasharray
```

---

## Empty States

### Nessuna area Finance creata
```
Icona: BarChart2 (Lucide), 48px, muted #8C9496
Testo: "Log your first finance check-in to see your trend."
CTA:   "Add Finance area" → naviga ad Add Area con tipo Finance pre-selezionato
```

### Area Finance esistente, nessun check-in
```
Icona: BarChart2 (Lucide), muted
Testo: "Log your first finance check-in to see your trend."
Nessuna CTA
```

---

## Stati UI

| Stato | Comportamento |
|---|---|
| Dati presenti | Grafico storico + linea proiezione tratteggiata |
| Meno di 3 check-in | Solo grafico storico, proiezione non mostrata |
| Nessun dato | Empty state con CTA |
| Loading | Skeleton grafico |

> La proiezione richiede un minimo di 3 punti dati per essere significativa. Sotto questa soglia, non viene mostrata.

---

## Edge Case

- Area Finance non creata → empty state con CTA per creare area con tipo Finance pre-selezionato
- Dati insufficienti per proiezione (< 3 check-in) → solo grafico storico senza linea tratteggiata
- Traiettoria piatta (slope ≈ 0) → proiezione orizzontale, colore neutro `#8C9496`
- Time range 365d con pochi dati → mostra i dati disponibili + proiezione se ≥ 3 punti

---

## Acceptance Criteria

- [ ] La schermata Finance mostra il grafico storico (linea continua)
- [ ] La proiezione appare come linea tratteggiata `strokeDasharray="4 4"` dello stesso colore
- [ ] La proiezione è basata su regressione lineare degli ultimi 30 giorni
- [ ] La label `"30-day estimate based on your current trend."` è visibile sotto il grafico
- [ ] Con meno di 3 check-in, la proiezione non viene mostrata
- [ ] Nessun campo per inserire obiettivi di risparmio
- [ ] Nessun messaggio del tipo "devi risparmiare X" o "sei in ritardo"
- [ ] L'empty state mostra il copy corretto

---

## Copy UI

| Elemento | Stringa |
|---|---|
| Header | `"Finance"` |
| Label proiezione | `"30-day estimate based on your current trend."` |
| Empty state (nessun dato) | `"Log your first finance check-in to see your trend."` |
| CTA empty state | `"Add Finance area"` |

---

## Note UI

- Brand tokens: `brand-system/betonme_brand_system_lovable.md`
- La proiezione usa la stessa palette grafico dell'area (slope-based)
- Mai usare rosso per proiezione negativa — sempre `#BFA37A`

---

## Stories

- `story-06-01` — Layout Finance con grafico storico
- `story-06-02` — Calcolo e rendering linea di proiezione tratteggiata
- `story-06-03` — Empty state Finance con CTA
