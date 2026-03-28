# Stories — Epic 17 — Diet Card (Scheda Dieta)

## Sequenza di implementazione

```
story-17-01 → Schema DB + wizard setup dieta
story-17-02 → Gestione schema: /cards/diet/edit con pasti, componenti, frequenze
story-17-03 → Registrazione giornaliera: /cards/diet con checklist per pasto
story-17-04 → Conferma pasto + pasto libero + auto check-in
story-17-05 → Frequenze a livello opzione con contatore settimanale
story-17-06 → Storico sessioni collassabile
story-17-07 → Entry point da Home: 5 attività separate per pasto
story-17-08 → Gating freemium/Plus + badge
```

> **Dipendenze:** Richiede Epic 03 (Check-in) per il completamento automatico. Epic 14 (Schede) per il sistema di pagine dedicate. Epic 15 (Plus) per il gating freemium/Plus. Epic 08 (i18n) per i label.

---

## story-17-01 — Schema DB e wizard setup dieta

opad.me è un'app di osservazione del benessere. Implementa il nuovo schema DB per la Diet Card e il wizard di primo accesso.

**Nuovo schema DB:**

- `diet_programs` — uno schema dieta per area (area_id, user_id), con `mode` TEXT DEFAULT 'choice' (estendibile in futuro), `free_meals_per_week` INT DEFAULT 0
- `diet_program_meals` — i pasti dello schema, con `meal_type` (enum: breakfast, morning_snack, lunch, afternoon_snack, dinner), `active` BOOLEAN, `order` INT (1-5 fisso)
- `diet_meal_items` — componenti per pasto (es. "Proteine", "Carboidrati"), con `max_per_week` INT nullable (frequenza max settimanale), `active` BOOLEAN, `order` INT
- `diet_sessions` — sessione giornaliera per data, con `notes` TEXT nullable. UNIQUE(area_id, date)
- `diet_session_meals` — pasti registrati nella sessione, con `completed` BOOLEAN (conferma utente) e `is_free` BOOLEAN (pasto libero)
- `diet_session_items` — componenti consumati in un pasto, con `consumed` BOOLEAN

Tutte le tabelle hanno RLS policy per `user_id = auth.uid()`.

**Wizard primo accesso:**

Quando l'utente apre `/cards/diet` o `/cards/diet/edit` e non esiste ancora un `diet_program` per quell'area, la pagina mostra un empty state con:
- Titolo: `"Configura la tua dieta"` (IT) / `"Set up your diet plan"` (EN)
- Sottotitolo: `"Definisci i pasti e i componenti del tuo schema alimentare."` (IT) / `"Define the meals and components of your diet plan."` (EN)
- CTA: `"Inizia"` (IT) / `"Get started"` (EN)

Tap su CTA → bottom sheet wizard con 3 step:
1. **Step 1 — Pasti:** toggle per ciascuno dei 5 slot predefiniti (almeno 1 obbligatorio). Label: Colazione/Breakfast, Spuntino mattina/Morning snack, Pranzo/Lunch, Spuntino pomeriggio/Afternoon snack, Cena/Dinner
2. **Step 2 — Componenti:** per ogni pasto attivato, campo per aggiungere componenti (testo libero, es. "Proteine", "Carboidrati", "Verdura"). Almeno 1 componente per pasto.
3. **Step 3 — Pasti liberi:** `"Pasti liberi a settimana"` (IT) / `"Free meals per week"` (EN) — numero intero da 0 a 14, default 0
4. CTA: `"Crea schema"` (IT) / `"Create plan"` (EN) — crea `diet_program` + `diet_program_meals` + `diet_meal_items`
5. Al salvataggio → redirect a `/cards/diet/edit` per rifinire componenti e frequenze

Il rilevamento dell'area dieta usa: `area.type === "health"` AND nome matcha `/dieta|diet|alimentazione|nutrition/i` (case insensitive).

---

### Prompt Lovable — story-17-01

