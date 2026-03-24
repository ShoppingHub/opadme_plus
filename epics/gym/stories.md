# Stories — Epic 11 — Gym Card (Scheda Palestra)

## Sequenza di implementazione

```
story-11-01 → Schema DB + wizard setup scheda
story-11-02 → Gestione piano: /cards/gym/edit
story-11-03 → Sessione giornaliera: /cards/gym con checklist e selezione giorno
story-11-04 → Azioni esercizio: DONE, EDIT peso, DISATTIVA
story-11-05 → Storico sessioni collassabile
story-11-06 → Entry point da Attività: bottom sheet con 2 CTA
```

> **Dipendenze:** Richiede Epic 03 (Check-in) per il completamento automatico. Epic 14 (Schede) per il sistema di pagine dedicate. Epic 08 (i18n) per i label.

---

## story-11-01 — Schema DB e wizard setup scheda

opad.me è un'app di osservazione del benessere. Implementa il nuovo schema DB per la Gym Card e il wizard di primo accesso.

**Nuovo schema DB:**

- `gym_programs` — una scheda per area gym (area_id, user_id)
- `gym_program_days` — i giorni di allenamento della scheda (es. Giorno 1, Giorno 2), con `day_index` e `name`
- `gym_muscle_groups` — gruppi muscolari per giorno (es. Gambe, Pettorali), con `order`
- `gym_program_exercises` — esercizi per gruppo muscolare, con `sets`, `reps`, `default_weight` (nullable), `is_daily` (bool), `active` (bool), `order`. `muscle_group_id` è nullable per gli esercizi `is_daily` che non appartengono a un gruppo specifico — in quel caso sono legati direttamente a `program_id`
- `gym_sessions` — sessione giornaliera, legata a un giorno della scheda e a una data. UNIQUE(area_id, date)
- `gym_session_exercises` — esercizi completati in una sessione, con `weight_used` e `completed`

Tutte le tabelle hanno RLS policy per `user_id = auth.uid()`.

**Wizard primo accesso:**

Quando l'utente apre `/cards/gym` o `/cards/gym/edit` e non esiste ancora una `gym_program` per quell'area, la pagina mostra un empty state con:
- Titolo: `"Configura la tua scheda"` (IT) / `"Set up your workout plan"` (EN)
- Sottotitolo: `"Crea la struttura dei tuoi allenamenti settimanali."` (IT) / `"Create the structure of your weekly workouts."` (EN)
- CTA: `"Inizia"` (IT) / `"Get started"` (EN)

Tap su CTA → bottom sheet di configurazione:
1. Campo `"Quanti giorni di allenamento?"` (IT) / `"How many training days?"` (EN) — numero intero 1–7
2. Per ogni giorno: campo nome con placeholder `"Giorno N"` (IT) / `"Day N"` (EN) — modificabile
3. CTA `"Crea scheda"` (IT) / `"Create plan"` (EN) — crea `gym_program` + `gym_program_days`
4. Al salvataggio → redirect a `/cards/gym/edit` per aggiungere gli esercizi

---

### Prompt Lovable — story-11-01

