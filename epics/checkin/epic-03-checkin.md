# Epic 03 — Check-in Giornaliero

## Obiettivo
Permettere all'utente di registrare l'osservazione del giorno per un'area in meno di 3 secondi, senza interruzioni e senza feedback valutativo.

---

## Behavior

### Aree standard (`tracking_mode = binary`)
Il check-in è un'azione binaria: fatto o non fatto. Si esegue dalla Dashboard o dall'Area Detail. L'unico feedback è il cambio di stato del bottone e l'animazione del grafico. Nessun popup, nessun toast, nessuna notifica positiva.

Un solo check-in è possibile per area per giorno (constraint: `UNIQUE(area_id, date)`).

### Aree di riduzione quantitativa (`tracking_mode = quantity_reduce`)
Il tracciamento non è binario: l'utente registra **quante volte** ha eseguito il comportamento oggi. L'interazione primaria è il tap su **+1** — estremamente rapida, zero frizione. Il sistema salva un totale giornaliero in `habit_quantity_daily`.

Non esiste un flusso di conferma fine giornata. L'utente può modificare il totale in qualsiasi momento.

---

## Flussi

### Flow principale — Check-in binario dalla Dashboard (< 3 secondi)
1. L'utente apre l'app
2. Vede la Dashboard con i grafici delle proprie aree
3. Tappa "Log today" sulla card dell'area desiderata
4. Il bottone mostra uno spinner (stato loading)
5. Il check-in viene registrato su Supabase (`completed = true`)
6. Il bottone transiziona a "Observed" (300ms Framer Motion)
7. Il grafico della card anima l'aggiornamento (300ms)
8. Fatto — nessun altro feedback

### Check-in binario già effettuato
- Al caricamento della Dashboard, il bottone mostra già "Observed"
- Il bottone "Observed" non è tappabile (nessuna azione al tap)

### Flow principale — Tracciamento quantitativo (< 2 secondi)

**Azione primaria: +1**
1. L'utente vede la card dell'area `quantity_reduce` in Dashboard o Area Detail
2. Tappa **+1** — nessun altro passaggio
3. Il contatore giornaliero si aggiorna immediatamente (ottimistico)
4. Il sistema salva/aggiorna il record `habit_quantity_daily` per oggi
5. La card mostra il nuovo totale: es. `"today: 4"`

**Azione secondaria: –1**
- Un tap su **–1** decrementa il totale di 1 (minimo: 0, non può andare sotto zero)

**Modifica manuale del totale**
- L'utente può toccare il numero del totale → si apre un input numerico inline
- L'utente inserisce il valore corretto → salva su blur o tap "Done"
- Funziona sia per oggi che per giorni passati

**Modifica giorni passati**
- Dall'Area Detail, l'utente seleziona un giorno passato nel calendario
- Il sistema carica il totale di quel giorno (o 0 se non registrato)
- L'utente può usare +1 / –1 / modifica manuale per aggiornarlo

---

## Specifiche CheckInButton (binario)

| Stato | Stile | Testo |
|---|---|---|
| Idle | `border border-[#7DA3A0] text-[#EAEAEA] bg-transparent` | `"Log today"` |
| Loading | `opacity-50 cursor-not-allowed` + spinner inline | — |
| Completed | `bg-[#7DA3A0]/20 text-[#7DA3A0] border border-[#7DA3A0]` | `"Observed"` |

**Tutti gli stati:** `min-h-[44px] w-full rounded-lg text-[16px] font-medium`

---

## Specifiche QuantityCounter (quantity_reduce)

Il componente sostituisce il CheckInButton per le aree `quantity_reduce`.

**Layout card in Dashboard:**
```
[Nome area]
[today: N]    [–]  [N]  [+1]
```

| Elemento | Comportamento |
|---|---|
| `today: N` | Totale giornaliero, si aggiorna in tempo reale |
| `+1` | Tap → incrementa di 1, save ottimistico |
| `–` | Tap → decrementa di 1 (min 0), save ottimistico |
| Numero `N` | Tap → input numerico inline per modifica manuale |

**Copy microcopy (neutro, non valutativo):**
- `"today: 4"` — totale del giorno
- `"recorded"` — conferma salvataggio (sostituisce toasts, inline, 1s poi scompare)

**Non usare:** "goal reached", "great job", "you're doing better", "failure", "worsening"

**Comportamento di valutazione interno (non mostrato in UI):**
```
se history_days < 7:
    reference_qty = baseline_initial
altrimenti:
    reference_qty = media_mobile(ultimi_7_giorni)

delta = qty_oggi - reference_qty
→ improving  : qty_oggi < reference_qty
→ stable     : qty_oggi ≈ reference_qty  (±10% del reference)
→ worsening  : qty_oggi > reference_qty
```
Questi stati sono segnali interni — **non vengono visualizzati come giudizio**. Potranno alimentare future feature di osservazione.

