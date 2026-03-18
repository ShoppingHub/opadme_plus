# Stories — Epic 14 — Schede (Moduli Specialistici)

## Sequenza di implementazione

```
story-14-01 → Schema DB user_cards + migrazione da extra_tab_enabled                ⏳ da fare
story-14-02 → Registro schede frontend (AVAILABLE_CARDS)                             ⏳ da fare
story-14-03 → Entry point schede nella pagina Attività (card + bottom sheet)         ⏳ da fare
story-14-04 → Pagina dedicata Scheda Palestra (/cards/gym)                           ⏳ da fare
story-14-05 → Pagina dedicata Proiezione Finanze (/cards/finance)                    ⏳ da fare
story-14-06 → Sezione Schede in Settings (toggle per ogni scheda)                    ⏳ da fare
story-14-07 → Suggerimento schede nel flusso di creazione area                       ⏳ da fare
story-14-08 → Aggiornamento CTA Home per navigazione a pagina scheda                ⏳ da fare
```

> **Dipendenze:** story-14-01 è prerequisito per tutte le altre. story-14-04 richiede Epic 11 (Gym Card). story-14-05 richiede Epic 06 (Finance). story-14-06 richiede story-14-01 + 14-02. story-14-07 richiede Epic 05. story-14-08 richiede story-14-04.

---

## story-14-01 — Schema DB user_cards + migrazione

opad.me è un'app di osservazione del benessere. Crea la tabella `user_cards` per gestire le schede (moduli specialistici) abilitate per ogni utente.

**Nuova tabella `user_cards`:**

| Colonna | Tipo | Note |
|---|---|---|
| `id` | UUID PK | Default `gen_random_uuid()` |
| `user_id` | UUID FK → `auth.users(id)` CASCADE | NOT NULL |
| `card_type` | TEXT | NOT NULL, CHECK IN ('gym', 'finance_projection') |
| `area_id` | UUID FK → `areas(id)` SET NULL | Nullable — area collegata alla scheda |
| `enabled` | BOOLEAN | DEFAULT `true` |
| `created_at` | TIMESTAMPTZ | DEFAULT `now()` |

- Constraint: `UNIQUE(user_id, card_type)`
- RLS: policy `user_id = auth.uid()` su SELECT, INSERT, UPDATE, DELETE

**Migrazione dati:**

1. Per ogni utente con `extra_tab_enabled = true` → inserisci record `user_cards` con `card_type = 'finance_projection'` e `area_id` = prima area `finance` dell'utente (se esiste)
2. Per ogni utente con un'area `health` il cui nome matcha `gym` o `palestra` (case insensitive) → inserisci record `user_cards` con `card_type = 'gym'` e `area_id` = quella area
3. Dopo migrazione: rimuovere la colonna `extra_tab_enabled` dalla tabella `users`

**Note:**
- La migrazione è idempotente (ON CONFLICT DO NOTHING)
- Se un utente non ha un'area corrispondente, `area_id` resta NULL

---

## story-14-02 — Registro schede frontend

Continua Epic 14 di opad.me. Crea la costante `AVAILABLE_CARDS` che definisce le schede disponibili nel sistema.

**Struttura:**

```typescript
const AVAILABLE_CARDS = [
  {
    id: 'gym',
    section: 'health',
    nameIT: 'Scheda Palestra',
    nameEN: 'Gym Card',
    descriptionIT: 'Gestisci la tua scheda di allenamento e registra i carichi.',
    descriptionEN: 'Manage your workout plan and log your weights.',
    icon: 'Dumbbell',
    route: '/cards/gym',
    requiresArea: true,
    areaDetection: { type: 'health', namePattern: /gym|palestra/i }
  },
  {
    id: 'finance_projection',
    section: 'finance',
    nameIT: 'Proiezione Finanze',
    nameEN: 'Finance Projection',
    descriptionIT: 'Visualizza la proiezione della tua traiettoria finanziaria.',
    descriptionEN: 'View the projection of your financial trajectory.',
    icon: 'TrendingUp',
    route: '/cards/finance',
    requiresArea: true,
    areaDetection: { type: 'finance' }
  }
]
```

**Hook `useUserCards`:**

Espone:
- `enabledCards`: lista schede abilitate per l'utente (da `user_cards` WHERE `enabled = true`)
- `toggleCard(cardType, enabled)`: abilita/disabilita una scheda
- `getCardForSection(section)`: restituisce le schede abilitate per una macro-sezione
- `isCardEnabled(cardType)`: check rapido

**Note:**
- Il registro è una costante hardcoded nel frontend — non è nel DB (il DB contiene solo lo stato utente)
- L'hook carica i dati da `user_cards` via Supabase
- Aggiornamenti ottimistici, rollback in caso di errore, nessun toast