```
opad.me è un'app mobile-first di osservazione del benessere personale.

Implementa lo schema DB e il wizard di setup per la Gym Card.

**Schema DB (Supabase):**
Crea queste tabelle con RLS policy user_id = auth.uid():

- gym_programs: id, area_id (FK areas), user_id (FK auth.users), created_at
- gym_program_days: id, program_id (FK gym_programs CASCADE), day_index INT, name TEXT
- gym_muscle_groups: id, day_id (FK gym_program_days CASCADE), name TEXT, order INT
- gym_program_exercises: id, muscle_group_id (FK gym_muscle_groups, nullable), program_id (FK gym_programs, per is_daily), name TEXT, sets INT, reps INT, default_weight DECIMAL(6,2) nullable, is_daily BOOLEAN DEFAULT false, active BOOLEAN DEFAULT true, order INT
- gym_sessions: id, program_day_id (FK gym_program_days), area_id (FK areas CASCADE), user_id (FK auth.users CASCADE), date DATE. UNIQUE(area_id, date)
- gym_session_exercises: id, session_id (FK gym_sessions CASCADE), program_exercise_id (FK gym_program_exercises), weight_used DECIMAL(6,2) nullable, completed BOOLEAN DEFAULT false

**Wizard primo accesso:**
Quando l'utente apre /cards/gym o /cards/gym/edit e non esiste nessuna gym_program per quell'area gym/palestra, mostra un empty state:
- Titolo: "Configura la tua scheda" (IT) / "Set up your workout plan" (EN)
- Sottotitolo: "Crea la struttura dei tuoi allenamenti settimanali." (IT) / "Create the structure of your weekly workouts." (EN)
- CTA: "Inizia" (IT) / "Get started" (EN)

Tap CTA → bottom sheet:
1. Campo "Quanti giorni di allenamento?" (IT) / "How many training days?" (EN), numero 1–7
2. Per ogni giorno: input nome, placeholder "Giorno N" (IT) / "Day N" (EN)
3. CTA "Crea scheda" (IT) / "Create plan" (EN)
4. Al salvataggio: crea gym_program + gym_program_days → redirect a /cards/gym/edit

Il rilevamento dell'area gym usa: area.type === "health" AND nome matches /gym|palestra/i (case insensitive).
```

---

## story-11-02 — Gestione piano: `/cards/gym/edit`

Continua Epic 11 di opad.me. Implementa la pagina di gestione del piano palestra.

**Route:** `/cards/gym/edit`

**Header:** titolo "Piano allenamento" (IT) / "Workout plan" (EN) + back button → `/activities`

**CTA primaria:** in cima alla pagina, bottone "Allenati oggi" (IT) / "Train today" (EN) → naviga a `/cards/gym`

**Corpo pagina — struttura del piano:**

```
Piano allenamento
  [Allenati oggi →]           ← CTA primaria in cima

  Giorno 1 — Gambe
    Addominali
      • Plank  3 × 60s
      • Crunch  3 × 20
    Gambe
      • Squat  4 × 80kg  [is_daily: no]
      • Leg press  3 × 70kg
    [+ Aggiungi gruppo muscolare]
  Giorno 2 — Push
    ...
```

Gli esercizi `is_daily = true` sono mostrati nel loro gruppo muscolare originale con un'etichetta visiva (es. badge "Giornaliero").

**CTA aggiungi gruppo muscolare:**
- Bottom sheet con campo nome gruppo
- Salvataggio silenzioso, il gruppo appare in fondo alla lista del giorno

**CTA aggiungi esercizio (dentro un gruppo):**
- Bottom sheet con: nome (obbligatorio), serie (obbligatorio), ripetizioni (obbligatorio), peso default in kg (opzionale), toggle `"Giornaliero / Daily"` con sub-label `"Appare in ogni sessione / Appears in every session"`
- Salvataggio silenzioso

**Modifica esercizio (tap su esercizio in lista):**
- Stesso bottom sheet precompilato
- CTA aggiuntiva `"Disattiva esercizio"` (IT) / `"Deactivate exercise"` (EN) — colore `#E24A4A` — imposta `active = false`, esercizio scompare dalla lista e dalle sessioni future

---

### Prompt Lovable — story-11-02