```
opad.me è un'app mobile-first di osservazione del benessere personale.

Implementa lo schema DB e il wizard di setup per la Diet Card.

**Schema DB (Supabase):**
Crea queste tabelle con RLS policy user_id = auth.uid():

- diet_programs: id, area_id (FK areas), user_id (FK auth.users), name TEXT, mode TEXT DEFAULT 'choice', free_meals_per_week INT DEFAULT 0, created_at
- diet_program_meals: id, program_id (FK diet_programs CASCADE), meal_type TEXT CHECK (IN 'breakfast', 'morning_snack', 'lunch', 'afternoon_snack', 'dinner'), active BOOLEAN DEFAULT true, order INT
- diet_meal_items: id, meal_id (FK diet_program_meals CASCADE), name TEXT, max_per_week INT nullable, active BOOLEAN DEFAULT true, order INT
- diet_sessions: id, area_id (FK areas CASCADE), user_id (FK auth.users CASCADE), date DATE, notes TEXT nullable. UNIQUE(area_id, date)
- diet_session_meals: id, session_id (FK diet_sessions CASCADE), program_meal_id (FK diet_program_meals), completed BOOLEAN DEFAULT false, is_free BOOLEAN DEFAULT false
- diet_session_items: id, session_meal_id (FK diet_session_meals CASCADE), meal_item_id (FK diet_meal_items), consumed BOOLEAN DEFAULT false

**Wizard primo accesso:**
Quando l'utente apre /cards/diet o /cards/diet/edit e non esiste nessun diet_program per quell'area, mostra un empty state:
- Titolo: "Configura la tua dieta" (IT) / "Set up your diet plan" (EN)
- Sottotitolo: "Definisci i pasti e i componenti del tuo schema alimentare." (IT) / "Define the meals and components of your diet plan." (EN)
- CTA: "Inizia" (IT) / "Get started" (EN)

Tap CTA → bottom sheet wizard:
1. Toggle per i 5 slot pasti: Colazione/Breakfast, Spuntino mattina/Morning snack, Pranzo/Lunch, Spuntino pomeriggio/Afternoon snack, Cena/Dinner. Almeno 1 obbligatorio.
2. Per ogni pasto attivato: campo per aggiungere componenti (testo libero). Almeno 1 per pasto.
3. "Pasti liberi a settimana" / "Free meals per week": numero 0–14, default 0
4. CTA "Crea schema" / "Create plan" → crea diet_program + meals + items → redirect a /cards/diet/edit

Il rilevamento dell'area dieta usa: area.type === "health" AND nome matches /dieta|diet|alimentazione|nutrition/i.
```

---

## story-17-02 — Gestione schema: `/cards/diet/edit`

Continua Epic 17 di opad.me. Implementa la pagina di gestione dello schema dieta.

**Route:** `/cards/diet/edit`

**Header:** titolo `"Schema dieta"` (IT) / `"Diet plan"` (EN) + back button → `/activities`

**CTA primaria:** in cima alla pagina, bottone `"Registra oggi"` (IT) / `"Log today"` (EN) → naviga a `/cards/diet`

**Corpo pagina — struttura dello schema:**

```
Schema dieta
  [Registra oggi →]           ← CTA primaria in cima

  Colazione
    • Proteine
    • Carboidrati integrali
    • Frutta                   [max 3/sett]
    [+ Aggiungi componente]

  Pranzo
    • Proteine
    • Carboidrati              [max 4/sett]
    • Verdura
    • Frutta
    [+ Aggiungi componente]

  [Toggle] Spuntino pomeriggio  [disattivato]

  Cena
    ...

  Pasti liberi: 4 / settimana  [modifica]
```

**Toggle pasti:** ogni slot pasto ha un toggle per attivare/disattivare. I pasti disattivati sono collassati con stato visivo.

**CTA aggiungi componente (dentro un pasto):**
- Bottom sheet con: nome (obbligatorio), frequenza max/settimana (opzionale, `"Max a settimana"` IT / `"Max per week"` EN)
- Salvataggio silenzioso

**Modifica componente (tap su componente in lista):**
- Stesso bottom sheet precompilato
- CTA aggiuntiva `"Disattiva componente"` (IT) / `"Deactivate component"` (EN) — colore `#E24A4A` — imposta `active = false`

**Pasti liberi:** campo numerico editabile inline, label `"Pasti liberi a settimana"` (IT) / `"Free meals per week"` (EN)

---

### Prompt Lovable — story-17-02

```
opad.me — Diet Card — pagina gestione schema

Crea la pagina /cards/diet/edit (DietPlanPage) per la gestione dello schema dieta.

**Struttura pagina:**
- Header: "Schema dieta" (IT) / "Diet plan" (EN) + back button → /activities
- CTA primaria in cima al contenuto: "Registra oggi" (IT) / "Log today" (EN) → naviga a /cards/diet
- Se non esiste nessun diet_program → mostra il wizard setup (story-17-01), al completamento rimane su /cards/diet/edit

**Lista schema:**
Mostra la struttura: pasti → componenti (solo active = true).
Ogni pasto ha un toggle per attivazione/disattivazione. Pasti disattivati mostrano stato collassato.
Componenti con max_per_week impostato: mostrare badge "max N/sett" accanto al nome.

**Aggiunta componente:**
- CTA "+ Aggiungi componente" (IT) / "+ Add component" (EN) in fondo a ogni pasto
- Bottom sheet: nome (obbligatorio), "Max a settimana" / "Max per week" (opzionale, numero intero)

**Modifica componente (tap su componente):**
- Stesso bottom sheet precompilato
- CTA "Disattiva componente" / "Deactivate component" in rosso (#E24A4A) → imposta active = false

**Pasti liberi:**
Campo numerico editabile: "Pasti liberi a settimana" (IT) / "Free meals per week" (EN), 0–14

**Aggiungi route /cards/diet/edit in App.tsx.**
**Plus gating:** stessa logica del resto della Diet Card.
```

