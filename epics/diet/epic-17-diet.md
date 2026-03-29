# Epic 17 — Diet Card (Scheda Dieta)

> **Nota architetturale:** Con la stessa logica della Gym Card (Epic 11), la Diet Card è un modulo specialistico (**scheda**, Epic 14) con due pagine dedicate:
> - `/cards/diet` — registrazione pasti giornaliera (checklist componenti)
> - `/cards/diet/edit` — gestione schema dieta (struttura CRUD)
>
> La sessione si avvia da Home durante la giornata, la gestione si avvia da Attività quando si vuole configurare lo schema.
>
> **Modalità corrente:** "a scelta" — l'utente definisce componenti per pasto e spunta quelli consumati. Altre modalità potranno essere aggiunte in futuro (es. piano rigido giornaliero, rotazione settimanale). L'architettura deve prevedere un campo `mode` per estensioni future.

## Obiettivo

Offrire all'utente uno strumento per **osservare l'aderenza alla propria dieta** registrando pasto per pasto cosa ha consumato rispetto allo schema definito — senza trasformare opad.me in un'app di nutrizione o contacalorie.

L'interazione principale è una **checklist per pasto**: l'utente apre il pasto, vede i componenti previsti dal suo schema, spunta quelli consumati, conferma. Fine.

---

## Versioni: Freemium vs Plus

| Versione | Comportamento |
|---|---|
| **Freemium (senza scheda)** | L'area "Dieta" sotto Salute funziona come area binaria standard. In Home: bottone "Fatto" come qualsiasi altra area. Badge "Plus" visibile per indicare che il tracking strutturato richiede l'abbonamento. |
| **Plus (con scheda attiva)** | In Home: **5 attività separate** (una per pasto attivo) con check-in individuale. Ogni pasto mostra la checklist dei componenti. Frequenze a livello opzione. Pasti liberi configurabili. |

> **Nota:** Il collegamento con Google Tasks è **escluso** per questa feature.

---

## Behavior

La Diet Card è un modulo specialistico collegato a un'area `health` con nome che matcha `/dieta|diet|alimentazione|nutrition/i`. Vive in due pagine dedicate: `/cards/diet` (registrazione) e `/cards/diet/edit` (gestione schema).

La Diet Card non sostituisce il check-in binario — quel meccanismo resta invariato per il grafico traiettoria. La Diet Card è un modulo di log strutturato che genera segnali nel sistema EWMA comune tramite l'area collegata (auto check-in quando tutti i pasti del giorno sono confermati).

---

## Modello mentale

```
schema dieta
  └── pasto (slot predefinito: colazione, spuntino mattina, pranzo, spuntino pomeriggio, cena)
       └── componente (es. "Proteine", "Carboidrati integrali", "Verdura", "Frutta")
            └── frequenza consigliata (es. max 3 volte/settimana)

sessione giornaliera
  └── data
       └── pasto (previsto per oggi)
            └── componente consumato sì/no + conferma pasto completato
```

---

## Slot pasti predefiniti

I 5 slot sono fissi e non rinominabili (per coerenza i18n):

| Slot | Chiave | IT | EN |
|---|---|---|---|
| 1 | `breakfast` | Colazione | Breakfast |
| 2 | `morning_snack` | Spuntino mattina | Morning snack |
| 3 | `lunch` | Pranzo | Lunch |
| 4 | `afternoon_snack` | Spuntino pomeriggio | Afternoon snack |
| 5 | `dinner` | Cena | Dinner |

L'utente **attiva solo gli slot che usa**. I pasti non attivati non appaiono né in Home né nella sessione.

---

## Pasti liberi

L'utente configura nella scheda quanti **pasti liberi a settimana** ha a disposizione (`free_meals_per_week`, default 0).

- Un pasto libero = nessun tracking richiesto per quel pasto
- L'utente segna un pasto come "libero" dalla sessione giornaliera
- Il contatore pasti liberi usati nella settimana corrente è visibile
- Se tutti i pasti di un giorno sono liberi → giorno **non tracciato** (nessun check-in richiesto)
- La settimana si resetta lunedì

---

## Frequenze a livello opzione

Ogni componente di un pasto può avere una **frequenza massima settimanale** (`max_per_week`, nullable).

