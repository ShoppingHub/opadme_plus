# Stories — Epic 11 — Gym Card (Scheda Palestra)

## Sequenza di implementazione

```
story-11-01 → Schema DB + wizard setup scheda
story-11-02 → Gestione scheda: giorni, gruppi muscolari, esercizi
story-11-03 → Sessione giornaliera: checklist e selezione giorno
story-11-04 → Azioni esercizio: DONE, EDIT peso, DISATTIVA
story-11-05 → Storico sessioni collassabile
```

> **Dipendenze:** Richiede Epic 03 (Check-in) per il completamento automatico. Epic 04 (Area Detail) per la posizione della sezione. Epic 08 (i18n) per i label.

---

## story-11-01 — Schema DB e wizard setup scheda

BetonMe è un'app di osservazione del benessere. La sezione Gym Card nell'Area Detail deve ora supportare una **scheda palestra strutturata**. Implementa il nuovo schema DB e il wizard di primo accesso.

**Nuovo schema DB** (sostituisce le tabelle `gym_sessions` e `gym_exercises` esistenti):

- `gym_programs` — una scheda per area gym
- `gym_program_days` — i giorni di allenamento della scheda (es. Giorno 1, Giorno 2)
- `gym_muscle_groups` — gruppi muscolari per giorno (es. Gambe, Pettorali)
- `gym_program_exercises` — esercizi per gruppo muscolare, con `sets`, `reps`, `default_weight` (nullable), `is_daily` (bool), `active` (bool)
- `gym_sessions` — sessione giornaliera, legata a un giorno della scheda e a una data
- `gym_session_exercises` — esercizi completati in una sessione, con `weight_used` e `completed`

Tutte le tabelle hanno RLS policy per `user_id = auth.uid()`.

**Wizard primo accesso (nessuna scheda esistente):**

Quando l'utente apre l'area Gym/Palestra e non esiste ancora una `gym_program` per quell'area, la sezione Gym Card mostra un empty state con:
- Titolo: `"Configura la tua scheda"` (IT) / `"Set up your workout plan"` (EN)
- Sottotitolo: `"Crea la struttura dei tuoi allenamenti settimanali."` (IT) / `"Create the structure of your weekly workouts."` (EN)
- CTA: `"Inizia"` (IT) / `"Get started"` (EN)

Tap su CTA → bottom sheet di configurazione:
1. Campo `"Quanti giorni di allenamento?"` (IT) / `"How many training days?"` (EN) — numero intero 1–7
2. Per ogni giorno: campo nome con placeholder `"Giorno N"` (IT) / `"Day N"` (EN) — modificabile
3. CTA `"Crea scheda"` (IT) / `"Create plan"` (EN) — crea `gym_program` + `gym_program_days`
4. Al salvataggio → sezione Gym Card mostra la vista sessione del Giorno 1

---

## story-11-02 — Gestione scheda: giorni, gruppi muscolari, esercizi

Continua Epic 11 di BetonMe. Implementa la schermata di gestione della scheda palestra.

**Accesso:** bottone `"Modifica scheda"` (IT) / `"Edit plan"` (EN) in cima alla sezione Gym Card.

**Vista gestione scheda:**

Mostra la struttura completa della scheda organizzata per giorni:

```
Scheda / Plan
  Giorno 1 — Gambe
    Addominali
      • Plank  3 × 60s
      • Crunch  3 × 20
    Gambe
      • Squat  4 × 80kg  [attivo]
      • Leg press  3 × 70kg  [attivo]
    [+ Aggiungi gruppo muscolare]
  Giorno 2 — Push
    ...
  [+ Aggiungi giorno]
```

**CTA aggiungi gruppo muscolare:**
- Bottom sheet con campo nome gruppo
- Salvataggio silenzioso, il gruppo appare in fondo alla lista del giorno

**CTA aggiungi esercizio (dentro un gruppo):**
- Bottom sheet con: nome (obbligatorio), serie (obbligatorio), ripetizioni (obbligatorio), peso default in kg (opzionale), toggle `"Giornaliero / Daily"` con sub-label `"Appare in ogni sessione / Appears in every session"`
- Salvataggio silenzioso

**Modifica esercizio (tap su esercizio in lista):**
- Stesso bottom sheet precompilato
- CTA aggiuntiva `"Disattiva esercizio"` (IT) / `"Deactivate exercise"` (EN) — colore `#E24A4A` — imposta `active = false`, esercizio scompare dalla lista e dalle sessioni future

**Esercizi con `is_daily = true`:**
- Mostrati in un gruppo speciale `"Giornalieri / Daily"` in cima alla lista, separati dagli altri
- Appaiono nella vista gestione sia nel gruppo originale sia come riferimento nel gruppo Giornalieri

