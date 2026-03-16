# Stories — Epic 03 — Check-in Giornaliero

## Sequenza di implementazione

```
story-03-01 → CheckInButton: tre stati (idle / loading / completed)
story-03-02 → Integrazione Supabase: salvataggio check-in e calcolo score
story-03-03 → Animazione grafico post check-in (Framer Motion 300ms)
story-03-04 → QuantityCounter: UI +1 / –1 / modifica manuale per aree quantity_reduce
story-03-05 → Integrazione Supabase: tabella habit_quantity_daily + logica di valutazione interna
```

> **Dipendenze:** Richiede Epic 02 (Dashboard + TrajectoryCard) completato. Il bottone vive all'interno della `<TrajectoryCard>`.

---

## story-03-01 — CheckInButton: tre stati

opad.me è un'app di osservazione del benessere. Costruisci il componente `<CheckInButton>` che appare in ogni card attività nella Home.

**Tre stati del bottone:**

| Stato | Label | Stile |
|---|---|---|
| Idle (non loggato oggi) | `"Log today"` | Bordo sottile, testo bianco, sfondo trasparente |
| Loading (invio in corso) | — | Spinner inline, opacity ridotta, non tappabile |
| Completed (già loggato oggi) | `"Observed"` | Sfondo teal semi-trasparente, testo teal, non tappabile |

**Behavior:**
- Il bottone ha `min-height: 44px` e occupa tutta la larghezza della card (full-width)
- Stato loading: blocca tap aggiuntivi (impedisce doppio invio)
- Stato completed: nessuna azione al tap
- Al caricamento della Dashboard, il bottone mostra già `"Observed"` se il check-in per oggi è già presente in `checkins`

**Cosa NON fare dopo il check-in:**
- Nessun toast
- Nessun popup
- Nessun messaggio di congratulazioni
- Nessuna animazione celebrativa

---

## story-03-02 — Integrazione Supabase: salvataggio check-in e traiettoria EWMA

Continua l'Epic 03 di opad.me. Implementa la logica di backend per il check-in con il modello di traiettoria EWMA.

**Al tap su `"Log today"`:**
1. Transizione a stato loading
2. Inserimento record in `checkins`: `{ area_id, user_id, date: oggi (data locale), completed: true }` — constraint UNIQUE(area_id, date) impedisce duplicati
3. Calcolo e inserimento/aggiornamento in `score_daily`:

**Segnale giornaliero (`daily_score`):**
```
completed = true          → daily_score = +1.0
primo giorno mancato      → daily_score =  0.0
secondo giorno mancato    → daily_score = -0.5
terzo+ giorno mancato     → daily_score = -1.0
```

**Stato traiettoria (`trajectory_state`) — modello EWMA:**
```
α = 0.08
trajectory_state_t = trajectory_state_(t-1) + α × (daily_score - trajectory_state_(t-1))

Se non esiste un record precedente per l'area: trajectory_state_(t-1) = 0.0
```

4. Transizione a stato completed

**Schema tabella `score_daily` (aggiornato):**
```
id                 uuid PK
area_id            uuid FK → areas.id
user_id            uuid FK → users.id
date               date
daily_score        float    ← segnale giornaliero (+1, 0, -0.5, -1)
trajectory_state   float    ← stato EWMA calcolato
consecutive_missed integer
UNIQUE(area_id, date)
```

**Edge case:**
- Errore di rete → il bottone torna a stato idle, errore inline non bloccante a fondo schermata (non un popup)
- La `date` usata è sempre la data locale del dispositivo dell'utente (non UTC)
- Se esiste già `cumulative_score` nella tabella: mantenere la colonna per compatibilità ma non usarla nei grafici

**Note:**
- Il `trajectory_state` alimenta il grafico. Non viene mai mostrato come numero (default `settings_score_visible = false`)
- Vedere `architecture/behavioral-trajectory-model.md` per la documentazione completa del modello

---

## story-03-03 — Animazione grafico post check-in

Continua l'Epic 03 di opad.me. Aggiungi l'animazione del grafico dopo il check-in.