---

## Logica Traiettoria (backend)

Al momento del check-in, viene calcolato e salvato il `trajectory_state` in `score_daily`.

Il sistema usa un modello **EWMA (Exponentially Weighted Moving Average)** descritto in dettaglio in `architecture/behavioral-trajectory-model.md`.

### Segnale giornaliero (`daily_score`)

```
completed = true         → daily_score = +1.0
primo giorno mancato     → daily_score =  0.0
secondo giorno mancato   → daily_score = -0.5
terzo+ giorno mancato    → daily_score = -1.0
```

### Stato traiettoria (`trajectory_state`)

```
trajectory_state_t = trajectory_state_(t-1) + α × (daily_score - trajectory_state_(t-1))
α = 0.08  (memoria effettiva ≈ 25–30 giorni)
```

Il `trajectory_state` alimenta il grafico. Non viene mai mostrato come numero (default `settings_score_visible = false`).

> Vedi `architecture/behavioral-trajectory-model.md` per la documentazione completa del modello, la razionale, i valori di riferimento e le note di implementazione.

---

## Stati UI

| Stato | Comportamento |
|---|---|
| Non loggato oggi | Bottone "Log today", stile idle |
| In caricamento | Bottone con spinner, opacity-50 |
| Loggato oggi | Bottone "Observed", stile completed, non tappabile |
| Errore di rete | Bottone torna idle, errore non bloccante in basso |

---

## Edge Case

**Aree binarie:**
- Tap multipli rapidi → solo il primo check-in viene registrato (stato loading blocca tap aggiuntivi)
- Check-in offline → operazione in coda, sincronizzata al rientro online
- Check-in a mezzanotte passata → si riferisce sempre alla data locale dell'utente
- Area archiviata → il bottone check-in non è visibile
- `cumulative_score` visualizzato come numero solo se `settings_score_visible = true`

**Aree quantity_reduce:**
- Tap +1 su totale già 0 → incrementa a 1 (zero è il minimo a riposo, non un blocco all'incremento)
- Tap –1 su totale = 0 → nessuna azione, il bottone è disabilitato o ignorato silenziosamente
- Modifica manuale con valore negativo → valore azzerato a 0
- Nessun record in `habit_quantity_daily` per oggi → il counter mostra 0 (lazy create al primo +1)
- Giorno futuro → counter non interattivo
- Area archiviata → counter non visibile

---

## Acceptance Criteria

**Aree binarie:**
- [ ] Tap su "Log today" registra `completed = true` per l'area e la data corrente
- [ ] Il bottone transiziona da idle → loading → completed senza refresh della pagina
- [ ] Il grafico anima l'aggiornamento entro 300ms dal check-in
- [ ] Nessun toast, popup o messaggio positivo al completamento
- [ ] Al reload della Dashboard, il bottone mostra già "Observed" se check-in già effettuato
- [ ] Il bottone "Observed" non risponde al tap
- [ ] Il constraint UNIQUE(area_id, date) impedisce duplicati a livello database

**Aree quantity_reduce:**
- [ ] Tap +1 incrementa il totale e salva/aggiorna `habit_quantity_daily` (UNIQUE area_id, date)
- [ ] Il totale si aggiorna ottimisticamente in UI prima della conferma Supabase
- [ ] Tap –1 decrementa il totale (minimo 0)
- [ ] Modifica manuale del numero salva il valore corretto
- [ ] I giorni passati sono modificabili da Area Detail
- [ ] Il counter mostra 0 se non esiste record per oggi
- [ ] Nessun feedback valutativo nel microcopy

---

## Note UI

- Il feedback è esclusivamente visivo: cambio stato bottone + animazione grafico
- Framer Motion: `animate={{ opacity: isCompleted ? 0.8 : 1 }}` sul bottone
- Framer Motion: transizione 300ms `ease-in-out` sul grafico dopo aggiornamento dati
- Brand tokens: `brand-system/betonme_brand_system_lovable.md`

---

## Stories

- `story-03-01` — CheckInButton con tre stati (idle, loading, completed)
- `story-03-02` — Integrazione Supabase: salvataggio check-in e calcolo score
- `story-03-03` — Animazione grafico post check-in (Framer Motion 300ms)
- `story-03-04` — QuantityCounter: UI +1 / –1 / modifica manuale per aree quantity_reduce
- `story-03-05` — Integrazione Supabase: tabella habit_quantity_daily + logica di riferimento interna