---

## story-14-03 — Entry point schede nella pagina Attività

Continua Epic 14 di opad.me. Aggiungi gli entry point delle schede nella pagina Attività (Tab 2).

**Dove appaiono:**
- In fondo alla lista di area card di ogni macro-sezione
- Solo le schede **abilitate** (`user_cards.enabled = true`) per la sezione corrispondente
- Sotto le aree e sopra la CTA `"+ Aggiungi"` della sezione

**Card scheda (nell'elenco Attività):**

| Proprietà | Valore |
|---|---|
| Layout | Riga: icona scheda (20px `#7DA3A0`) + nome + badge stato + chevron |
| Background | `bg-[#1F4A50]/60 rounded-lg border border-[#7DA3A0]/20 border-dashed` |
| Altezza | min 48px |
| Badge configurata | `"Configurata"` (IT) / `"Set up"` (EN) — `text-[#7DA3A0] text-xs` |
| Badge da configurare | `"Da configurare"` (IT) / `"Not set up"` (EN) — `text-[#BFA37A] text-xs` |
| Tap | Apre bottom sheet anteprima |

**Logica badge:**
- `"Configurata"` se: la scheda ha un `area_id` collegato E (per gym: esiste un `gym_program` per quell'area; per finance: l'area ha almeno 1 check-in)
- `"Da configurare"` altrimenti

**Bottom sheet anteprima:**
- Icona + nome scheda (grande, centrato)
- Descrizione breve (1-2 righe, dalla definizione della scheda)
- Badge stato (configurata / da configurare)
- CTA: `"Apri"` (IT) / `"Open"` (EN) — `bg-[#7DA3A0] text-[#0F2F33]` full-width → naviga alla route dedicata della scheda
- Tap fuori o swipe down → chiude il bottom sheet

**Se la sezione non ha schede abilitate:**
- Nessun entry point mostrato — la sezione mostra solo le aree standard + CTA aggiungi

---

## story-14-04 — Pagina dedicata Scheda Palestra

Continua Epic 14 di opad.me. Crea la pagina dedicata per la Scheda Palestra a `/cards/gym`.

**Dipende da:** Epic 11 (Gym Card) per tutta la logica e UI della scheda palestra.

**Cosa contiene:**
- Header: `"Scheda Palestra"` (IT) / `"Gym Card"` (EN) + back button (→ `/activities`)
- Sotto l'header: mini-grafico traiettoria dell'area collegata (30d, compatto, altezza 120px) — opzionale, visibile solo se l'area ha dati
- Sotto il grafico: tutta l'interfaccia Gym Card attuale (Epic 11):
  - Wizard setup (se nessuna scheda configurata)
  - Sessione di oggi (checklist esercizi)
  - Selettore giorno
  - Link "Modifica scheda"
  - Storico sessioni

**Se nessuna area collegata (area_id NULL):**
- Empty state con:
  - Icona: Dumbbell, 48px, `#7DA3A0`
  - Testo IT: `"Nessuna area palestra trovata."`
  - Testo EN: `"No gym area found."`
  - Sub IT: `"Crea un'area Salute con nome 'Palestra' per iniziare."`
  - Sub EN: `"Create a Health area named 'Gym' to get started."`
  - CTA: `"Crea area"` (IT) / `"Create area"` (EN) → `/activities/new?type=health`

**Check-in automatico:**
- Il primo esercizio DONE nella sessione di oggi → auto-completa il check-in dell'area collegata (stesso comportamento Epic 11)
- Il segnale EWMA viene generato normalmente via `score_daily`

**Back navigation:** ← torna a `/activities`

---

## story-14-05 — Pagina dedicata Proiezione Finanze

Continua Epic 14 di opad.me. Crea la pagina dedicata per la Proiezione Finanze a `/cards/finance`.

**Dipende da:** Epic 06 (Finance Projection) per la logica di proiezione.

**Cosa contiene:**
- Header: `"Proiezione Finanze"` (IT) / `"Finance Projection"` (EN) + back button (→ `/activities`)
- TimeRangeSelector: 30d / 90d / 365d
- Grafico Recharts `<LineChart>` con `type="monotone"`:
  - Linea continua (storico)
  - Linea tratteggiata `strokeDasharray="4 4"` (proiezione 30 giorni)
  - Colore slope-based (mai rosso per declino — `#BFA37A`)
- Label sotto il grafico:
  - IT: `"Stima a 30 giorni basata sulla tua traiettoria attuale."`
  - EN: `"30-day estimate based on your current trend."`

**Proiezione:**
- Regressione lineare sugli ultimi 30 punti `score_daily` per l'area finance collegata
- Minimo 3 check-in per mostrare la proiezione — sotto questa soglia, solo storico

**Se nessuna area collegata (area_id NULL):**
- Empty state:
  - Icona: BarChart2, 48px, muted
  - Testo IT: `"Registra il primo check-in finanziario per vedere il tuo trend."`
  - Testo EN: `"Log your first finance check-in to see your trend."`
  - CTA: `"Crea area Finanze"` (IT) / `"Add Finance area"` (EN) → `/activities/new?type=finance`

**Se area collegata ma nessun check-in:**
- Stessa icona e testo, nessuna CTA

**Back navigation:** ← torna a `/activities`

---

## story-14-06 — Sezione Schede in Settings

Continua Epic 14 di opad.me. Sostituisci il toggle `"Mostra sezione Finanze"` nella schermata Settings con una sezione generica **Schede / Cards**.

**Dipende da:** story-14-01 (tabella `user_cards`) + story-14-02 (registro schede)

**Cosa rimuovere:**
- Il toggle `"Mostra sezione Finanze / Show Finance tab"` (story-07-07) — non più necessario
- La sotto-label `"Aggiunge un accesso rapido alla proiezione finanziaria."`
- Qualsiasi riferimento a `extra_tab_enabled` in Settings

**Cosa aggiungere — sezione "Schede / Cards":**

Posizione: dopo la sezione Preferenze, prima della sezione Account.

- Label sezione: `"Schede"` (IT) / `"Cards"` (EN)
- Sotto-label sezione: `"Moduli specialistici per le tue aree."` (IT) / `"Specialized modules for your areas."` (EN)

**Per ogni scheda in `AVAILABLE_CARDS`:**
- Riga con: icona (20px, `#7DA3A0`) + nome nella lingua corrente + toggle
- Sotto il nome: label macro-sezione in testo secondario (`text-[#B9C0C1] text-xs`)
  - Es. `"Salute"` / `"Health"` per la scheda Gym

**Behavior toggle:**
- Toggle ON → `user_cards` INSERT con `enabled = true` (o UPDATE se esiste)
- Toggle OFF → `user_cards` UPDATE `enabled = false`
- Salvataggio silenzioso, aggiornamento ottimistico
- L'entry point in Attività si aggiorna immediatamente
- In caso di errore → toggle torna al valore precedente

---

## story-14-07 — Suggerimento schede nel flusso di creazione area

Continua Epic 14 di opad.me. Aggiungi il suggerimento di schede nel flusso di creazione di una nuova area (Epic 05).

**Quando mostrare il suggerimento:**
Dopo il salvataggio dell'area (tap "Start observing"), **se**:
1. L'area appena creata corrisponde a una scheda disponibile (es. area health con nome "Palestra" → scheda Gym)
2. **E** quella scheda non è già abilitata per l'utente

**Come mostrare:**
- Un banner/card non bloccante appare sopra il redirect standard, con animazione slide-up
- Contenuto:
  - Testo IT: `"Vuoi configurare anche la [nome scheda]?"`
  - Testo EN: `"Would you also like to set up the [nome scheda]?"`
  - CTA primaria: `"Configura"` (IT) / `"Set up"` (EN) → abilita la scheda in `user_cards` con `area_id` dell'area appena creata, poi naviga a `/cards/[tipo]`
  - CTA secondaria: `"Non ora"` (IT) / `"Not now"` (EN) → chiude il banner, redirect normale ad Attività

**Se nessuna scheda corrisponde:**
- Il flusso di creazione procede normalmente senza suggerimento

**Logica di matching:**
- Per Gym: area `type = 'health'` E nome matcha `/gym|palestra/i`
- Per Finance Projection: area `type = 'finance'` (qualsiasi nome)

---

## story-14-08 — Aggiornamento CTA Home per navigazione a scheda

Continua Epic 14 di opad.me. Aggiorna la CTA dell'area Gym nella Home per navigare alla pagina dedicata della scheda.

**Dipende da:** story-14-04 (pagina dedicata Gym) + Epic 02 (Home hub)

**Cosa cambia:**
- La CTA `"Apri scheda"` (IT) / `"Open session"` (EN) per le aree Gym in Home ora naviga a `/cards/gym` anziché all'Area Detail (`/activities/:id`)
- L'indicatore gym day nella Home mantiene il tap → `/cards/gym`

**Condizione:**
- Se l'utente ha una scheda `gym` in `user_cards` con `enabled = true` → la CTA naviga a `/cards/gym`
- Se la scheda non è abilitata → la CTA resta `"Fatto"` (check-in binario standard)

**Nessun altro cambiamento alla Home.**
