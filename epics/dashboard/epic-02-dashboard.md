# Epic 02 — Home (Hub Giornaliero)

## Obiettivo
Offrire all'utente un accesso immediato alle attività da registrare — per oggi o per qualsiasi giorno della settimana corrente. Registrazione, retroattività e note in meno di 3 tap dall'apertura dell'app.

> **Nota architetturale:** Questa è la Home tab (Tab 1). La visualizzazione dei trend storici è nella sezione Progress (Epic 12). Home è interazione, Progress è osservazione.

---

## Behavior

All'apertura l'utente vede il selettore settimanale in cima e la lista delle proprie aree attive per il giorno selezionato (default: oggi). Può registrare o annullare un check-in su qualsiasi giorno passato o sul giorno corrente. I giorni futuri sono visibili ma non interattivi.

Per ogni attività, un'icona note permette di aggiungere un appunto libero legato a quella specifica occorrenza (area + giorno).

---

## Flussi

### Flusso principale — Check-in del giorno selezionato

1. L'utente apre l'app → Home, giorno selezionato = oggi
2. Vede il selettore settimanale + lista attività di oggi
3. Tappa "Fatto" su un'area
4. Il check-in viene registrato → bottone passa a "Osservato ✓" (300ms)

### Navigazione settimana

- Due frecce (`‹` / `›`) ai lati del selettore: saltano alla settimana precedente / successiva
- I 7 giorni della settimana sono mostrati come chip quadrati con bordi stondati (es. `Lu 10`, `Ma 11`…)
- Il giorno selezionato è evidenziato (background attivo)
- Tap su un chip → carica la lista attività di quel giorno
- Giorni futuri: chip grayed out, non tappabili
- Non c'è limite nel navigare indietro nelle settimane precedenti

### Retroattività — giorni passati

- Il selettore mostra lo stato di check-in reale per ogni giorno passato
- L'utente può marcare come "Fatto" un'attività di un giorno passato
- L'utente può anche **annullare** un check-in già registrato (tap su "Osservato ✓" → torna a "Fatto")
- La modifica viene salvata immediatamente (aggiornamento ottimistico)

### Giorni futuri

- I chip dei giorni futuri (dopo oggi) sono grayed out e non tappabili
- Se l'utente tenta un tap, non succede nulla (nessun feedback, nessun errore)

### CTA per area standard vs. area Gym vs. area Reduce quantitativa

**Area standard (tracking_mode = binary o non definito):**
- CTA: `"Fatto"` (IT) / `"Done"` (EN)
- Tap → registra il check-in direttamente

**Area Gym (type health con scheda configurata, Epic 14):**
- CTA: `"Apri scheda"` (IT) / `"Open session"` (EN)
- Tap → naviga alla pagina dedicata della scheda (`/cards/gym`, Epic 14)
- Il check-in viene registrato automaticamente al primo esercizio DONE

**Area Reduce quantitativa (tracking_mode = quantity_reduce, show_quick_add_home = true):**
- Nessuna CTA binaria
- Nella card compare il `<QuantityCounter>`: `today: N` + bottoni **–** · **+1**
- Tap **+1** → incrementa il totale di oggi direttamente dalla Home
- Il counter è la stessa interazione primaria della Area Detail (Epic 03, story-03-04)
- Se `show_quick_add_home = false` → la card mostra solo il nome area e naviga all'Area Detail al tap

### Note per attività

- Ogni card ha un'icona note (matita o foglio) in fondo a destra
- Tap sull'icona → apre un campo testo inline (espansione della card) o modal leggero
- Testo libero, max **1500 caratteri** — contatore caratteri visibile
- La nota è salvata legata ad `area_id + date` (una nota per occorrenza)
- Se esiste già una nota per quella occorrenza, l'icona è visivamente distinta (es. filled vs. outline)
- Tap sull'icona con nota esistente → apre il testo precompilato, modificabile
- Il salvataggio avviene on-blur o con tasto "Salva" / "Save"

### Indicatore Gym Day

- Se esiste un'area Gym con scheda e la sessione del giorno selezionato non è ancora completata:
  - Una riga nella card mostra il nome del giorno scheda (es. `"Giorno 2 — Gambe →"`)
  - Stile secondario, tap → naviga ad Area Detail gym

### Empty state (nessuna area)

- Messaggio neutro + CTA per andare ad Attività e creare la prima area

---

## Layout