---

## story-17-03 — Registrazione giornaliera: `/cards/diet`

Continua Epic 17 di opad.me. Implementa la pagina di registrazione pasti giornaliera.

**Route:** `/cards/diet`

**Header:** titolo `"Scheda Dieta"` (IT) / `"Diet Card"` (EN) + icona ⚙️ in alto a destra → naviga a `/cards/diet/edit`

**Back navigation:** back → `/activities`

**Layout:**
- Lista verticale dei pasti attivi previsti per oggi, nell'ordine degli slot (colazione → cena)
- Ogni pasto è una **sezione collassabile**
- Espandendo un pasto: lista dei componenti (solo `active = true`) con checkbox
- Tap su checkbox componente = toggle `consumed` (sì/no)
- Sotto i componenti: CTA `"Completa"` (IT) / `"Complete"` (EN) e CTA `"Pasto libero"` (IT) / `"Free meal"` (EN)

**Empty state (nessun componente attivo per un pasto):**
- `"Nessun componente attivo per questo pasto"` (IT) / `"No active components for this meal"` (EN)
- Sub-label: `"Aggiungi componenti dallo schema"` (IT) / `"Add components from your plan"` (EN) — link a `/cards/diet/edit`

**Empty state (nessun pasto previsto per oggi):**
- `"Nessun pasto previsto per oggi"` (IT) / `"No meals planned for today"` (EN)

La creazione della sessione (`diet_sessions`) avviene al primo componente spuntato o alla prima conferma pasto.

---

### Prompt Lovable — story-17-03

```
opad.me — Diet Card — pagina registrazione giornaliera

Crea la pagina /cards/diet (DietCardPage) per la registrazione pasti.

**Header:**
- Titolo "Scheda Dieta" (IT) / "Diet Card" (EN)
- Icona ⚙️ in alto a destra → naviga a /cards/diet/edit

**Layout:**
Lista verticale dei pasti attivi, ordine: colazione → spuntino mattina → pranzo → spuntino pomeriggio → cena.
Ogni pasto è una sezione collassabile con titolo (nome slot localizzato).
Espandendo: lista componenti con checkbox.
Tap checkbox = toggle consumed.
Sotto i componenti: CTA "Completa" / "Complete" e CTA "Pasto libero" / "Free meal".

**Empty state pasto senza componenti:**
- "Nessun componente attivo per questo pasto" / "No active components for this meal"
- Sub: "Aggiungi componenti dallo schema" / "Add components from your plan" (link a /cards/diet/edit)

**Empty state nessun pasto:**
- "Nessun pasto previsto per oggi" / "No meals planned for today"

La sessione (diet_sessions) si crea al primo componente spuntato o alla prima conferma.

**Aggiungi route /cards/diet in App.tsx.**
**Back navigation:** → /activities
```

---

## story-17-04 — Conferma pasto + pasto libero + auto check-in

Continua Epic 17 di opad.me. Implementa la logica di conferma pasto, pasti liberi e auto check-in.

**Conferma pasto:**
- CTA `"Completa"` sotto i componenti → imposta `diet_session_meals.completed = true`
- Il pasto diventa attenuato (`opacity-50`) con badge check
- Il testo CTA cambia in `"Completato"` (IT) / `"Completed"` (EN)
- Tap su pasto completato → riapre (annulla conferma, `completed = false`)

**Pasto libero:**
- CTA `"Pasto libero"` (IT) / `"Free meal"` (EN) — visibile solo se l'utente ha pasti liberi disponibili nella settimana corrente
- Imposta `diet_session_meals.is_free = true`
- Il pasto diventa attenuato con etichetta `"Libero"` (IT) / `"Free"` (EN)
- Tap su pasto libero → annulla (torna a non registrato)
- Contatore visibile in fondo alla pagina: `"Pasti liberi: N/M usati"` (IT) / `"Free meals: N/M used"` (EN)
- Il conteggio dei pasti liberi usati nella settimana ISO corrente (lunedì–domenica)

