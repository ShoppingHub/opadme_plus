# Epic 11 — Gym Card (Scheda Palestra)

## Obiettivo

Offrire all'utente uno strumento semplice per gestire la propria **scheda palestra strutturata** e registrare i carichi sessione per sessione — senza trasformare BetonMe in una fitness app.

L'interazione principale è una **checklist rapida**: l'utente apre l'app, vede gli esercizi del giorno con i carichi abituali, spunta quelli completati. Fine.

---

## Behavior

Quando un utente crea un'area `health` con nome `Gym` o `Palestra` (case insensitive), l'Area Detail mostra una sezione aggiuntiva: la **Gym Card / Scheda Palestra**.

La Gym Card non sostituisce il check-in binario — quel meccanismo resta invariato per il grafico traiettoria. La Gym Card è una sezione di log strutturato che appare **sotto** il grafico, l'heatmap e il bottone check-in.

---

## Modello mentale

```
scheda
  └── giorno (es. "Giorno 1", "Push", "Full body")
       └── gruppo muscolare (es. "Gambe", "Addominali")
            └── esercizio (es. "Squat", 4×10, 80kg)

sessione
  └── data + giorno scheda
       └── esercizio completato + peso usato
```

Gli esercizi contrassegnati come **giornalieri** (`is_daily = true`) appaiono in ogni sessione, indipendentemente dal giorno corrente della scheda (es. riscaldamento, addominali).

---

## Rilevamento automatico

| Condizione | Comportamento |
|---|---|
| `area.type === "health"` AND nome corrisponde a `gym` o `palestra` (case insensitive) | Mostra la sezione Gym Card |
| Qualsiasi altra area | Sezione non mostrata |

La Gym Card appare automaticamente — l'utente non deve attivarla.

---

## Flusso 1 — Prima apertura (nessuna scheda configurata)

Quando l'utente apre l'area Gym per la prima volta e non esiste ancora una scheda:

1. La sezione Gym Card mostra un **empty state con prompt setup**:
   > `"Configura la tua scheda"` (IT) / `"Set up your workout plan"` (EN)
   > `"Crea la struttura dei tuoi allenamenti settimanali."`
   > CTA: `"Inizia / Get started"`

2. L'utente tappa `"Inizia"` → appare un bottom sheet di configurazione:
   - Campo: `"Quanti giorni di allenamento? / How many training days?"` — numero da 1 a 7
   - Per ogni giorno: input nome (placeholder `"Giorno 1 / Day 1"`, `"Giorno 2 / Day 2"`, ecc.)
   - CTA: `"Crea scheda / Create plan"`

3. Alla conferma la scheda viene creata con i giorni specificati. L'utente viene portato direttamente alla vista sessione del Giorno 1.

---

## Flusso 2 — Gestione scheda (aggiunta esercizi)

Raggiungibile tramite il bottone `"Modifica scheda / Edit plan"` nella sezione Gym Card.

La vista di gestione mostra:
- Lista dei giorni della scheda
- Per ogni giorno: lista dei gruppi muscolari, per ogni gruppo i suoi esercizi
- CTA per aggiungere gruppo muscolare a un giorno
- CTA per aggiungere esercizio a un gruppo
- Possibilità di rinominare e riordinare

### Aggiunta gruppo muscolare
- Bottom sheet con: campo nome gruppo (es. `"Gambe"`, `"Pettorali"`)
- Salvataggio silenzioso

### Aggiunta / modifica esercizio
Bottom sheet con:

| Campo | Tipo | Obbligatorio |
|---|---|---|
| Nome | Testo libero | Sì |
| Serie | Numero intero | Sì |
| Ripetizioni | Numero intero | Sì |
| Peso default (kg) | Numero decimale | No |
| Giornaliero / Daily | Toggle | No — default OFF |

- Toggle `"Giornaliero / Daily"`: se attivo, l'esercizio appare in **ogni sessione** indipendentemente dal giorno (es. riscaldamento, addominali)
- CTA in modifica: `"Disattiva / Deactivate"` (colore `#E24A4A`) → imposta `active = false`, l'esercizio scompare dalle sessioni future ma i dati storici restano

---

## Flusso 3 — Sessione giornaliera

### Selezione del giorno

Quando l'utente apre la Gym Card e non ha ancora fatto una sessione oggi:
- L'app mostra automaticamente il **giorno successivo** all'ultima sessione (rotazione ciclica)
- Se non ci sono sessioni precedenti → mostra Giorno 1
- Un selettore `"Giorno X ▼"` in cima alla sezione permette di cambiare manualmente il giorno
- Se oggi esiste già una sessione → mostra quella sessione

### Layout sessione