---

## story-11-03 — Sessione giornaliera: checklist e selezione giorno

Continua Epic 11 di BetonMe. Implementa la visualizzazione della sessione di oggi nella Gym Card.

**Selezione automatica del giorno:**
- Se oggi esiste già una sessione → mostrarla
- Se non esiste → selezionare automaticamente il giorno successivo all'ultima sessione completata (rotazione ciclica: dopo Giorno N torna a Giorno 1)
- Se non ci sono sessioni precedenti → mostrare Giorno 1
- Se la scheda ha un solo giorno → nessun selettore visibile

**Selettore manuale:**
- In cima alla sezione: `"Giorno N ▼"` — tap apre un picker con tutti i giorni della scheda
- La selezione manuale crea una sessione per quel giorno nella data odierna
- Se oggi esiste già una sessione per un giorno diverso → sovrascriverla (nessun avviso)

**Layout checklist:**

La sessione mostra gli esercizi raggruppati:
1. Gruppo `"Giornalieri / Daily"` (se ci sono esercizi `is_daily`) — sempre in cima
2. Gruppi muscolari del giorno selezionato, nell'ordine definito nella scheda

Per ogni esercizio:
- Riga con: checkbox + nome + `"sets × reps · peso_default kg"` (peso non mostrato se assente)
- Esercizio completato: checkbox pieno, riga attenuata (`opacity-50`)
- Esercizio non completato: checkbox vuoto

**Empty state (nessun esercizio attivo):**
- `"Nessun esercizio attivo per questo giorno"` (IT) / `"No active exercises for this day"` (EN)
- Sub-label: `"Aggiungi esercizi dalla modifica scheda"` (IT) / `"Add exercises from Edit plan"` (EN)

---

## story-11-04 — Azioni esercizio: DONE, EDIT peso, DISATTIVA

Continua Epic 11 di BetonMe. Implementa le azioni rapide sugli esercizi durante la sessione.

**DONE — tap sul checkbox:**
- Al primo tap: crea (o aggiorna) il record `gym_session_exercises` con `completed = true` e `weight_used = default_weight` dell'esercizio
- Secondo tap sullo stesso esercizio → annulla, `completed = false`
- Il **primo esercizio** marcato DONE nella sessione di oggi → completa automaticamente il check-in del giorno (se non già fatto, come in Epic 03)

**EDIT — modifica il peso per questa sessione:**
- Accesso: tap sull'area del peso (es. `"80kg"`) nella riga dell'esercizio, oppure tap su un'icona ✏️ accanto al peso
- Apre un piccolo bottom sheet con un solo campo: `"Peso (kg)"` precompilato con `default_weight`
- CTA `"Salva / Save"` — al salvataggio:
  1. Aggiorna `weight_used` nel record `gym_session_exercises` della sessione di oggi
  2. Aggiorna `default_weight` nell'esercizio della scheda (`gym_program_exercises`) — il nuovo peso diventa il default per le sessioni future
- CTA `"Disattiva esercizio / Deactivate exercise"` in fondo al bottom sheet (colore `#E24A4A`) — imposta `active = false`, l'esercizio scompare dalla lista corrente e dalle sessioni future (il record `gym_session_exercises` già creato oggi rimane nello storico)
- CTA `"Annulla / Cancel"` → chiude senza modifiche

**Peso zero / assente:**
- Se `default_weight` è null o 0 → non mostrare il peso nella riga, solo `"sets × reps"`
- Il bottom sheet EDIT mostra il campo peso vuoto (non `"0"`)

---

## story-11-05 — Storico sessioni collassabile

Continua Epic 11 di BetonMe. Aggiungi la sezione storico sessioni in fondo alla Gym Card.

**Posizione:** sotto la sessione di oggi, separata da un divider.

**Struttura:**
- Titolo: `"Storico sessioni"` (IT) / `"Session history"` (EN)
- Lista di sessioni precedenti (esclusa quella di oggi), ordinate dalla più recente
- Ogni riga collassata: data formattata (es. `"Lun 3 mar"`) + nome giorno (es. `"Giorno 2"`) + numero esercizi completati (es. `"6 esercizi / 6 exercises"`)
- Tap → espande la lista degli esercizi completati con il peso usato (read-only)
  - Formato riga: nome · `"sets × peso_usato kg"` oppure `"sets × reps"` se nessun peso
- Solo una sessione espandibile alla volta

**Empty state:**
- Se non ci sono sessioni precedenti → la sezione non viene mostrata

**Edge case:**
- `"1 esercizio / 1 exercise"` (singolare) se la sessione contiene 1 solo esercizio completato
- Sessione con 0 esercizi completati → non mostrata nello storico
