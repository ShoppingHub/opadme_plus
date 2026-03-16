# Stories — Epic 05 — Add / Edit Area

## Sequenza di implementazione

```
story-05-01 → Form Add Area: layout con 3 campi e CTA
story-05-02 → Pill selector tipo area (Health / Study / Reduce / Finance)
story-05-03 → Stepper frequenza 1–7
story-05-04 → Integrazione Supabase: creazione area → redirect Dashboard
story-05-05 → Modalità Edit: form pre-compilato + "Save changes" + "Archive area"
story-05-06 → Tracking mode per aree Reduce: Binary / Quantity + campi quantitativi
story-05-07 → Toggle Google Tasks sync per area (solo binary + Google collegato) ⏳ da fare
story-05-08 → Toggle Gym template per aree Health                               ⏳ da fare
```

> **Dipendenze:** Richiede Epic 02 (Dashboard) completato. La nuova area creata deve apparire come card nella Dashboard.

---

## story-05-01 — Form Add Area

opad.me è un'app di osservazione del benessere. Costruisci la schermata Add Area (route `/areas/new`).

**Cosa mostra:**
- Header: `"Add area"` con back navigation verso `/`
- Form con esattamente 3 campi:
  1. **Nome area** — input testo, placeholder: `"e.g. Morning walk"`
  2. **Tipo** — pill selector (costruito nella story-05-02)
  3. **Frequenza settimanale** — stepper (costruito nella story-05-03)
- CTA full-width in fondo: `"Start observing"`

**Behavior:**
- CTA `"Start observing"` disabilitata se nome vuoto o tipo non selezionato
- La frequenza ha sempre un valore valido (default 7), quindi non blocca la CTA
- Nome con soli spazi → validazione fallisce, trattato come nome vuoto
- Tipo non selezionato al tap su CTA → errore inline: `"Please select a type"`

**Note:**
- Mai usare: `"Create goal"`, `"Set target"`, `"Begin challenge"`, `"Track habit"`
- Il form non ha campi aggiuntivi oltre i 3 specificati

---

## story-05-02 — Pill selector tipo area

Continua Epic 05 di opad.me. Costruisci il componente pill selector per il tipo area.

**4 pill selezionabili (una alla volta):**
- `"Health"` — stile teal
- `"Study"` — stile grigio
- `"Reduce"` — stile warm
- `"Finance"` — stile scuro

**Behavior:**
- Solo un tipo selezionabile alla volta
- Pill selezionata: bordo e background attivi secondo il brand system
- Pill non selezionata: stile muted
- La selezione abilita la CTA (se anche il nome è compilato)

**Note:**
- I colori di ogni pill sono definiti nel brand system (`<AreaTypePill>`) — usare esattamente quelli

---

## story-05-03 — Stepper frequenza

Continua Epic 05 di opad.me. Costruisci lo stepper per la frequenza settimanale.

**Cosa mostra:**
- Label: `"How many days per week?"`
- Stepper: bottone `–` · valore numerico centrale · bottone `+`
- Range: 1–7, default 7

**Behavior:**
- Il bottone `–` è disabilitato quando il valore è 1
- Il bottone `+` è disabilitato quando il valore è 7
- Ogni controllo ha `min-height: 44px` per accessibilità touch

---

## story-05-04 — Integrazione Supabase: creazione area

Continua Epic 05 di opad.me. Implementa il salvataggio dell'area su Supabase.

**Al tap su `"Start observing"`:**
1. CTA mostra spinner + opacity ridotta (stato loading)
2. Inserimento record in `areas`: `{ user_id, name, type, frequency_per_week }` (`archived_at` = null)
3. Redirect immediato a `/` (Dashboard)

**Cosa NON mostrare:**
- Nessuna schermata di conferma o celebrazione
- Nessun toast del tipo "Area created!"
- Nessun testo motivazionale
- La nuova card appare silenziosamente nella Dashboard

**Edge case:**
- Errore di rete → CTA torna abilitata, messaggio di errore inline non bloccante

---

## story-05-05 — Modalità Edit

Continua Epic 05 di opad.me. Aggiungi la modalità modifica area (route `/areas/:id/edit`).

**Come si accede:**
- Dal link testuale `"Edit area"` nella schermata Area Detail

**Differenze rispetto alla modalità Add:**
- Header: `"Edit area"` (invece di `"Add area"`)
- I 3 campi sono pre-compilati con i dati dell'area esistente
- CTA: `"Save changes"` (invece di `"Start observing"`)
- In fondo alla schermata: link testuale `"Archive area"` (non un bottone)

**Behavior "Save changes":**
- Aggiornamento record in `areas` con i nuovi valori
- Redirect all'Area Detail aggiornata (`/areas/:id`)
- Nessun messaggio di conferma