```
opad.me — Gym Card — pagina gestione piano

Crea la pagina /cards/gym/edit (GymPlanPage) per la gestione del piano di allenamento.

**Struttura pagina:**
- Header: "Piano allenamento" (IT) / "Workout plan" (EN) + back button → /activities
- CTA primaria in cima al contenuto: "Allenati oggi" (IT) / "Train today" (EN) → naviga a /cards/gym
- Se non esiste nessun gym_program → mostra il wizard setup (story-11-01), al completamento rimane su /cards/gym/edit

**Lista piano:**
Mostra la struttura gerarchica: giorni → gruppi muscolari → esercizi (solo active = true).

Per ogni giorno: lista dei gruppi muscolari con i loro esercizi.
Esercizi con is_daily = true: mostrati nel loro gruppo con un badge visivo "Giornaliero" (IT) / "Daily" (EN).

**Aggiunta gruppo muscolare:**
- CTA "+ Aggiungi gruppo muscolare" (IT) / "+ Add muscle group" (EN) in fondo a ogni giorno
- Bottom sheet: solo campo nome gruppo
- Salvataggio silenzioso

**Aggiunta esercizio:**
- CTA "+ Aggiungi esercizio" (IT) / "+ Add exercise" (EN) in fondo a ogni gruppo
- Bottom sheet: nome (obbligatorio), serie (obbligatorio), ripetizioni (obbligatorio), peso default kg (opzionale), toggle "Giornaliero / Daily" con sub "Appare in ogni sessione / Appears in every session"

**Modifica esercizio (tap su esercizio):**
- Stesso bottom sheet precompilato
- CTA "Disattiva esercizio" (IT) / "Deactivate exercise" (EN) in rosso (#E24A4A) → imposta active = false

**Aggiungi route /cards/gym/edit in App.tsx.**
**Plus gating:** stessa logica del resto della Gym Card.
```

---

## story-11-03 — Sessione giornaliera: `/cards/gym`

Continua Epic 11 di opad.me. Implementa la pagina sessione giornaliera.

**Route:** `/cards/gym`

**Header:** titolo "Scheda Palestra" (IT) / "Gym Card" (EN) + icona ⚙️ in alto a destra → naviga a `/cards/gym/edit`

**Back navigation:** back → `/activities`

**Selezione automatica del giorno:**
- Se oggi esiste già una sessione → mostrarla
- Se non esiste → selezionare automaticamente il giorno successivo all'ultima sessione completata (rotazione ciclica: dopo Giorno N torna a Giorno 1)
- Se non ci sono sessioni precedenti → mostrare Giorno 1
- Se la scheda ha un solo giorno → nessun selettore visibile

**Selettore manuale:**
- In cima alla pagina: `"Giorno N ▼"` — tap apre un picker con tutti i giorni della scheda
- La selezione di un giorno diverso sovrascrive silenziosamente la sessione del giorno precedente per la stessa data

**Layout checklist:**

La sessione mostra gli esercizi raggruppati:
1. Gruppo `"Giornalieri / Daily"` (se ci sono esercizi `is_daily`) — sempre in cima
2. Gruppi muscolari del giorno selezionato, nell'ordine definito nella scheda

Per ogni esercizio:
- Riga con: checkbox + nome + `"sets × reps · peso_default kg"` (peso non mostrato se assente)
- Esercizio completato: checkbox pieno, riga attenuata (`opacity-50`)

**Empty state (nessun esercizio attivo):**
- `"Nessun esercizio attivo per questo giorno"` (IT) / `"No active exercises for this day"` (EN)
- Sub-label: `"Aggiungi esercizi dal piano"` (IT) / `"Add exercises from your plan"` (EN)

---

### Prompt Lovable — story-11-03