```
┌─────────────────────────────────────────┐
│ opad.me                                 │  ← Header (wordmark)
├─────────────────────────────────────────┤
│  ‹   Lu  Ma  Me  Gi  Ve  Sa  Do   ›    │  ← Selettore settimanale
│      10  11  12  13  14  15  16        │    chip stondati, giorno attivo evidenziato
│     [ ] [ ] [■] [ ] [·] [·] [·]       │    · = futuro (grayed), ■ = oggi selezionato
├─────────────────────────────────────────┤
│                                         │
│  ┌───────────────────────────────────┐  │
│  │ Palestra                          │  │  ← Area card (binary gym)
│  │ Giorno 2 — Gambe →               │  │  ← Indicatore gym day (se presente)
│  │ [Apri scheda]           [📝]      │  │  ← CTA gym + icona note
│  └───────────────────────────────────┘  │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │ Studio                            │  │  ← Area card (binary standard)
│  │ [Osservato ✓]           [📝●]     │  │  ← Già loggato, nota presente (filled)
│  └───────────────────────────────────┘  │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │ Sigarette          today: 4       │  │  ← Area card (quantity_reduce)
│  │                 [–]  4  [+1]      │  │  ← QuantityCounter — nessuna CTA binaria
│  └───────────────────────────────────┘  │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │ Social media                      │  │  ← Area card (binary standard)
│  │ [Fatto]                 [📝]      │  │
│  └───────────────────────────────────┘  │
│                                         │
├─────────────────────────────────────────┤
│  [Nav]                                  │
└─────────────────────────────────────────┘
```

---

## Area Card nella Home

| Proprietà | Valore |
|---|---|
| Background | `bg-[#1F4A50] rounded-xl p-4` |
| Nome area | Testo principale, `text-[#EAEAEA]` |
| Indicatore gym day | Testo piccolo `text-[#B9C0C1]`, tap → Area Detail gym |
| CTA standard (binary) | `"Fatto"` / `"Done"` — segue stati Epic 03 |
| CTA gym | `"Apri scheda"` / `"Open session"` → naviga Area Detail gym |
| QuantityCounter (quantity_reduce) | `today: N` + bottoni **–** · **+1** — nessuna CTA binaria |
| Icona note | Outline se nessuna nota, filled/colorata se nota presente (solo aree binary) |
| Ordine aree | Ordine di creazione dell'utente |

> Le aree `quantity_reduce` non mostrano l'icona note nella Home — la nota è accessibile dall'Area Detail.

---

## Selettore Settimanale

| Proprietà | Valore |
|---|---|
| Contenitore | Row orizzontale, centrata, con frecce `‹` `›` ai lati |
| Chip giorno | Quadrato con bordi stondati — mostra giorno abbreviato + numero (es. `Lu\n10`) |
| Stato: oggi | Evidenziato con `bg-[#1F4A50]` o bordo attivo |
| Stato: selezionato (non oggi) | Evidenziato differente (bordo o bg più tenue) |
| Stato: futuro | `opacity-30`, non tappabile |
| Stato: passato con check-in | Piccolo indicatore (punto verde o simile) sotto il numero |
| Frecce `‹` `›` | Saltano di 7 giorni interi — non si può andare oltre la settimana in corso verso il futuro |

---

## Stati UI

### Empty State (nessuna area)
```
Icona:    Eye (Lucide), 48px, #7DA3A0
Testo IT: "Cosa vuoi osservare?"
Testo EN: "What do you want to observe?"
Sub IT:   "Aggiungi un'area in Attività per iniziare."
Sub EN:   "Add an area in Activities to start observing."
CTA IT:   "Vai ad Attività"
CTA EN:   "Go to Activities"
→ naviga a /activities
```

### Loading State
```
Skeleton: 2-3 card bg-[#1F4A50] animate-pulse rounded-xl h-24
```

### Tutto loggato per il giorno selezionato
```
Messaggio IT: "Tutto registrato per questo giorno."
Messaggio EN: "All logged for this day."
Stile: testo piccolo centrato, text-[#B9C0C1]
```

---

## Edge Case

- Area archiviata → non compare mai nella lista
- Più di 5 aree → scroll verticale naturale
- Area Gym senza scheda configurata → CTA standard `"Fatto"`, nessun indicatore gym day
- Sessione gym già completata per il giorno selezionato → nessun indicatore gym day
- Giorno futuro selezionato (non raggiungibile via tap, solo eventuale stato iniziale) → tutti i CTA disabilitati
- Annullamento check-in su giorno passato → stato torna a idle, nota non viene cancellata automaticamente
- Nota vuota → non salvare (nessun record nel db)
- 1500 caratteri raggiunti → input bloccato, contatore in rosso
- Area `quantity_reduce` + giorno futuro → QuantityCounter disabilitato, nessuna interazione
- Area `quantity_reduce` + `show_quick_add_home = false` → la card non mostra il counter; tap sulla card naviga ad Area Detail
- Area `quantity_reduce` + giorno passato → QuantityCounter interattivo, salva il record per quel giorno