**Behavior "Archive area":**
- Nessun modale di conferma in MVP — azione immediata
- Aggiornamento record: `archived_at = now()`
- L'area scompare dalla Dashboard e dalla lista aree
- Redirect a `/` (Dashboard)
- L'area non viene eliminata dal database

**Note:**
- Modifica del tipo su area con dati storici: consentita, i dati esistenti rimangono

---

## story-05-06 — Tracking mode per aree Reduce ⏳

Continua Epic 05 di opad.me. Aggiungi la sezione "tracking mode" che compare solo quando il tipo area selezionato è **Reduce**.

**Quando compare:**
- Solo se l'utente ha selezionato la pill tipo `"Reduce"` (in Add o Edit)

**Cosa mostra:**
- Label: `"How do you want to track this?"`
- 2 pill selezionabili (una alla volta):
  - `"Simple (done / not done)"` → tracking_mode = `binary` (default)
  - `"Count occurrences"` → tracking_mode = `quantity_reduce`

**Se l'utente seleziona "Count occurrences" compaiono 2 campi aggiuntivi:**
1. Label: `"What are you counting?"` — input testo, placeholder: `"e.g. cigarettes"` (unit_label)
2. Label: `"Typical daily amount (your starting reference)"` — input numerico intero ≥ 0, placeholder: `"e.g. 10"` (baseline_initial)

**Behavior:**
- Default tracking mode per Reduce: Binary (non obbliga a riempire campi extra)
- Con Quantity selezionato: Unit label obbligatorio (blocca submit se vuoto — errore inline: `"Please add a unit label"`)
- Starting quantity ammette 0 come valore valido; vuoto blocca il submit
- In modalità Edit per area già Quantity: campi pre-compilati con i valori salvati
- Toggle aggiuntivo (solo Edit): `"Show quick-add on Home"` (booleano, default true) — determina se la card quick-add appare in Dashboard per questa area

**Supabase — campi da salvare in `areas`:**
- `tracking_mode`: `"binary"` | `"quantity_reduce"`
- `unit_label`: stringa (solo per quantity_reduce)
- `baseline_initial`: intero (solo per quantity_reduce)
- `show_quick_add_home`: booleano (solo per quantity_reduce, default true)

**Cosa NON fare:**
- Non mostrare questa sezione per aree Health, Study, Finance
- Non usare linguaggio di obiettivo ("Set your goal", "Target", "Limit")
- Non aggiungere altri campi non specificati

---

## story-05-07 — Toggle Google Tasks sync per area ⏳

Continua Epic 05 di opad.me. Aggiungi il toggle di sincronizzazione Google Tasks nel form area.

> **Riferimento completo:** vedi `epics/google-tasks/epic-13-google-tasks.md` (Story 13-05).

**Posizione nel form:** sotto il selettore frequenza.

**Toggle:** `"Sincronizza con Google Tasks"` (IT) / `"Sync with Google Tasks"` (EN)

**Condizioni di visibilità — il toggle appare SOLO se:**
1. L'utente ha un account Google collegato (record `google_oauth_tokens` con status `active`)
2. L'area ha `tracking_mode = 'binary'`

**Behavior:**
- Toggle ON → `areas.google_tasks_sync = true`
- Toggle OFF → `areas.google_tasks_sync = false`
- In create mode: OFF di default
- In edit mode: mostra stato corrente

### Acceptance criteria

- [ ] Toggle visibile solo con Google collegato + tracking_mode binary
- [ ] Nascosto per aree quantity_reduce
- [ ] Nascosto se Google non collegato
- [ ] Valore salvato su INSERT/UPDATE
- [ ] Testi localizzati IT/EN

---

## story-05-08 — Toggle Gym template per aree Health ⏳

Continua Epic 05 di opad.me. Aggiungi il toggle per attivare la scheda palestra nelle aree Health.

**Condizione di visibilità:** solo se tipo selezionato = `"Health"`

**Toggle:** `"Abilita scheda palestra"` (IT) / `"Enable gym plan"` (EN)

**Behavior:**
- Toggle ON → l'area viene trattata come area gym nell'Area Detail (mostra GymCard)
- La condizione di attivazione della GymCard non si basa più solo sul nome "gym/palestra" ma anche su questo flag
- In create mode: OFF di default
- In edit mode: mostra stato corrente

**Nota:** Questo toggle è una semplificazione rispetto alla detection basata sul nome. In futuro il flag esplicito sostituirà completamente l'euristica name-based.

### Acceptance criteria

- [ ] Toggle visibile solo per aree Health
- [ ] Nascosto per Study, Reduce, Finance
- [ ] GymCard appare in Area Detail quando attivo
- [ ] Testi localizzati IT/EN
