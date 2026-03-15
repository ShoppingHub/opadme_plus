# Epic 01 — Onboarding

## Obiettivo
Portare un nuovo utente da "non autenticato" a "dashboard con almeno un'area attiva" in meno di 2 minuti, senza friction e senza tono motivazionale.

---

## Behavior

L'onboarding è composto da 4 step sequenziali. Non ci sono skip button nei primi 3 step. Non ci sono tutorial overlay. Al completamento, l'utente arriva direttamente alla Dashboard — senza schermate di celebrazione.

---

## Flussi

### Step 1 — Account (Magic Link)
1. L'utente inserisce la propria email
2. Tappa "Send me a link"
3. Schermata statica di conferma: `"Check your email. We sent you a magic link."`
4. L'utente clicca il link ricevuto via email
5. Redirect automatico alla Step 2

**Nessuna autenticazione social in MVP.**

### Step 2 — Cosa vuoi osservare?
1. Header: `"What do you want to observe?"`
2. 4 chip di suggerimento (tap per aggiungere):
   - 🚶 Morning movement
   - 📖 Daily reading
   - 📵 Less screen time
   - 💰 Monthly saving
3. Link `+ Add custom` per input libero
4. CTA: `"Continue"` — disabilitato fino a ≥ 1 area selezionata

### Step 3 — Con quale frequenza?
1. Per ogni area selezionata: stepper 1–7 giorni/settimana
2. Label: `"How many days per week?"` — mai "Set your goal"
3. Default: 7
4. CTA: `"Start observing"`

### Step 4 — Dashboard
- Redirect immediato alla Dashboard
- Nessuna schermata "You're all set!"
- Nessuna animazione di congratulazioni
- Nessun confetti

---

## Stati UI

| Stato | Comportamento |
|---|---|
| Email input vuota | CTA disabilitata |
| Email inviata | Schermata statica di conferma, nessun loader |
| Step 2 — nessuna area selezionata | CTA "Continue" disabilitata |
| Step 2 — area selezionata | Chip evidenziato, CTA abilitata |
| Step 3 — frequenza impostata | Stepper visivo per area |
| Completamento | Redirect diretto alla Dashboard |

---

## Edge Case

- Utente inserisce email già registrata → riceve comunque il link (stesso flusso)
- Utente non clicca il link → non entra nell'app, nessun messaggio aggiuntivo
- Utente riapre il link scaduto → schermata di errore con nuovo campo email
- Utente aggiunge un'area custom con nome duplicato → consentito (distinzione per id)
- Connessione assente durante onboarding → errore inline non bloccante sul submit

---

## Acceptance Criteria

- [ ] L'utente può completare l'intero onboarding in < 2 minuti
- [ ] La CTA "Continue" al Step 2 è disabilitata senza selezione
- [ ] Al completamento del Step 3 l'utente atterra sulla Dashboard senza schermate intermedie
- [ ] Non compare nessun messaggio di tipo "Great job!", "You're all set!", "Let's go!"
- [ ] Le aree create in onboarding sono visibili come card nella Dashboard
- [ ] Il magic link funziona e autentica correttamente l'utente

---

## Copy UI

| Elemento | Stringa |
|---|---|
| Email input placeholder | `"Your email address"` |
| CTA Step 1 | `"Send me a link"` |
| Sotto l'input | `"We'll send you a magic link. No password needed."` |
| Conferma email inviata | `"Check your email. We sent you a magic link."` |
| Header Step 2 | `"What do you want to observe?"` |
| CTA Step 2 | `"Continue"` |
| Label frequenza Step 3 | `"How many days per week?"` |
| CTA Step 3 | `"Start observing"` |

---

## Note UI

- Colori e componenti: `brand-system/betonme_brand_system_lovable.md`
- I chip di suggerimento al Step 2 usano lo stesso stile di `<AreaTypePill>` per il tipo corrispondente
- Mai usare: "Create goal", "Set target", "Begin challenge", "Track habit"

---

## Stories

- `story-01-01` — Setup schermata magic link
- `story-01-02` — Step 2: selezione aree con chip
- `story-01-03` — Step 3: stepper frequenza per area
- `story-01-04` — Redirect finale alla Dashboard