**Behavior:**
- Dopo il salvataggio riuscito del check-in, la card attività si aggiorna visivamente
- Transizione Framer Motion 300ms ease-in-out sul cambio stato del bottone
- Il grafico in Progress si aggiorna al prossimo caricamento (non in real-time)

**Cosa NON fare:**
- Nessuna animazione che celebri il completamento (nessun pulse, glow, confetti)
- L'animazione è solo il naturale aggiornamento del dato nel grafico

---

## story-03-04 — QuantityCounter: UI per aree quantity_reduce ⏳

Continua l'Epic 03 di opad.me. Costruisci il componente `<QuantityCounter>` per le aree con `tracking_mode = quantity_reduce`. Sostituisce il `<CheckInButton>` per questo tipo di area.

**Cosa mostra (all'interno della card area in Dashboard / Area Detail):**
- Testo `"today: N"` — totale giornaliero corrente (N = 0 se nessun dato per oggi)
- Bottone **–** a sinistra
- Numero N centrale (tappabile per modifica manuale)
- Bottone **+1** a destra

**Behavior:**
- Tap **+1** → totale +1, aggiornamento ottimistico in UI, salvataggio asincrono
- Tap **–** → totale –1 (minimo 0); se totale = 0, il bottone è disabilitato silenziosamente
- Tap sul **numero** → si apre un input numerico inline al posto del numero; l'utente inserisce il valore desiderato; si salva su blur o tap "Done" sulla tastiera; il valore minimo accettato è 0
- Dopo ogni salvataggio: microcopy `"recorded"` appare inline per ~1 secondo, poi scompare
- Giorno futuro → componente non interattivo (read-only)
- Giorno passato (da Area Detail) → completamente interattivo (stesse azioni di oggi)

**Stile:**
- Il testo `"today: N"` è secondario rispetto al nome area — dimensione ridotta, colore `#B9C0C1`
- I bottoni +1 e – hanno `min-height: 44px` per accessibilità touch
- Il colore del +1 usa il tono Reduce del brand (`#BFA37A`)

**Cosa NON mostrare:**
- Nessun indicatore di "miglioramento" o "peggioramento"
- Nessun confronto con il baseline o la media
- Nessun colore che cambia in base alla performance
- Nessun messaggio del tipo "goal reached", "you did better"

---

## story-03-05 — Integrazione Supabase: habit_quantity_daily ⏳

Continua l'Epic 03 di opad.me. Implementa la persistenza dei dati quantitativi per le aree `quantity_reduce`.

**Tabella da creare: `habit_quantity_daily`**
```
id         uuid PK
area_id    uuid FK → areas.id
date       date
quantity   integer ≥ 0
source     text CHECK (source IN ('quick_add', 'manual_edit', 'system_init'))
created_at timestamptz
updated_at timestamptz
UNIQUE(area_id, date)
```

**Al tap +1 da Dashboard (quick_add):**
- Se esiste record per (area_id, oggi) → `UPDATE quantity = quantity + 1`, `source = 'quick_add'`, `updated_at = now()`
- Se non esiste → `INSERT (area_id, date=oggi, quantity=1, source='quick_add')`

**Al tap –1:**
- `UPDATE quantity = MAX(quantity - 1, 0)`, `source = 'manual_edit'`

**Alla modifica manuale:**
- `UPDATE quantity = valore_inserito`, `source = 'manual_edit'`

**Al caricamento del componente:**
- Query `habit_quantity_daily` WHERE area_id = X AND date = selectedDay
- Se nessun record → mostra 0 senza creare il record (il record si crea solo al primo +1)

**Logica di valutazione interna (segnale, non mostrato in UI):**
```
history_days = COUNT(DISTINCT date) FROM habit_quantity_daily WHERE area_id = X

se history_days < 7:
    reference_qty = areas.baseline_initial
altrimenti:
    reference_qty = AVG(quantity) delle ultime 7 date con record

delta = qty_oggi - reference_qty
→ improving  : delta < 0
→ stable     : ABS(delta) ≤ reference_qty * 0.10
→ worsening  : delta > 0 e non stable
```

Questi stati possono essere salvati in un campo calcolato o calcolati al momento della query — **non vengono mostrati in UI in questa fase**.

**RLS:** Le stesse policy delle altre tabelle — ogni utente accede solo ai propri record.
