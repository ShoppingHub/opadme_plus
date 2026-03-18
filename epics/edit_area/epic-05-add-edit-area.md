# Epic 05 — Add / Edit Area

## Obiettivo
Permettere all'utente di creare o modificare un'area di vita in meno di 10 secondi, con un form minimale a 3 campi e un tono neutro e osservativo.

---

## Behavior

Il form ha esattamente 3 campi per le aree standard. Per le aree di tipo **Reduce**, compare una sezione opzionale per scegliere la modalità di tracciamento. La creazione non genera schermate di celebrazione — al completamento l'utente torna alla Dashboard e vede la nuova card.

La stessa schermata è usata sia per la creazione (Add) che per la modifica (Edit) — il titolo dell'header cambia di conseguenza.

---

## Flussi

### Creazione Area (tipo standard — Health, Study, Finance)
1. L'utente tappa "Add your first area" (empty state) o un pulsante "Add area"
2. Vede il form con 3 campi
3. Compila nome, tipo, frequenza
4. Tappa "Start observing"
5. L'area viene creata su Supabase
6. **Se l'area corrisponde a una scheda disponibile** (es. nome "Palestra" → scheda Gym) **e la scheda non è già abilitata** → appare un suggerimento non bloccante per configurare la scheda (Epic 14, story-14-07)
7. Altrimenti → redirect alla Dashboard — la nuova card è visibile
8. Nessuna schermata di celebrazione

### Creazione Area di tipo Reduce
1. L'utente seleziona il tipo "Reduce"
2. Compare la sezione "Tracking mode" con 2 opzioni:
   - **Binary** — osservazione semplice fatto/non fatto (default)
   - **Quantity** — conta quante volte fai questa cosa oggi
3. Se sceglie **Quantity**: compaiono 2 campi aggiuntivi:
   - `Unit label` — etichetta unità (es. "cigarettes", "coffees") — testo libero
   - `Starting quantity` — quantità di riferimento iniziale — numero intero ≥ 0
4. Tappa "Start observing"
5. L'area viene creata con `tracking_mode = quantity_reduce`
6. Redirect alla Dashboard

### Modifica Area
1. L'utente tappa "Edit area" dall'Area Detail
2. Il form si apre pre-compilato con i dati dell'area
3. L'utente modifica i campi desiderati
4. Tappa "Save changes"
5. Redirect all'Area Detail aggiornata

### Archiviazione Area (da Edit)
1. In modalità Edit, link testuale "Archive area" in fondo
2. Nessuna cancellazione hard in MVP — l'area viene archiviata (`archived_at = now()`)
3. Le aree archiviate non compaiono nella Dashboard

---

## Campi del Form

| Campo | Tipo | Placeholder / Opzioni | Default | Visibilità |
|---|---|---|---|---|
| Nome area | Text input | `"e.g. Morning walk"` | vuoto | sempre |
| Tipo | 4 pill selector | Health · Study · Reduce · Finance | nessuno | sempre |
| Frequenza / settimana | Stepper 1–7 | — | 7 | sempre |
| Tracking mode | 2 pill selector | Binary · Quantity | Binary | solo tipo Reduce |
| Unit label | Text input | `"e.g. cigarettes"` | vuoto | solo Quantity |
| Starting quantity | Number input | `"e.g. 10"` | — | solo Quantity |

> I campi **Tracking mode**, **Unit label** e **Starting quantity** compaiono solo quando il tipo selezionato è **Reduce**.
> **Unit label** e **Starting quantity** compaiono solo se il tracking mode è **Quantity**.

---

## Specifiche Pill Tipo

| Tipo | Stile |
|---|---|
| Health | `bg-[#7DA3A0]/20 text-[#7DA3A0] border border-[#7DA3A0]/40` |
| Study | `bg-[#8C9496]/20 text-[#B9C0C1] border border-[#8C9496]/40` |
| Reduce | `bg-[#BFA37A]/20 text-[#BFA37A] border border-[#BFA37A]/40` |
| Finance | `bg-[#1F4A50] text-[#EAEAEA] border border-[#7DA3A0]/30` |

Tutte le pill: `rounded-full px-3 py-1 text-[14px] font-medium border`  
Solo un tipo selezionabile alla volta.

---

## CTA

**Creazione:** `"Start observing"` — `bg-[#7DA3A0] text-[#0F2F33]`, full-width, `min-h-[44px]`  
**Modifica:** `"Save changes"` — stesso stile  
**Disabilitato:** se nome vuoto o tipo non selezionato