- Se impostata: l'app mostra un indicatore visivo del consumo corrente nella settimana (es. "2/3" accanto al componente)
- Quando il limite è raggiunto: il componente appare con un'etichetta visiva neutra (mai rosso, mai bloccante) — es. badge `"3/3"` in colore `#BFA37A`
- L'utente **può comunque spuntare** il componente oltre il limite — opad.me osserva, non giudica
- Se `max_per_week` è null → nessun limite, nessun contatore
- Il conteggio si resetta lunedì (settimana ISO)

---

## Rilevamento e abilitazione

| Condizione | Comportamento |
|---|---|
| Scheda `diet` abilitata in `user_cards` + area collegata presente | Pagine `/cards/diet` e `/cards/diet/edit` accessibili |
| Scheda `diet` abilitata ma nessuna area collegata | Pagine mostrano empty state con CTA per creare area |
| Scheda `diet` non abilitata | Nessun entry point. Badge "Plus" visibile in Attività |

L'utente abilita la scheda Diet da Settings (sezione Schede) o dal suggerimento post-creazione area (Epic 14). Il rilevamento automatico dell'area collegata usa: `area.type === "health"` AND nome matcha `/dieta|diet|alimentazione|nutrition/i`.

> **Vincolo:** Non è possibile avere due aree dieta attive contemporaneamente (come per Gym).

---

## Entry point

| Da dove | Azione | Destinazione |
|---|---|---|
| Home — CTA per pasto singolo | Tap | `/cards/diet` (scroll al pasto) |
| Attività — card Diet | Tap | `/cards/diet` (registrazione) |
| Attività — card Diet (long press o secondario) | Tap | `/cards/diet/edit` (gestione) |
| Sessione `/cards/diet` — icona ⚙️ header | Tap | `/cards/diet/edit` (gestione) |
| Gestione `/cards/diet/edit` — CTA "Registra oggi" | Tap | `/cards/diet` (registrazione) |

> **Nota:** Nessun bottom sheet da Attività (da rimuovere anche per Gym in futuro).

---

## Flusso 1 — Prima apertura (nessuno schema configurato)

Quando l'utente accede a `/cards/diet` o `/cards/diet/edit` e non esiste ancora uno schema:

1. La pagina mostra un **empty state con prompt setup**:
   > `"Configura la tua dieta"` (IT) / `"Set up your diet plan"` (EN)
   > `"Definisci i pasti e i componenti del tuo schema alimentare."`
   > CTA: `"Inizia / Get started"`

2. L'utente tappa `"Inizia"` → appare un bottom sheet wizard:
   - **Step 1:** Toggle per ciascuno dei 5 slot pasto (almeno 1 obbligatorio)
   - **Step 2:** Per ogni pasto attivato, campo per aggiungere componenti (es. "Proteine", "Carboidrati", "Verdura")
   - **Step 3:** Pasti liberi a settimana (numero da 0 a 14, default 0)
   - CTA: `"Crea schema / Create plan"`

3. Alla conferma lo schema viene creato. L'utente viene portato a `/cards/diet/edit` per rifinire componenti e frequenze.

---

## Flusso 2 — Gestione schema (`/cards/diet/edit`)

**Entry point:** Attività → card Diet (secondario) oppure icona ⚙️ nell'header della sessione.

**Back navigation:** `/cards/diet/edit` → back → `/activities`

La pagina di gestione mostra:
- CTA primaria in cima: `"Registra oggi / Log today"` → naviga a `/cards/diet`
- Lista dei pasti attivi, nell'ordine degli slot (colazione → cena)
- Per ogni pasto: lista componenti con frequenza (se impostata)
- CTA per aggiungere componente a un pasto
- Toggle per attivare/disattivare pasti
- Campo pasti liberi settimanali (editabile)

### Aggiunta / modifica componente

Bottom sheet con:

| Campo | Tipo | Obbligatorio |
|---|---|---|
| Nome | Testo libero | Sì |
| Frequenza max/settimana | Numero intero (nullable) | No |

- CTA in modifica: `"Disattiva / Deactivate"` (colore `#E24A4A`) → imposta `active = false`, il componente scompare dalle sessioni future ma i dati storici restano

---

## Flusso 3 — Registrazione giornaliera (`/cards/diet`)

**Entry point:** Home → tap su un pasto, oppure Attività → card Diet.

**Back navigation:** `/cards/diet` → back → `/activities`

**Header:** titolo `"Scheda Dieta / Diet Card"` + icona ⚙️ in alto a destra → naviga a `/cards/diet/edit`

