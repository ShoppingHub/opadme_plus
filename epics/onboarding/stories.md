# Stories — Epic 01 — Onboarding

## Sequenza di implementazione

```
story-01-01 → Step 2: selezione aree con chip preset + custom
story-01-02 → Step 3: stepper frequenza per area
story-01-03 → Completamento onboarding → salvataggio su Supabase → redirect Dashboard
```

> **Dipendenze:** Richiede Epic 00 completato. Il magic link (Step 1) è già coperto da `story-00-02`. L'onboarding inizia dopo il redirect da Supabase Auth alla route `/onboarding`.

---

## story-01-01 — Step 2: selezione aree

BetonMe è un'app di osservazione del benessere personale. Costruisci lo Step 2 dell'onboarding (route `/onboarding/areas`).

**Cosa mostra:**
- Header: `"What do you want to observe?"`
- 4 chip suggeriti (tap per selezionare/deselezionare):
  - `"Morning movement"` (tipo: health)
  - `"Daily reading"` (tipo: study)
  - `"Less screen time"` (tipo: reduce)
  - `"Monthly saving"` (tipo: finance)
- Link testuale `"+ Add custom"` → apre un input libero per aggiungere un'area con nome personalizzato
- CTA: `"Continue"` — disabilitata finché non è selezionata almeno 1 area

**Behavior:**
- I chip sono multi-selezionabili
- Un chip selezionato mostra stile evidenziato (bordo + background attivo)
- `"+ Add custom"` aggiunge un input testuale; il nome inserito crea un'area custom di tipo da scegliere nello step successivo
- Stato con 0 aree selezionate: CTA `"Continue"` disabilitata
- Stato con ≥ 1 area selezionata: CTA `"Continue"` abilitata

**Note:**
- I chip usano lo stile `<AreaTypePill>` del brand system per il tipo corrispondente
- Non usare le label "Set your goal", "Choose your habits", "Start your journey"

---

## story-01-02 — Step 3: frequenza per area

Continua l'onboarding di BetonMe. Costruisci lo Step 3 (route `/onboarding/frequency`).

**Cosa mostra:**
- Per ogni area selezionata nello Step 2: una riga con nome dell'area e uno stepper
- Label sopra gli stepper: `"How many days per week?"`
- Stepper per ogni area: range 1–7, default 7, controlli +/– con valore centrale
- CTA: `"Start observing"`

**Behavior:**
- Lo stepper non può scendere sotto 1 né salire sopra 7
- La CTA `"Start observing"` è sempre abilitata (le frequenze hanno già un default valido)
- Le aree custom create nello Step 2 con tipo non ancora assegnato mostrano il pill selector tipo (Health / Study / Reduce / Finance) accanto allo stepper

**Note:**
- Label frequenza: sempre `"How many days per week?"` — mai `"Set your goal"` o `"Set your target"`
- I controlli +/– hanno `min-height: 44px` per accessibilità touch

---

## story-01-03 — Completamento onboarding

Continua l'onboarding di BetonMe. Implementa il salvataggio dei dati e il redirect finale.

**Behavior al tap su `"Start observing"`:**
1. CTA mostra spinner + opacity ridotta (stato loading)
2. Salvataggio su Supabase:
   - Creazione record in `areas` per ogni area selezionata (con `user_id`, `name`, `type`, `frequency_per_week`)
   - Creazione record in `users` per l'utente con preferenze default
3. Redirect immediato a `/` (Dashboard)

**Cosa NON mostrare:**
- Nessuna schermata "You're all set!"
- Nessuna schermata di benvenuto intermedia
- Nessuna animazione di congratulazioni, confetti o feedback positivo
- Nessun testo del tipo "Great!", "Let's go!", "You're ready!"

**Edge case:**
- Errore di rete durante il salvataggio → errore inline non bloccante, CTA torna abilitata
- Se l'utente riapre l'app prima di completare l'onboarding → riprende dallo step dove si era fermato