**Auto check-in:**
- Quando **tutti i pasti attivi** del giorno sono confermati (completed = true OPPURE is_free = true) → il check-in dell'area collegata viene completato automaticamente (come Epic 03)
- Se l'utente annulla una conferma → il check-in non viene rimosso (comportamento coerente con Gym)

**Pasti liberi esauriti:**
- Se `free_meals_used_this_week >= free_meals_per_week` → CTA "Pasto libero" non viene mostrata

---

### Prompt Lovable — story-17-04

```
opad.me — Diet Card — conferma pasto, pasti liberi, auto check-in

Implementa la logica di conferma sulla pagina /cards/diet.

**Conferma pasto:**
- CTA "Completa" / "Complete" → diet_session_meals.completed = true
- Pasto diventa opacity-50 con badge check
- Testo CTA → "Completato" / "Completed"
- Tap su pasto completato → annulla (completed = false)

**Pasto libero:**
- CTA "Pasto libero" / "Free meal" → diet_session_meals.is_free = true
- Pasto diventa opacity-50 con etichetta "Libero" / "Free"
- Tap → annulla (is_free = false)
- Visibile solo se free_meals_used_this_week < free_meals_per_week
- Contatore in fondo: "Pasti liberi: N/M usati" / "Free meals: N/M used"
- Settimana ISO (lunedì–domenica), reset ogni lunedì

**Auto check-in:**
- Tutti i pasti attivi del giorno completed o is_free → auto check-in area (Epic 03)
- Se annulla conferma, il check-in non viene rimosso
```

---

## story-17-05 — Frequenze a livello opzione con contatore settimanale

Continua Epic 17 di opad.me. Implementa il contatore frequenza per i componenti con `max_per_week`.

**Logica:**
- Per ogni componente con `max_per_week` impostato: contare quante volte è stato `consumed = true` nella settimana ISO corrente (lunedì–domenica), considerando tutte le sessioni della settimana
- Mostrare un badge accanto al componente nella sessione: `"N/M"` (es. `"2/3"`)

**Indicatore visivo:**
- `N < M` → badge neutro (colore testo secondario)
- `N >= M` → badge in colore `#BFA37A` (mai rosso, mai bloccante)
- L'utente **può comunque spuntare** il componente anche se ha superato il limite

**Dove appare:**
- Nella sessione giornaliera (`/cards/diet`): accanto a ogni componente con frequenza
- Non appare nella gestione schema (`/cards/diet/edit`) — lì si mostra solo `"max N/sett"` statico

---

### Prompt Lovable — story-17-05

```
opad.me — Diet Card — contatore frequenza settimanale per componente

Aggiungi il contatore frequenza nella pagina /cards/diet.

**Logica:**
Per ogni componente con max_per_week impostato: conta quante volte consumed = true nella settimana ISO corrente (lunedì–domenica), da tutte le diet_session_items della settimana.

**Badge:**
- Mostra "N/M" accanto al checkbox del componente (es. "2/3")
- Se N < M: colore testo secondario (neutro)
- Se N >= M: colore #BFA37A (mai rosso)
- L'utente può sempre spuntare il componente, anche oltre il limite — nessun blocco

**Nota:** Il badge appare solo nella sessione /cards/diet, non in /cards/diet/edit.
In /cards/diet/edit i componenti mostrano solo "max N/sett" statico.
```

---

## story-17-06 — Storico sessioni collassabile

Continua Epic 17 di opad.me. Aggiungi la sezione storico sessioni in fondo a `/cards/diet`.

**Posizione:** sotto i pasti di oggi, separata da un divider.

**Struttura:**
- Titolo: `"Storico"` (IT) / `"History"` (EN)
- Lista di sessioni precedenti (esclusa oggi), ordinate dalla più recente
- Ogni riga collassata: data formattata (es. `"Lun 3 mar"`) + numero pasti completati (es. `"4/5 pasti / 4/5 meals"`)
- Tap → espande il dettaglio: per ogni pasto, lista componenti consumati (read-only)
  - Formato: nome pasto in grassetto, sotto i componenti spuntati
  - Pasti liberi: etichetta `"Libero / Free"` invece della lista componenti
- Solo una sessione espandibile alla volta

**Edge case:**
- Sessione con 0 pasti completati → non mostrata nello storico
- Se non ci sono sessioni precedenti → la sezione non viene mostrata

---

### Prompt Lovable — story-17-06