```
┌─────────────────────────────┐
│ Scheda / Gym Card           │
│ Sessione di oggi            │
│ Giorno 2 — Gambe  [▼] [✏️] │  ← Selettore giorno + link modifica scheda
├─────────────────────────────┤
│                             │
│  ── Giornalieri ──          │  ← Gruppo speciale is_daily (se presenti)
│  ☐ Plank  3 × 60s          │
│                             │
│  ── Gambe ──                │  ← Gruppi muscolari del giorno
│  ☐ Squat         4 × 80kg  │
│  ☐ Leg press     3 × 70kg  │
│  ☐ Leg curl      3 × 50kg  │
│                             │
│  ── Spalle ──               │
│  ☑ Lento avanti  4 × 30kg  │  ← Completato: riga attenuata + check
│  ☐ Alzate lat.   3 × 12kg  │
│                             │
│  [Storico sessioni ▼]       │
│                             │
└─────────────────────────────┘
```

### Azioni per esercizio

| Azione | Trigger | Effetto |
|---|---|---|
| **DONE** | Tap sul checkbox | Segna l'esercizio come completato per questa sessione, con il `default_weight` corrente. Tap di nuovo → annulla. |
| **EDIT** | Tap sull'icona peso / long press sulla riga | Apre bottom sheet con solo il campo peso precompilato. Al salvataggio: aggiorna `weight_used` nella sessione E aggiorna `default_weight` nell'esercizio della scheda. |
| **DISATTIVA** | Dentro il bottom sheet EDIT | Imposta `active = false` sull'esercizio. Scompare dalle sessioni future. |

Il primo esercizio contrassegnato come DONE per la sessione di oggi → completa automaticamente il check-in del giorno (se non già fatto).

---

## Flusso 4 — Storico sessioni

Sotto la sessione di oggi, una sezione collassabile `"Storico sessioni / Session history"`.

- Lista sessioni precedenti ordinate dalla più recente
- Ogni riga collassata: data + nome giorno + numero esercizi completati (es. `"Lun 3 Mar · Giorno 2 · 8 esercizi"`)
- Tap → espande la lista esercizi con pesi usati (read-only)
- Solo una sessione espansa alla volta
- Se non ci sono sessioni precedenti → sezione nascosta

---

## Database

> Nota: le tabelle `gym_sessions` e `gym_exercises` della v1 vengono **sostituite** da questo schema. I dati della v1 possono essere ignorati (nessun utente reale in produzione al momento della migrazione).

```
gym_programs
- id UUID PK
- area_id UUID FK → areas(id) CASCADE
- user_id UUID FK → auth.users(id) CASCADE
- name TEXT
- created_at TIMESTAMPTZ

gym_program_days
- id UUID PK
- program_id UUID FK → gym_programs(id) CASCADE
- day_index INT          -- 1, 2, 3...
- name TEXT              -- "Giorno 1", "Push", ecc.

gym_muscle_groups
- id UUID PK
- day_id UUID FK → gym_program_days(id) CASCADE
- name TEXT
- order INT

gym_program_exercises
- id UUID PK
- muscle_group_id UUID FK → gym_muscle_groups(id) CASCADE  (nullable se is_daily)
- program_id UUID FK → gym_programs(id) CASCADE             (per gli is_daily non legati a un gruppo)
- name TEXT
- sets INT
- reps INT
- default_weight DECIMAL(6,2) nullable
- is_daily BOOLEAN DEFAULT false
- order INT
- active BOOLEAN DEFAULT true

gym_sessions
- id UUID PK
- program_day_id UUID FK → gym_program_days(id)  (nullable se sessione libera)
- area_id UUID FK → areas(id) CASCADE
- user_id UUID FK → auth.users(id) CASCADE
- date DATE
- UNIQUE(area_id, date)

gym_session_exercises
- id UUID PK
- session_id UUID FK → gym_sessions(id) CASCADE
- program_exercise_id UUID FK → gym_program_exercises(id)
- weight_used DECIMAL(6,2) nullable
- completed BOOLEAN DEFAULT false
```

> RLS: ogni tabella ha policy `user_id = auth.uid()` su SELECT, INSERT, UPDATE, DELETE.

---

## Stati UI

| Stato | Comportamento |
|---|---|
| Nessuna scheda → empty state setup | Prompt con CTA "Inizia" |
| Scheda creata, nessuna sessione oggi | Vista sessione con esercizi tutti non completati |
| Sessione in corso | Checklist con esercizi misti completati/non |
| Tutti gli esercizi completati | Nessun feedback speciale — l'utente chiude semplicemente |
| Esercizio completato | Riga attenuata, checkbox pieno |
| Bottom sheet EDIT aperto | Overlay con campo peso precompilato |
| Modifica scheda | Vista struttura con CTA aggiungi |
| Storico espanso | Lista esercizi read-only |

---

## Edge Case

