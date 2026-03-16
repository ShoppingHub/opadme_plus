# Epic 12 — Progress (Osservazione Traiettoria)

## Obiettivo
Offrire all'utente una visione storica dell'andamento complessivo del proprio benessere, con la possibilità di filtrare per macro-area e selezionare il range temporale di osservazione.

> **Nota architetturale:** Questa è la tab Progress (Tab 3). Il contenuto di questo epic è stato estratto dall'Epic 02 (Dashboard), che nella v1 conteneva il grafico aggregato. Con la navigazione v2, Home è diventata un hub giornaliero e il grafico aggregato trova la sua sede naturale qui.

---

## Behavior

La schermata Progress mostra un unico grafico aggregato che rappresenta l'andamento totale dell'utente — media del `trajectory_state` di tutte le aree attive. Il `trajectory_state` è calcolato con il modello EWMA descritto in `architecture/behavioral-trajectory-model.md`. È il layer di osservazione del prodotto.

Sotto il grafico, un selettore di macro-area permette di filtrare la traiettoria per una singola categoria. Il time range è controllabile con i pill 30d / 90d / 365d.

Progress non contiene azioni. Non si fa il check-in qui. Non si gestiscono aree. È solo osservazione.

---

## Flussi

### Visualizzazione Progress — Vista Totale
1. L'utente tappa la tab Progress
2. Vede il grafico totale aggregato (default: 30d, macro-area: Tutto)
3. Il grafico usa la media del `trajectory_state` di tutte le aree attive

### Filtro per Macro-area
1. L'utente tappa una macro-area nel selettore
2. Il grafico si aggiorna mostrando solo le aree di quel tipo
3. Tap su `Tutto / All` → riporta alla vista aggregata completa

### Cambio Time Range
1. L'utente tappa un pill (30d / 90d / 365d)
2. Il grafico si aggiorna con il range selezionato
3. La selezione macro-area rimane attiva

---

## Layout

```
┌─────────────────────────────┐
│ Progress                    │  ← Header
├─────────────────────────────┤
│  30d  │  90d  │  365d       │  ← TimeRangeSelector
├─────────────────────────────┤
│                             │
│   [Grafico totale — 60vh]   │
│                             │
├─────────────────────────────┤
│ Tutto│Salute│Studio│Riduci│€ │  ← MacroAreaSelector
├─────────────────────────────┤
│  [Nav]                      │
└─────────────────────────────┘
```

### MacroAreaSelector

| Proprietà | Valore |
|---|---|
| Tipo | Pill row scorrevole orizzontalmente |
| Opzioni | Tutto · Salute · Studio · Riduci · Finanze (IT) / All · Health · Study · Reduce · Finance (EN) |
| Default | `Tutto / All` attivo |
| Pill attiva | Background `#7DA3A0`, testo dark |
| Pill inattiva | Background trasparente, bordo sottile, testo `#B9C0C1` |

---

## Specifiche Grafico

| Proprietà | Valore |
|---|---|
| Componente | Recharts `<LineChart>` |
| Tipo linea | `type="monotone"` |
| Dati | Media di `trajectory_state` di tutte le aree attive (o filtrate per macro-area) — vedi `architecture/behavioral-trajectory-model.md` |
| Colore linea | Slope del periodo corrente → `#7DA3A0` / `#8C9496` / `#BFA37A` |
| Griglia | Solo orizzontale, `opacity-10` |
| Asse Y | Nascosto |
| Asse X | Tick adattivi (vedi granularità) |
| Altezza | 60vh |
| Background | `bg-[#1F4A50] rounded-xl p-4` |

```typescript
function getLineColor(slope: number): string {
  if (slope > 0.1)  return '#7DA3A0'; // positivo
  if (slope < -0.1) return '#BFA37A'; // declino — mai rosso
  return '#8C9496';                   // neutro
}
```

### Granularità adattiva

La granularità non dipende dal range selezionato ma dallo **span effettivo dei dati da visualizzare**. Questo gestisce correttamente anche il range "All" che cresce nel tempo.

| Span dei dati | Granularità | Dato per punto |
|---|---|---|
| ≤ 90 giorni | Giornaliera | `trajectory_state` del giorno |
| 91 giorni – 2 anni | Settimanale | `trajectory_state` dell'ultimo giorno della settimana |
| > 2 anni | Mensile | `trajectory_state` dell'ultimo giorno del mese |

**Corrispondenza pratica con i range UI:**