---

## Acceptance Criteria

- [ ] Il selettore settimanale mostra i 7 giorni della settimana corrente come chip stondati
- [ ] Le frecce `‹` `›` saltano alla settimana precedente / successiva (sette giorni interi)
- [ ] I chip dei giorni futuri sono grayed out e non tappabili
- [ ] Il giorno selezionato evidenzia le attività di quel giorno
- [ ] L'utente può marcare "Fatto" su giorni passati
- [ ] L'utente può annullare un check-in su giorni passati
- [ ] La CTA delle aree standard è `"Fatto"` / `"Done"` (non "Log today")
- [ ] La CTA dell'area Gym è `"Apri scheda"` / `"Open session"` → naviga a `/cards/gym` (pagina dedicata scheda, Epic 14)
- [ ] Ogni card binary ha un'icona note che apre un campo testo per quella occorrenza (area + giorno)
- [ ] La nota è salvata con `area_id + date`, max 1500 caratteri
- [ ] L'icona note è visivamente distinta quando una nota è già presente
- [ ] Al caricamento, le aree già loggata nel giorno selezionato mostrano stato "Osservato ✓"
- [ ] L'indicatore gym day appare solo se area gym con scheda e sessione non completata nel giorno selezionato
- [ ] L'empty state mostra il copy corretto nella lingua utente con CTA verso Attività
- [ ] Le aree archiviate non compaiono
- [ ] La Home non contiene grafici (i grafici sono in Progress — Epic 12)
- [ ] Le aree `quantity_reduce` con `show_quick_add_home = true` mostrano il QuantityCounter (`today: N` + **–** · **+1**)
- [ ] Tap +1 da Home aggiorna il totale della area `quantity_reduce` per il giorno selezionato
- [ ] Le aree `quantity_reduce` con `show_quick_add_home = false` non mostrano il counter — tap naviga ad Area Detail

---

## Note UI

- Nessun grafico in Home — la Home è azione, non osservazione
- Nessun feedback celebrativo al check-in — coerente con tono osservativo del prodotto
- Il selettore settimanale non è uno slider/scroll orizzontale: è una row fissa con 7 chip + frecce
- Brand tokens: `brand-system/brand_system.md`

---

## Dipendenze

- Epic 03 (Check-in) — logica check-in e stati bottone
- Epic 08 (i18n) — label IT/EN
- Epic 11 (Gym) — CTA "Apri scheda" e indicatore gym day
- Epic 14 (Schede) — CTA Gym naviga a `/cards/gym` (pagina dedicata scheda)
- Epic 12 (Progress) — la visualizzazione dei trend è stata spostata lì

---

## Stato implementazione

**Da refactorare** — l'attuale `src/pages/Index.tsx` va completamente riscritto.

| Componente da creare | Descrizione |
|---|---|
| `src/components/WeekSelector.tsx` | Selettore settimanale con chip e frecce |
| `src/components/TodayActivityList.tsx` | Lista aree per il giorno selezionato |
| `src/components/ActivityNoteInput.tsx` | Campo note inline o modal per area+giorno |
| `src/components/GymDayIndicator.tsx` | Indicatore giorno scheda palestra |

| Dead code da rimuovere | Motivo |
|---|---|
| `src/components/TrajectoryCard.tsx` | Non più usato |
| `src/components/TrajectoryCardSkeleton.tsx` | Non più usato |

Il grafico aggregato e il MacroAreaSelector migrano in `src/pages/Progress.tsx` (Epic 12).

---

## Stories

- `story-02-01` — Layout Home: header + selettore settimanale + lista attività — **da fare**
- `story-02-02` — Stato check-in per ciascuna area (caricamento + ottimistico) — **da fare**
- `story-02-03` — Retroattività: marcare e annullare check-in su giorni passati — **da fare**
- `story-02-04` — Note per attività: icona, campo testo, salvataggio per area+giorno — **da fare**
- `story-02-05` — CTA Gym: "Apri scheda" → Area Detail gym — **da fare**
- `story-02-06` — Indicatore gym day nella Home — **da fare**
- `story-02-07` — Empty state Home (nessuna area → CTA Attività) — **da fare**
- `story-02-08` — QuantityCounter in Home per aree quantity_reduce — **da fare**

> Le story 02-01..02-05 precedenti (grafico aggregato) sono **deprecate**. Il comportamento è stato spostato in Epic 12 (Progress).