```
opad.me — Gym Card — pagina sessione giornaliera

Modifica la pagina /cards/gym (GymCardPage) per essere solo la vista sessione.

**Header:**
- Titolo "Scheda Palestra" (IT) / "Gym Card" (EN)
- Icona ⚙️ in alto a destra → naviga a /cards/gym/edit
- Rimuovi qualsiasi bottone "Modifica scheda" o sezione di editing dal body della pagina

**Selezione giorno automatica:**
- Se sessione oggi già esistente → mostrarla
- Altrimenti → giorno successivo all'ultima sessione (rotazione ciclica), default Giorno 1
- Selettore "Giorno N ▼" in cima se la scheda ha più giorni; nascosto se 1 solo giorno
- Cambio giorno manuale → sovrascrive silenziosamente la sessione precedente della stessa data

**Layout checklist:**
1. Gruppo "Giornalieri" (IT) / "Daily" (EN) in cima — solo esercizi is_daily = true
2. Gruppi muscolari del giorno, in ordine
Per ogni esercizio: checkbox + nome + "sets × reps · peso kg" (peso omesso se assente)
Completato: checkbox pieno + opacity-50

**Empty state (nessun esercizio attivo):**
- "Nessun esercizio attivo per questo giorno" / "No active exercises for this day"
- Sub: "Aggiungi esercizi dal piano" / "Add exercises from your plan"

**Back navigation:** → /activities
```

---

## story-11-04 — Azioni esercizio: DONE, EDIT peso, DISATTIVA

Continua Epic 11 di opad.me. Implementa le azioni rapide sugli esercizi durante la sessione.

**DONE — tap sul checkbox:**
- Al primo tap: crea (o aggiorna) il record `gym_session_exercises` con `completed = true` e `weight_used = default_weight` dell'esercizio
- Secondo tap sullo stesso esercizio → annulla, `completed = false`
- Il **primo esercizio** marcato DONE nella sessione di oggi → completa automaticamente il check-in del giorno (se non già fatto, come in Epic 03)

**EDIT — modifica il peso per questa sessione:**
- Accesso: tap sull'area del peso (es. `"80kg"`) nella riga dell'esercizio, oppure tap su icona ✏️ accanto al peso
- Apre un bottom sheet con un solo campo: `"Peso (kg)"` precompilato con `default_weight`
- CTA `"Salva / Save"` — al salvataggio:
  1. Aggiorna `weight_used` nel record `gym_session_exercises` della sessione di oggi
  2. Aggiorna `default_weight` nell'esercizio della scheda (`gym_program_exercises`) — il nuovo peso diventa il default per le sessioni future
- CTA `"Disattiva esercizio / Deactivate exercise"` in fondo al bottom sheet (colore `#E24A4A`) — imposta `active = false`, l'esercizio scompare dalla lista corrente e dalle sessioni future
- CTA `"Annulla / Cancel"` → chiude senza modifiche

**Peso zero / assente:**
- Se `default_weight` è null o 0 → non mostrare il peso nella riga, solo `"sets × reps"`
- Il bottom sheet EDIT mostra il campo peso vuoto (non `"0"`)

---

### Prompt Lovable — story-11-04

```
opad.me — Gym Card — azioni esercizio nella sessione

Implementa le azioni DONE, EDIT e DISATTIVA sulla pagina /cards/gym.

**DONE (checkbox):**
- Tap → completed = true, weight_used = default_weight → salva gym_session_exercises
- Secondo tap → completed = false
- Primo esercizio DONE nella sessione di oggi → auto check-in del giorno (Epic 03), solo se non già fatto

**EDIT peso:**
- Trigger: tap sull'area peso nella riga (es. "80kg") oppure icona ✏️ accanto al peso
- Bottom sheet con campo "Peso (kg)" precompilato con default_weight (vuoto se default_weight è null o 0)
- CTA "Salva" / "Save": aggiorna weight_used nella sessione E default_weight in gym_program_exercises
- CTA "Disattiva esercizio" / "Deactivate exercise" (testo rosso #E24A4A): imposta active = false
- CTA "Annulla" / "Cancel": chiude senza modifiche

**Peso assente:**
- Se default_weight è null o 0: riga mostra solo "sets × reps", nessun peso
- Bottom sheet EDIT mostra campo vuoto

**Nota:** non usare long press come trigger — solo tap su area peso o icona ✏️.
```

---

## story-11-05 — Storico sessioni collassabile

Continua Epic 11 di opad.me. Aggiungi la sezione storico sessioni in fondo a `/cards/gym`.

**Posizione:** sotto la sessione di oggi, separata da un divider.