- Utente rinomina area da "Palestra" a "Corsa" → Gym Card scompare, dati restano nel DB
- Utente riporta il nome a "Palestra" → Gym Card riappare con scheda e storico intatti
- Tutti gli esercizi del giorno disattivati → sessione mostra solo gli is_daily (se presenti), altrimenti messaggio `"Nessun esercizio attivo per questo giorno / No active exercises for this day"`
- Sessione con solo is_daily → completamento check-in funziona normalmente
- `default_weight` aggiornato → sessioni passate non cambiano (storico immutabile)
- Scheda con 1 solo giorno → nessun selettore giorno visibile, sempre quel giorno
- Utente cambia giorno tramite selettore → crea nuova sessione per quel giorno nella stessa data (max 1 sessione per area per data — la precedente viene sovrascritta o l'utente viene avvisato)

---

## Acceptance Criteria

- [ ] La Gym Card appare solo per `area.type === "health"` con nome `gym` o `palestra` (case insensitive)
- [ ] Prima apertura senza scheda → empty state con prompt setup
- [ ] Wizard crea scheda con N giorni nominabili
- [ ] Per ogni giorno è possibile aggiungere gruppi muscolari ed esercizi
- [ ] Gli esercizi `is_daily` appaiono in ogni sessione come gruppo separato
- [ ] La sessione di oggi mostra automaticamente il giorno successivo all'ultima sessione (rotazione)
- [ ] Il selettore giorno permette di cambiare il giorno manualmente
- [ ] Tap checkbox → DONE, secondo tap → annulla DONE
- [ ] DONE con peso corrente → crea record `gym_session_exercises` con `weight_used = default_weight`
- [ ] EDIT peso → aggiorna `weight_used` nella sessione E `default_weight` nell'esercizio
- [ ] DISATTIVA → `active = false`, esercizio scompare dalle sessioni future
- [ ] Primo esercizio DONE → auto check-in del giorno (se non già fatto)
- [ ] Storico sessioni collassabile, read-only, ordinato per data decrescente
- [ ] Scheda modificabile in qualsiasi momento da `"Modifica scheda / Edit plan"`
- [ ] Tutte le label seguono la lingua selezionata (Epic 08)

---

## Copy UI

| Elemento | IT | EN |
|---|---|---|
| Titolo sezione | `Scheda Palestra` | `Gym Card` |
| Sottotitolo sessione | `Sessione di oggi` | `Today's session` |
| Selettore giorno | `Giorno N` | `Day N` |
| Link modifica scheda | `Modifica scheda` | `Edit plan` |
| Empty state titolo | `Configura la tua scheda` | `Set up your workout plan` |
| Empty state sub | `Crea la struttura dei tuoi allenamenti settimanali.` | `Create the structure of your weekly workouts.` |
| CTA setup | `Inizia` | `Get started` |
| Label giorni wizard | `Quanti giorni di allenamento?` | `How many training days?` |
| CTA crea scheda | `Crea scheda` | `Create plan` |
| Gruppo is_daily | `Giornalieri` | `Daily` |
| CTA aggiungi gruppo | `+ Aggiungi gruppo muscolare` | `+ Add muscle group` |
| CTA aggiungi esercizio | `+ Aggiungi esercizio` | `+ Add exercise` |
| Label toggle giornaliero | `Giornaliero` | `Daily` |
| Sub toggle giornaliero | `Appare in ogni sessione` | `Appears in every session` |
| Azione disattiva | `Disattiva esercizio` | `Deactivate exercise` |
| Label peso | `Peso (kg)` | `Weight (kg)` |
| Salva | `Salva` | `Save` |
| Annulla | `Annulla` | `Cancel` |
| Storico | `Storico sessioni` | `Session history` |
| Nessun esercizio attivo | `Nessun esercizio attivo per questo giorno` | `No active exercises for this day` |

---

## Note UI

- L'interazione principale è checklist-like: veloce, pochi tap
- Nessun grafico di volume, nessuna statistica fitness — solo log dei carichi
- Il peso mostrato sulla riga è sempre `default_weight` (peso abituale), non un valore inserito manualmente
- La riga completata appare attenuata (`opacity-50`) con checkbox pieno
- DISATTIVA ha colore testo `#E24A4A` (azione distruttiva)

---

## Dipendenze

- Epic 03 (Check-in) — il check-in si completa automaticamente al primo esercizio DONE
- Epic 04 (Area Detail) — la Gym Card è una sezione aggiuntiva nell'Area Detail
- Epic 08 (i18n) — tutte le label in IT/EN

---

## Stories

- `story-11-01` — Schema DB + wizard setup scheda
- `story-11-02` — Gestione scheda: giorni, gruppi muscolari, esercizi
- `story-11-03` — Sessione giornaliera: visualizzazione checklist e selezione giorno
- `story-11-04` — Azioni esercizio: DONE, EDIT peso, DISATTIVA
- `story-11-05` — Storico sessioni collassabile