```
opad.me — Diet Card — storico sessioni

Aggiungi la sezione storico sessioni in fondo alla pagina /cards/diet, dopo i pasti di oggi.

**Struttura:**
- Titolo collassabile: "Storico" (IT) / "History" (EN)
- Lista sessioni precedenti (esclusa oggi), dalla più recente
- Riga collassata: data formattata locale + "N/M pasti" / "N/M meals"
- Tap → espande dettaglio read-only: nome pasto in grassetto + componenti consumati sotto. Pasti liberi: etichetta "Libero" / "Free"
- Una sola sessione espandibile alla volta

**Visibilità:**
- Sezione nascosta se non ci sono sessioni precedenti
- Sessioni con 0 pasti completati non vengono mostrate
```

---

## story-17-07 — Entry point da Home: 5 attività separate per pasto

Continua Epic 17 di opad.me. Modifica la visualizzazione dell'area Dieta in Home quando la scheda Diet è attiva (Plus).

**Comportamento attuale:** l'area Dieta in Home mostra un singolo bottone "Fatto" (area binaria standard).

**Comportamento nuovo (con scheda Plus attiva):**
L'area Dieta in Home si espande in **5 sotto-attività** (una per pasto attivo), ognuna con il proprio stato:

| Stato pasto | Visualizzazione |
|---|---|
| Non ancora registrato | Nome pasto + CTA `"Registra"` (IT) / `"Log"` (EN) |
| Completato | Nome pasto + check (attenuato, `opacity-50`) |
| Pasto libero | Nome pasto + etichetta `"Libero"` (IT) / `"Free"` (EN) (attenuato) |
| Pasto non attivo nello schema | Non mostrato |

- Tap su un pasto → naviga a `/cards/diet` con scroll al pasto selezionato
- Quando tutti i pasti sono confermati, l'intera area Dieta risulta completata in Home (auto check-in, vedi story-17-04)

**Nota:** I pasti in Home non sono collegabili a Google Tasks.

---

### Prompt Lovable — story-17-07

```
opad.me — Diet Card — entry point Home con 5 attività per pasto

Modifica la Home per l'area Dieta quando la scheda diet è attiva (Plus).

**Comportamento attuale:** area Dieta → singolo bottone "Fatto".

**Nuovo comportamento (scheda attiva):**
L'area Dieta si espande in sotto-attività, una per pasto attivo nello schema.

Per ogni pasto:
- Non registrato: nome pasto localizzato + CTA "Registra" (IT) / "Log" (EN)
- Completato: nome pasto + check, opacity-50
- Pasto libero: nome pasto + etichetta "Libero" / "Free", opacity-50
- Pasto non attivo: non mostrato

Tap su pasto → /cards/diet (con scroll al pasto)

I pasti NON sono collegabili a Google Tasks (escluso per questa feature).

L'area risulta completata quando tutti i pasti attivi del giorno sono completed o is_free (auto check-in da story-17-04).
```

---

## story-17-08 — Gating freemium/Plus + badge

Continua Epic 17 di opad.me. Implementa il gating Plus per la Diet Card.

**Senza Plus:**
- In Attività: la card Diet mostra un badge `"Plus"` — tap porta a `/plus`
- In Settings (sezione Schede): toggle Diet Card disabilitato con badge `"Plus"`
- In Home: l'area Dieta resta binaria standard (bottone "Fatto")
- Le route `/cards/diet` e `/cards/diet/edit` non sono accessibili → redirect a `/plus`

**Con Plus:**
- Tutti i comportamenti descritti nelle story precedenti
- Badge "Plus" non mostrato

**Graceful degradation:**
- Se l'utente aveva Plus attivo e scade → le sessioni passate restano nel DB
- L'area Dieta torna a funzionare come binaria in Home
- Le route `/cards/diet` e `/cards/diet/edit` mostrano un messaggio con CTA per riattivare Plus

---

### Prompt Lovable — story-17-08

```
opad.me — Diet Card — gating freemium/Plus

Implementa il gating Plus per la Diet Card, coerente con il pattern di Epic 15.

**Senza Plus:**
- Attività: card Diet con badge "Plus", tap → /plus
- Settings (sezione Schede): toggle Diet disabilitato con badge "Plus"
- Home: area Dieta = bottone "Fatto" standard (binaria)
- Route /cards/diet e /cards/diet/edit → redirect a /plus

**Con Plus:**
- Tutto funziona normalmente, nessun badge

**Degradation (Plus scaduto):**
- Sessioni passate restano nel DB
- Home: area torna binaria
- Route /cards/diet e /cards/diet/edit: messaggio + CTA riattivazione Plus

Segui i pattern di gating già implementati per la Gym Card e gli altri moduli Plus.
```