### Layout sessione

```
┌─────────────────────────────┐
│ Scheda Dieta            [⚙️] │  ← Header + icona gestione
├─────────────────────────────┤
│                             │
│  ── Colazione ──            │  ← Pasto (sezione collassabile)
│  ☐ Proteine                 │
│  ☐ Carboidrati integrali    │
│  ☑ Frutta            [2/3]  │  ← Consumato + contatore frequenza
│  ☐ Yogurt            [3/3]  │  ← Limite raggiunto (badge #BFA37A)
│  [Completato ✓]             │  ← CTA conferma pasto
│                             │
│  ── Pranzo ──               │
│  ☐ Proteine                 │
│  ☐ Carboidrati              │
│  ☐ Verdura                  │
│  ☐ Frutta                   │
│  [Completa]                 │  ← CTA conferma pasto
│                             │
│  ── Spuntino pomeriggio ──  │
│  [Pasto libero 🏷️]          │  ← Segnato come libero
│                             │
│  Pasti liberi: 2/4 usati    │  ← Contatore settimanale
│                             │
│  [Storico ▼]                │
│                             │
└─────────────────────────────┘
```

### Interazione per pasto

1. L'utente espande un pasto → vede i componenti previsti
2. Spunta i componenti effettivamente consumati (tap = toggle)
3. Clicca **"Completa"** per confermare il pasto → il pasto diventa attenuato (`opacity-50`) con check
4. In alternativa, può segnare il pasto come **"Pasto libero"** (se ha pasti liberi disponibili)
5. Un pasto confermato può essere riaperto con un tap (annulla conferma)

### Auto check-in

Quando **tutti i pasti previsti per oggi** sono confermati (completati o liberi) → il check-in dell'area viene completato automaticamente (se non già fatto). Questo spunta automaticamente l'attività in Home.

### Pasti in Home

Con la scheda Plus attiva, in Home l'area Dieta mostra **5 attività separate** (una per pasto attivo):

| Stato pasto | Visualizzazione in Home |
|---|---|
| Non ancora registrato | Nome pasto + CTA "Registra" |
| Completato | Nome pasto + check (attenuato) |
| Pasto libero | Nome pasto + etichetta "Libero" (attenuato) |
| Pasto non previsto oggi | Non mostrato |

Tap su un pasto in Home → apre `/cards/diet` con scroll al pasto selezionato.

---

## Flusso 4 — Storico sessioni

Sotto la sessione di oggi, una sezione collassabile `"Storico / History"`.

- Lista giorni precedenti ordinati dal più recente
- Ogni riga collassata: data + numero pasti completati (es. `"Lun 3 Mar · 4/5 pasti"`)
- Tap → espande il dettaglio: per ogni pasto, lista componenti consumati (read-only)
- Solo una sessione espansa alla volta
- Se non ci sono sessioni precedenti → sezione nascosta
- Sessioni con 0 pasti completati → non mostrate

---

## Database

```
diet_programs
- id UUID PK
- area_id UUID FK → areas(id) CASCADE
- user_id UUID FK → auth.users(id) CASCADE
- name TEXT
- mode TEXT DEFAULT 'choice'    -- 'choice' per MVP, estendibile in futuro
- free_meals_per_week INT DEFAULT 0
- created_at TIMESTAMPTZ

diet_program_meals
- id UUID PK
- program_id UUID FK → diet_programs(id) CASCADE
- meal_type TEXT CHECK (meal_type IN ('breakfast', 'morning_snack', 'lunch', 'afternoon_snack', 'dinner'))
- active BOOLEAN DEFAULT true
- order INT                     -- 1-5, ordine fisso degli slot

diet_meal_items
- id UUID PK
- meal_id UUID FK → diet_program_meals(id) CASCADE
- name TEXT
- max_per_week INT nullable     -- frequenza max settimanale (null = nessun limite)
- active BOOLEAN DEFAULT true
- order INT

diet_sessions
- id UUID PK
- area_id UUID FK → areas(id) CASCADE
- user_id UUID FK → auth.users(id) CASCADE
- date DATE
- notes TEXT nullable           -- note giornaliere
- UNIQUE(area_id, date)

diet_session_meals
- id UUID PK
- session_id UUID FK → diet_sessions(id) CASCADE
- program_meal_id UUID FK → diet_program_meals(id)
- completed BOOLEAN DEFAULT false   -- utente ha cliccato "Completa"
- is_free BOOLEAN DEFAULT false     -- segnato come pasto libero

diet_session_items
- id UUID PK
- session_meal_id UUID FK → diet_session_meals(id) CASCADE
- meal_item_id UUID FK → diet_meal_items(id)
- consumed BOOLEAN DEFAULT false
```