> ⚠️ Mai usare: "Create goal", "Set target", "Begin challenge", "Track habit"

---

## Stati UI

| Stato | Comportamento |
|---|---|
| Form vuoto | CTA disabilitata |
| Nome compilato, tipo selezionato | CTA abilitata |
| Loading (submit) | Spinner inline nel bottone, CTA disabilitata |
| Errore validazione | Messaggio inline sotto il campo, `text-[#E24A4A] text-sm` |
| Successo | Redirect — nessun messaggio |

---

## Edge Case

- Nome area con solo spazi → validazione fallisce (richiede almeno 1 carattere non-spazio)
- Tipo non selezionato al submit → errore inline "Please select a type"
- Modifica del tipo di un'area con dati storici → consentito, i dati storici rimangono
- Frequenza = 1 → ammesso, lo stepper non va sotto 1
- Frequenza = 7 → ammesso, lo stepper non va sopra 7
- Tipo Reduce + Quantity + Unit label vuoto → validazione fallisce ("Please add a unit label")
- Tipo Reduce + Quantity + Starting quantity vuoto → si ammette 0 come valore valido; vuoto blocca il submit
- Cambio tracking_mode in Edit → consentito; i dati storici `habit_quantity_daily` rimangono (non vengono cancellati)
- Cambio tipo da Reduce a non-Reduce in Edit → `tracking_mode` non viene azzerato dal DB, ma è ignorato dalla UI
- `show_quick_add_home` è visibile solo per aree `quantity_reduce` — in Edit, l'utente vede un toggle "Show quick-add on Home"

---

## Acceptance Criteria

- [ ] Il form ha esattamente 3 campi (nome, tipo, frequenza) per aree non-Reduce
- [ ] La CTA è disabilitata senza nome e tipo compilati
- [ ] Al submit, l'area è creata e la Dashboard mostra la nuova card
- [ ] Nessuna schermata di celebrazione dopo la creazione
- [ ] In modalità Edit, il form è pre-compilato
- [ ] Il tipo selezionato ha lo stile visivo corretto (pill attiva)
- [ ] Lo stepper frequenza è limitato tra 1 e 7
- [ ] Il link "Archive area" in modalità Edit archivia l'area senza cancellarla
- [ ] Selezionando tipo Reduce → compare la sezione Tracking mode (Binary · Quantity)
- [ ] Selezionando Quantity → compaiono i campi Unit label e Starting quantity
- [ ] Aree create con Quantity hanno `tracking_mode = quantity_reduce` in Supabase
- [ ] Unit label vuoto blocca il submit con errore inline
- [ ] Toggle "Show quick-add on Home" visibile solo per aree `quantity_reduce` in Edit

---

## Copy UI

| Elemento | Stringa |
|---|---|
| Header (creazione) | `"Add area"` |
| Header (modifica) | `"Edit area"` |
| Placeholder nome | `"e.g. Morning walk"` |
| Label frequenza | `"How many days per week?"` |
| CTA creazione | `"Start observing"` |
| CTA modifica | `"Save changes"` |
| Link archiviazione | `"Archive area"` |
| Label tracking mode | `"How do you want to track this?"` |
| Pill Binary | `"Simple (done / not done)"` |
| Pill Quantity | `"Count occurrences"` |
| Label unit label | `"What are you counting?"` |
| Placeholder unit label | `"e.g. cigarettes"` |
| Label starting quantity | `"Typical daily amount (your starting reference)"` |
| Placeholder starting quantity | `"e.g. 10"` |
| Toggle quick-add | `"Show quick-add on Home"` |

---

## Note UI

- Brand tokens: `brand-system/betonme_brand_system_lovable.md`
- `<AreaTypePill>` per i selettori tipo
- Stepper frequenza: frecce +/– con valore centrale, `min-h-[44px]` per ogni controllo

---

## Dipendenze

- Epic 14 (Schede) — suggerimento schede nel flusso post-creazione area

---

## Stories

- `story-05-01` — Layout form con 3 campi e CTA
- `story-05-02` — Pill selector per tipo area
- `story-05-03` — Stepper frequenza 1–7
- `story-05-04` — Integrazione Supabase: creazione e modifica area
- `story-05-05` — Archiviazione area
- `story-05-06` — Sezione Tracking mode per aree Reduce (Binary / Quantity) + campi quantitativi
- `story-05-07` — Suggerimento schede post-creazione area — **da fare** *(implementata in Epic 14, story-14-07)*