**Struttura:**
- Titolo: `"Storico sessioni"` (IT) / `"Session history"` (EN)
- Lista di sessioni precedenti (esclusa quella di oggi), ordinate dalla più recente
- Ogni riga collassata: data formattata (es. `"Lun 3 mar"`) + nome giorno (es. `"Giorno 2"`) + numero esercizi completati (es. `"6 esercizi / 6 exercises"`)
- Tap → espande la lista degli esercizi completati con il peso usato (read-only)
  - Formato riga: nome · `"sets × peso_usato kg"` oppure `"sets × reps"` se nessun peso
- Solo una sessione espandibile alla volta

**Edge case:**
- `"1 esercizio / 1 exercise"` (singolare) se la sessione contiene 1 solo esercizio completato
- Sessione con 0 esercizi completati → non mostrata nello storico
- Se non ci sono sessioni precedenti → la sezione non viene mostrata

---

### Prompt Lovable — story-11-05

```
opad.me — Gym Card — storico sessioni

Aggiungi la sezione storico sessioni in fondo alla pagina /cards/gym, dopo la checklist della sessione di oggi.

**Struttura:**
- Titolo collassabile: "Storico sessioni" (IT) / "Session history" (EN)
- Lista sessioni precedenti (esclusa oggi), dalla più recente
- Riga collassata: data formattata locale (es. "Lun 3 mar") + nome giorno + "N esercizi" / "N exercises" (singolare "1 esercizio" / "1 exercise")
- Tap → espande lista esercizi completati read-only: nome · "sets × peso kg" o "sets × reps" se nessun peso
- Una sola sessione espandibile alla volta

**Visibilità:**
- La sezione è nascosta se non ci sono sessioni precedenti
- Le sessioni con 0 esercizi completati non vengono mostrate
```

---

## story-11-06 — Entry point da Attività: bottom sheet con 2 CTA

Continua Epic 11 di opad.me. Modifica l'entry point della Gym Card nella sezione Attività.

**Comportamento attuale:** tap sulla card Gym → naviga direttamente a `/cards/gym`

**Comportamento nuovo:** tap sulla card Gym → apre un bottom sheet con due opzioni:

| CTA | Testo IT | Testo EN | Destinazione |
|---|---|---|---|
| Primaria | `Allenati oggi` | `Train today` | `/cards/gym` |
| Secondaria | `Modifica piano` | `Edit plan` | `/cards/gym/edit` |

Il bottom sheet mostra:
- Titolo: nome della card (`"Scheda Palestra"` IT / `"Gym Card"` EN)
- Descrizione breve: `"Registra la sessione di oggi o modifica il tuo piano di allenamento."` (IT) / `"Log today's session or edit your workout plan."` (EN)
- CTA primaria: `"Allenati oggi"` (filled)
- CTA secondaria: `"Modifica piano"` (outline o testo)

Questa modifica riguarda **solo la card Gym**, non le altre card (Finance, ecc.).

---

### Prompt Lovable — story-11-06

```
opad.me — Gym Card — entry point Attività con scelta sessione/gestione

Modifica l'entry point della card Gym nella sezione Attività (CardEntryPoints).

Attualmente: tap sulla card Gym → naviga a /cards/gym.
Nuovo comportamento: tap sulla card Gym → apre un bottom sheet.

**Bottom sheet:**
- Titolo: "Scheda Palestra" (IT) / "Gym Card" (EN)
- Descrizione: "Registra la sessione di oggi o modifica il tuo piano di allenamento." (IT) / "Log today's session or edit your workout plan." (EN)
- CTA primaria: "Allenati oggi" (IT) / "Train today" (EN) → /cards/gym
- CTA secondaria (outline): "Modifica piano" (IT) / "Edit plan" (EN) → /cards/gym/edit

Stile del bottom sheet: segue i pattern esistenti nell'app.
Questa modifica riguarda solo la card Gym, non le altre card.
```