> RLS: ogni tabella ha policy `user_id = auth.uid()` su SELECT, INSERT, UPDATE, DELETE.

---

## Stati UI

| Stato | Pagina | Comportamento |
|---|---|---|
| Nessuno schema → empty state setup | `/cards/diet` o `/cards/diet/edit` | Prompt con CTA "Inizia" → wizard |
| Wizard in corso | Bottom sheet | Step 1-3 → CTA "Crea schema" |
| Sessione — nessun componente attivo | `/cards/diet` | Messaggio + sub-label "Aggiungi componenti dallo schema →" |
| Pasto non ancora registrato | `/cards/diet` | Sezione espandibile con checklist componenti |
| Pasto completato | `/cards/diet` | Sezione attenuata (`opacity-50`), badge check |
| Pasto libero | `/cards/diet` | Sezione attenuata, etichetta "Libero" |
| Tutti i pasti completati | `/cards/diet` | Nessun feedback speciale — l'utente chiude |
| Limite frequenza raggiunto | `/cards/diet` | Badge contatore in `#BFA37A`, non bloccante |
| Bottom sheet modifica aperto | `/cards/diet/edit` | Overlay con campi componente |
| Storico espanso | `/cards/diet` | Lista pasti read-only |
| Vista gestione | `/cards/diet/edit` | Lista pasti → componenti con CTA CRUD |
| Freemium (senza scheda) | Home | Area binaria standard + badge "Plus" |

---

## Edge Case

- **Utente rinomina area:** l'area è sempre sotto Salute; se il nome non matcha più `/dieta|diet|alimentazione|nutrition/i` → Diet Card si scollega, dati restano nel DB. Al ripristino del nome → riappare con schema e storico intatti.
- **Utente archivia area collegata:** la card si disattiva. Le sessioni passate restano nel DB. Riattivazione a scelta dell'utente.
- **Due aree "dieta":** non permesso — una sola area dieta attiva alla volta.
- **Tutti i componenti di un pasto disattivati:** pasto mostra messaggio "Nessun componente attivo" + link a gestione schema.
- **Pasti liberi esauriti:** il CTA "Pasto libero" scompare. L'utente deve registrare normalmente.
- **Frequenza superata:** solo indicatore visivo neutro (`#BFA37A`), nessun blocco.
- **Schema modificato:** sessioni passate sono snapshot immutabili. Le modifiche allo schema si applicano solo alle sessioni future.
- **Check-in retroattivo:** l'utente può registrare pasti di giorni passati dalla navigazione temporale in Home (week selector).
- **Nessun pasto attivo per oggi:** stato vuoto "Nessun pasto previsto per oggi" + link a gestione.

---

## Acceptance Criteria

- [ ] La Diet Card appare solo per `area.type === "health"` con nome che matcha `/dieta|diet|alimentazione|nutrition/i`
- [ ] Versione freemium: area binaria standard con badge "Plus" per il tracking strutturato
- [ ] Versione Plus: 5 attività separate in Home (una per pasto attivo)
- [ ] Prima apertura senza schema → empty state con prompt setup (sia su `/cards/diet` che su `/cards/diet/edit`)
- [ ] Wizard crea schema: toggle pasti → componenti → pasti liberi → redirect a `/cards/diet/edit`
- [ ] Per ogni pasto è possibile aggiungere/modificare/disattivare componenti da `/cards/diet/edit`
- [ ] Ogni componente ha campo opzionale `max_per_week` con contatore visivo in sessione
- [ ] Limite raggiunto → badge `#BFA37A`, mai bloccante
- [ ] Tap checkbox componente → toggle consumato
- [ ] CTA "Completa" per pasto → conferma, pasto attenuato
- [ ] CTA "Pasto libero" (se disponibili) → segna come libero, nessun tracking
- [ ] Contatore pasti liberi settimanali visibile (reset lunedì)
- [ ] Tutti i pasti confermati → auto check-in area (se non già fatto)
- [ ] Storico sessioni collassabile, read-only, ordinato per data decrescente
- [ ] Icona ⚙️ in header di `/cards/diet` → naviga a `/cards/diet/edit`
- [ ] CTA "Registra oggi" in `/cards/diet/edit` → naviga a `/cards/diet`
- [ ] Nessun bottom sheet da Attività — navigazione diretta
- [ ] Una sola dieta attiva per utente
- [ ] Sessioni passate immutabili (snapshot)
- [ ] Campo `mode` nel DB per estensioni future
- [ ] Google Tasks sync esclusa per questa feature
- [ ] Campo note a livello sessione (giornaliero)
- [ ] Tutte le label seguono la lingua selezionata (Epic 08)