| Range selezionato | Granularità tipica | Punti circa |
|---|---|---|
| 15d | Giornaliera | 15 |
| 1m | Giornaliera | 30 |
| 3m | Giornaliera | 90 |
| 6m | Settimanale | 26 |
| 1y | Settimanale | 52 |
| All ≤ 2 anni | Settimanale | ≤ 104 |
| All > 2 anni | Mensile | ~30–60 |

**Rationale:** la granularità è scelta per mantenere 30–80 punti visibili nel grafico, indipendentemente dalla lunghezza della storia. Il valore di fine periodo (settimana o mese) del `trajectory_state` è sufficiente perché l'EWMA incorpora già tutta la storia con il peso corretto — non è necessario ricalcolare una media dei valori già mediati.

### Tooltip / puntatore

Il tooltip si adatta alla granularità attiva:

**Granularità giornaliera:**
```
12 mar 2026
Traiettoria: 0.72
```

**Granularità settimanale:**
```
Settimana del 10 mar 2026
↑ in salita
Traiettoria: 0.72
```

**Granularità mensile:**
```
Marzo 2026
↑ in salita
Traiettoria: 0.72
```

La freccia di tendenza (settimanale e mensile) usa la slope intra-periodo:
- `↑` se slope > 0.1
- `→` se neutro (tra -0.1 e 0.1)
- `↓` se slope < -0.1

Non usare colori nel tooltip per indicare direzione — solo il simbolo freccia.

### Colore linea — adattamento alla granularità

- **Giornaliera:** slope calcolata sugli ultimi 7 punti
- **Settimanale:** slope calcolata sulle ultime 4 settimane
- **Mensile:** slope calcolata sugli ultimi 3 mesi

---

## Stati UI

### Empty State (nessuna area)
```
Icona:    Eye (Lucide), 48px, #7DA3A0
Testo IT: "Nessun dato ancora."
Testo EN: "No data yet."
Sub IT:   "Aggiungi un'area in Attività e inizia a registrare."
Sub EN:   "Add an area in Activities and start logging."
CTA IT:   "Vai ad Attività"
CTA EN:   "Go to Activities"
→ naviga a /activities
```

### Loading State
```
Skeleton: bg-[#1F4A50] animate-pulse rounded-xl h-[60vh]
Skeleton MacroAreaSelector: 5 pill muted
```

### Macro-area senza dati
```
Il grafico mostra un messaggio testuale "Nessun dato per questa categoria / No data for this category yet"
```

---

## Edge Case

- Utente con 1 sola area → grafico totale e filtrato coincidono
- Aree solo in 2 macro-categorie → le altre pill mostrano messaggio "no data"
- Time range 365d senza dati sufficienti → grafico con i dati disponibili, senza errori
- Nessuna connessione → dati cached mostrati, errore non bloccante

---

## Acceptance Criteria

- [ ] La tab Progress mostra un unico grafico aggregato
- [ ] Il grafico usa la media del `trajectory_state` (EWMA) di tutte le aree attive
- [ ] Il MacroAreaSelector ha 5 opzioni nella lingua corrente
- [ ] La selezione di una macro-area filtra il grafico
- [ ] `Tutto / All` è l'opzione di default
- [ ] Il TimeRangeSelector (30d / 90d / 365d) aggiorna il grafico mantenendo la selezione macro-area
- [ ] L'empty state mostra il copy corretto con CTA verso Attività
- [ ] Il loading state mostra skeleton animate-pulse
- [ ] Nessuna azione (check-in, modifica area) è possibile da questa schermata

---

## Note UI

- Progress è solo osservazione — nessun bottone di azione
- Il grafico è identico a quello dell'ex Dashboard (Epic 02 v1)
- Brand tokens: `brand-system/brand_system.md`

---

## Migrazione da Epic 02

La logica del grafico aggregato e del `MacroAreaSelector` è già implementata in `src/pages/Index.tsx`. Per implementare questo epic:
1. Crea `src/pages/Progress.tsx`
2. Sposta il grafico e il `MacroAreaSelector` da `Index.tsx` a `Progress.tsx`
3. Aggiungi route `/progress` nel router
4. L'`Index.tsx` viene riscritto come Home (Epic 02 v2)

---

## Dipendenze

- Epic 08 (i18n) — label IT/EN per MacroAreaSelector e stati UI
- Epic 09 (Layout) — tab Progress nella nav

---

## Stories

- `story-12-01` — Layout Progress: grafico aggregato + TimeRangeSelector
- `story-12-02` — MacroAreaSelector: filtro per macro-area
- `story-12-03` — Empty state e loading state