---

## Copy UI

| Elemento | IT | EN |
|---|---|---|
| Titolo sezione sessione | `Scheda Dieta` | `Diet Card` |
| Titolo pagina gestione | `Schema dieta` | `Diet plan` |
| Sottotitolo sessione | `Pasti di oggi` | `Today's meals` |
| CTA sessione da gestione | `Registra oggi` | `Log today` |
| Empty state titolo | `Configura la tua dieta` | `Set up your diet plan` |
| Empty state sub | `Definisci i pasti e i componenti del tuo schema alimentare.` | `Define the meals and components of your diet plan.` |
| CTA setup | `Inizia` | `Get started` |
| CTA crea schema | `Crea schema` | `Create plan` |
| CTA completa pasto | `Completa` | `Complete` |
| CTA pasto completato | `Completato` | `Completed` |
| CTA pasto libero | `Pasto libero` | `Free meal` |
| Contatore pasti liberi | `Pasti liberi: N/M usati` | `Free meals: N/M used` |
| Contatore frequenza | `N/M` | `N/M` |
| CTA aggiungi componente | `+ Aggiungi componente` | `+ Add component` |
| Label frequenza max | `Max a settimana` | `Max per week` |
| Azione disattiva | `Disattiva componente` | `Deactivate component` |
| Salva | `Salva` | `Save` |
| Annulla | `Annulla` | `Cancel` |
| Storico | `Storico` | `History` |
| Nessun componente attivo | `Nessun componente attivo per questo pasto` | `No active components for this meal` |
| Sub nessun componente | `Aggiungi componenti dallo schema` | `Add components from your plan` |
| Badge Plus | `Plus` | `Plus` |
| Nessun pasto previsto | `Nessun pasto previsto per oggi` | `No meals planned for today` |
| Slot Colazione | `Colazione` | `Breakfast` |
| Slot Spuntino mattina | `Spuntino mattina` | `Morning snack` |
| Slot Pranzo | `Pranzo` | `Lunch` |
| Slot Spuntino pomeriggio | `Spuntino pomeriggio` | `Afternoon snack` |
| Slot Cena | `Cena` | `Dinner` |

---

## Note UI

- L'interazione principale è checklist-like: veloce, pochi tap per pasto
- Nessun conteggio calorie, nessuna statistica nutrizionale — solo osservazione dell'aderenza
- Il colore `#BFA37A` per i limiti frequenza è coerente con il brand system (trend neutro/negativo, mai rosso)
- `#E24A4A` solo per azioni distruttive (disattiva componente)
- L'icona ⚙️ nell'header è un accesso secondario, non prominente
- Le note sono a livello sessione (giornaliero), non per singolo pasto

---

## Dipendenze

- Epic 03 (Check-in) — il check-in si completa automaticamente quando tutti i pasti sono confermati
- Epic 14 (Schede) — la Diet Card ha due pagine dedicate: `/cards/diet` e `/cards/diet/edit`
- Epic 15 (Plus) — la scheda è gated da Plus; versione freemium = area binaria
- Epic 08 (i18n) — tutte le label in IT/EN

---

## Stories

- `story-17-01` — Schema DB + wizard setup dieta
- `story-17-02` — Gestione schema: `/cards/diet/edit` con pasti, componenti, frequenze
- `story-17-03` — Registrazione giornaliera: `/cards/diet` con checklist per pasto
- `story-17-04` — Conferma pasto + pasto libero + auto check-in
- `story-17-05` — Frequenze a livello opzione con contatore settimanale
- `story-17-06` — Storico sessioni collassabile
- `story-17-07` — Entry point da Home: 5 attività separate per pasto
- `story-17-08` — Gating freemium/Plus + badge
