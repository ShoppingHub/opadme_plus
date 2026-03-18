# Epic 14 — Schede (Moduli Specialistici)

## Obiettivo

Introdurre le **Schede** come moduli specialistici separati dalle attività, ma collegati alle rispettive macro-sezioni del prodotto. Ogni scheda offre un'interfaccia dedicata per un tipo specifico di osservazione strutturata — senza creare sistemi paralleli per il trend.

Le Schede non sono attività: sono **strumenti di approfondimento** che vivono accanto alle aree, ne estendono le capacità, e alimentano lo stesso sistema EWMA comune.

---

## Concetto

```
Attività (Areas)                    Schede (Cards)
─────────────────                   ─────────────────
Unità di osservazione               Moduli specialistici
Check-in binario / quantitativo     Interfacce strutturate dedicate
Vivono nella lista Attività         Vivono in pagine dedicate
Segnale diretto → EWMA             Segnale derivato → EWMA (via area collegata)
```

Una scheda:
- È collegata a una **macro-sezione** (Salute, Studio, Riduci, Finanze)
- Può essere collegata a un'**area specifica** (es. Scheda Palestra → area "Palestra")
- Ha una **pagina dedicata** mobile-first per gestione e interazione
- Appare come **entry point** nella sezione Attività della sua macro-categoria
- Deriva i suoi segnali nel sistema Trend comune — nessun sistema parallelo

---

## Schede Disponibili (MVP)

| ID | Nome IT | Nome EN | Sezione | Icona Lucide | Route | Area collegata |
|---|---|---|---|---|---|---|
| `gym` | Scheda Palestra | Gym Card | Salute (`health`) | `Dumbbell` | `/cards/gym` | Sì — area con nome gym/palestra |
| `finance_projection` | Proiezione Finanze | Finance Projection | Finanze (`finance`) | `TrendingUp` | `/cards/finance` | Sì — prima area di tipo `finance` |

> Il sistema è progettato per essere estensibile. Nuove schede possono essere aggiunte in futuro (es. Piano di studio, Diario meditazione) senza modifiche architetturali.

---

## Architettura

### Registro Schede (frontend)

Le schede disponibili sono definite come costante nel frontend:

```typescript
interface CardDefinition {
  id: string
  section: AreaType          // 'health' | 'study' | 'reduce' | 'finance'
  nameIT: string
  nameEN: string
  descriptionIT: string
  descriptionEN: string
  icon: LucideIcon
  route: string
  requiresArea: boolean      // true = la scheda è collegata a un'area
  areaDetection?: {          // come trovare l'area collegata (se requiresArea)
    type: AreaType
    namePattern?: RegExp     // es. /gym|palestra/i
  }
}

const AVAILABLE_CARDS: CardDefinition[] = [
  {
    id: 'gym',
    section: 'health',
    nameIT: 'Scheda Palestra',
    nameEN: 'Gym Card',
    descriptionIT: 'Gestisci la tua scheda di allenamento e registra i carichi.',
    descriptionEN: 'Manage your workout plan and log your weights.',
    icon: Dumbbell,
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
    icon: TrendingUp,
    route: '/cards/finance',
    requiresArea: true,
    areaDetection: { type: 'finance' }
  }
]
```

### Stato utente (DB)

```
user_cards
- id UUID PK
- user_id UUID FK → auth.users(id) CASCADE
- card_type TEXT NOT NULL CHECK (card_type IN ('gym', 'finance_projection'))
- area_id UUID FK → areas(id) SET NULL (nullable — area collegata)
- enabled BOOLEAN DEFAULT true
- created_at TIMESTAMPTZ DEFAULT now()
- UNIQUE(user_id, card_type)
```

> RLS: `user_id = auth.uid()` su tutte le operazioni.

### Migrazione da architettura precedente

```sql
-- Creare tabella user_cards
CREATE TABLE user_cards (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
  card_type TEXT NOT NULL CHECK (card_type IN ('gym', 'finance_projection')),
  area_id UUID REFERENCES areas(id) ON DELETE SET NULL,
  enabled BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(user_id, card_type)
);

-- Migrazione: utenti con extra_tab_enabled = true → abilitare scheda finance_projection
INSERT INTO user_cards (user_id, card_type, area_id, enabled)
SELECT u.id, 'finance_projection', a.id, true
FROM users u
LEFT JOIN areas a ON a.user_id = u.id AND a.type = 'finance' AND a.archived_at IS NULL
WHERE u.extra_tab_enabled = true;

-- Migrazione: utenti con area gym/palestra → abilitare scheda gym
INSERT INTO user_cards (user_id, card_type, area_id, enabled)
SELECT DISTINCT u.id, 'gym', a.id, true
FROM users u
JOIN areas a ON a.user_id = u.id
  AND a.type = 'health'
  AND (LOWER(a.name) LIKE '%gym%' OR LOWER(a.name) LIKE '%palestra%')
  AND a.archived_at IS NULL;

-- Dopo migrazione: rimuovere colonna obsoleta
ALTER TABLE users DROP COLUMN extra_tab_enabled;
```

---

## Flussi

### Flusso 1 — Accesso da Attività (entry point)

1. L'utente apre la tab **Attività**
2. Nella macro-sezione rilevante (es. Salute), sotto le area card, appare un **card scheda** con stile differenziato
3. Tap sulla card scheda → si apre un **bottom sheet di anteprima**:
   - Icona + nome scheda
   - Descrizione breve (1-2 righe)
   - Stato: `"Configurata"` / `"Da configurare"` (IT) · `"Set up"` / `"Not set up"` (EN)
   - CTA: `"Apri"` / `"Open"` → naviga alla pagina dedicata
4. La gestione e modifica avviene **solo** nella pagina dedicata

### Flusso 2 — Proposta durante creazione attività

1. L'utente crea una nuova area (Add Area, Epic 05)
2. Seleziona una sezione che ha schede disponibili (es. Salute)
3. Dopo il salvataggio dell'area, appare un **suggerimento non bloccante**:
   - Testo IT: `"Vuoi configurare anche la Scheda Palestra?"`
   - Testo EN: `"Would you also like to set up the Gym Card?"`
   - CTA: `"Configura"` / `"Set up"` → naviga alla pagina dedicata della scheda
   - Dismiss: `"Non ora"` / `"Not now"` → chiude il suggerimento
4. Il suggerimento appare **solo se**:
   - La scheda non è già abilitata per l'utente
   - L'area appena creata corrisponde al pattern di rilevamento della scheda (es. nome contiene "palestra")
   - Oppure la sezione ha schede disponibili e l'utente non ne ha ancora abilitate

### Flusso 3 — Abilitazione da Settings

1. L'utente apre **Impostazioni**
2. Nella sezione **Schede**, vede la lista delle schede disponibili con toggle per ciascuna
3. Attiva un toggle → la scheda viene abilitata (`user_cards` INSERT/UPDATE)
4. La card entry point appare nella sezione Attività corrispondente
5. Disattiva un toggle → la scheda viene disabilitata, l'entry point scompare da Attività

### Flusso 4 — Pagina dedicata della scheda

1. L'utente naviga alla pagina dedicata (da Attività entry point, da Home CTA, o da URL diretto)
2. Vede l'interfaccia completa della scheda (specifica per tipo — vedi Epic 11 per Gym, Epic 06 per Finance)
3. Tutte le azioni di gestione e modifica avvengono qui
4. Back navigation → torna ad Attività (`/activities`)

---

## Entry Point in Attività

Nella pagina Attività, ogni macro-sezione mostra le schede abilitate per quella categoria come **card speciali** in fondo alla lista delle aree.

### Card Scheda (nell'elenco Attività)

| Proprietà | Valore |
|---|---|
| Layout | Riga: icona scheda + nome + indicatore stato + chevron |
| Background | `bg-[#1F4A50]/60 rounded-lg border border-[#7DA3A0]/20 border-dashed` |
| Altezza riga | min 48px |
| Icona | Lucide icon della scheda, 20px, `#7DA3A0` |
| Nome | Nome scheda nella lingua corrente |
| Indicatore stato | Badge piccolo: `"Configurata"` / `"Da configurare"` (IT) |
| Tap | Apre bottom sheet di anteprima |

> Lo stile `border-dashed` e il background ridotto distinguono visivamente le schede dalle aree standard.

### Bottom Sheet Anteprima

```
┌─────────────────────────────┐
│                             │
│  🏋️ Scheda Palestra         │  ← Icona + nome
│                             │
│  Gestisci la tua scheda di  │  ← Descrizione
│  allenamento e registra     │
│  i carichi.                 │
│                             │
│  Stato: Configurata         │  ← Badge stato
│                             │
│  [Apri]                     │  ← CTA → /cards/gym
│                             │
└─────────────────────────────┘
```

---

## Integrazione con il Sistema Trend

Le schede **non creano sistemi di trend paralleli**. Ogni scheda genera segnali che alimentano il sistema EWMA comune tramite l'area collegata.

### Gym → Trend

- La scheda Gym è collegata a un'area di tipo `health`
- Il primo esercizio marcato DONE in una sessione → auto-completa il check-in dell'area collegata
- Il check-in dell'area alimenta `score_daily` con il segnale EWMA standard
- Il grafico traiettoria dell'area (in Area Detail e Progress) riflette i dati

### Finance Projection → Trend

- La scheda Finance è collegata a un'area di tipo `finance`
- I check-in dell'area finance alimentano `score_daily` normalmente
- La proiezione è una **visualizzazione derivata** (regressione lineare sui dati `score_daily`)
- Nessun dato aggiuntivo di trend — solo una view diversa degli stessi dati

### Principio per schede future

Ogni nuova scheda deve specificare:
1. A quale area si collega (o se ne crea una nuova)
2. Come genera il segnale giornaliero (+1.0, 0.0, -0.5, -1.0) per l'area collegata
3. Nessun sistema di score separato — tutto passa per `score_daily`

---

## Pagine Dedicate

### `/cards/gym` — Scheda Palestra

Contiene tutto il contenuto precedentemente nella sezione Gym Card dell'Area Detail (Epic 11):
- Sessione di oggi (checklist esercizi)
- Selettore giorno
- Modifica scheda
- Storico sessioni
- Wizard setup (primo accesso)

In aggiunta:
- Header: `"Scheda Palestra"` (IT) / `"Gym Card"` (EN) + back button → `/activities`
- Grafico traiettoria compatto dell'area collegata (opzionale, in cima)
- Link all'Area Detail dell'area collegata

### `/cards/finance` — Proiezione Finanze

Contiene tutto il contenuto precedentemente nella schermata Finance (Epic 06):
- Grafico storico (linea continua)
- Proiezione tratteggiata (regressione lineare)
- TimeRangeSelector
- Label proiezione

In aggiunta:
- Header: `"Proiezione Finanze"` (IT) / `"Finance Projection"` (EN) + back button → `/activities`
- Empty state se nessuna area finance

---

## Sezione Schede in Settings

Sostituisce il toggle `"Mostra sezione Finanze"` con una sezione generica.

### Layout

```
┌─────────────────────────────┐
│                             │
│  Schede / Cards             │  ← Label sezione
│                             │
│  🏋️ Scheda Palestra    [○]  │  ← Toggle per ogni scheda
│     Salute                  │  ← Sotto-label: macro-sezione
│                             │
│  📈 Proiezione Finanze [○]  │
│     Finanze                 │
│                             │
└─────────────────────────────┘
```

### Behavior

- Ogni scheda disponibile ha un toggle
- Attivazione: inserisce un record `user_cards` con `enabled = true`
- Disattivazione: aggiorna `enabled = false`
- Salvataggio silenzioso
- L'entry point in Attività appare/scompare immediatamente

---

## Impatto su Home (Epic 02)

### Area Gym con scheda configurata

- CTA nella Home: `"Apri scheda"` (IT) / `"Open session"` (EN)
- Tap → naviga a `/cards/gym` (pagina dedicata della scheda) anziché all'Area Detail
- L'indicatore gym day resta invariato

### Area Finance

- Nessun cambiamento — la card finance in Home mostra solo il check-in binario standard

---

## Impatto su Navigazione (Epic 09)

- **Rimosso:** il concetto di 5° tab opzionale
- **Rimosso:** `extra_tab_enabled` dalla tabella `users`
- La nav resta fissa a **4 tab**: Home · Attività · Progress · Impostazioni
- Le schede sono accessibili tramite Attività e le proprie route dedicate
- Nessun impatto sulla logica `useNavConfig` (diventa più semplice: non serve più il 5th tab)

---

## Stati UI

| Stato | Comportamento |
|---|---|
| Scheda abilitata, configurata | Entry point in Attività con badge `"Configurata"` |
| Scheda abilitata, non configurata | Entry point con badge `"Da configurare"`, pagina mostra wizard |
| Scheda disabilitata | Non visibile in Attività |
| Nessuna scheda abilitata per una sezione | Nessun entry point, solo le aree standard |
| Area collegata archiviata | La scheda diventa non configurata, entry point mostra `"Da configurare"` |
| Area collegata rinominata (perde match) | La scheda perde il collegamento area, mostra `"Da configurare"` |

---

## Edge Case

- Utente abilita scheda Gym ma non ha un'area gym → entry point visibile con stato `"Da configurare"`, la pagina dedicata guida alla creazione
- Utente archivia area Palestra → scheda Gym perde il collegamento, mostra `"Da configurare"`
- Utente rinomina area da "Palestra" a "Running" → scheda Gym perde il rilevamento automatico, `area_id` in `user_cards` viene azzerato
- Utente crea nuova area "Palestra" → la scheda Gym rileva automaticamente la nuova area e aggiorna `area_id`
- Utente ha due aree che matchano gym → la scheda usa la prima creata
- Toggle scheda in Settings → salvataggio silenzioso, entry point si aggiorna in tempo reale

---

## Acceptance Criteria

- [ ] Le schede Gym e Finance Projection sono definite nel registro frontend
- [ ] La tabella `user_cards` gestisce lo stato di abilitazione per utente
- [ ] La colonna `extra_tab_enabled` è rimossa dalla tabella `users`
- [ ] Il 5° tab opzionale è rimosso dalla navigazione
- [ ] In Attività, le schede abilitate appaiono come entry point nella sezione corrispondente
- [ ] Tap su entry point → bottom sheet anteprima con CTA verso pagina dedicata
- [ ] La pagina dedicata Gym (`/cards/gym`) contiene tutta la funzionalità Gym Card
- [ ] La pagina dedicata Finance (`/cards/finance`) contiene tutta la funzionalità Finance Projection
- [ ] Back navigation da pagine scheda → `/activities`
- [ ] In Settings, la sezione Schede mostra toggle per ogni scheda disponibile
- [ ] Il toggle Finance in Settings è sostituito dalla sezione Schede generica
- [ ] Durante la creazione area, le schede rilevanti vengono proposte
- [ ] Le schede derivano i segnali dal sistema EWMA comune (nessun sistema parallelo)
- [ ] Il gym auto-check-in funziona dalla pagina dedicata (primo DONE → check-in area)
- [ ] Le label seguono la lingua selezionata (Epic 08)

---

## Copy UI

| Elemento | IT | EN |
|---|---|---|
| Label sezione Settings | `"Schede"` | `"Cards"` |
| Sotto-label sezione | `"Moduli specialistici per le tue aree."` | `"Specialized modules for your areas."` |
| Badge configurata | `"Configurata"` | `"Set up"` |
| Badge da configurare | `"Da configurare"` | `"Not set up"` |
| CTA bottom sheet | `"Apri"` | `"Open"` |
| Suggerimento post-creazione | `"Vuoi configurare anche la [nome scheda]?"` | `"Would you also like to set up the [card name]?"` |
| CTA suggerimento | `"Configura"` | `"Set up"` |
| Dismiss suggerimento | `"Non ora"` | `"Not now"` |
| Header pagina Gym | `"Scheda Palestra"` | `"Gym Card"` |
| Header pagina Finance | `"Proiezione Finanze"` | `"Finance Projection"` |

---

## Dipendenze

- **Epic 05** (Add/Edit Area) — per il suggerimento schede nel flusso di creazione
- **Epic 06** (Finance) — il contenuto della scheda Finance Projection
- **Epic 07** (Settings) — la sezione Schede in Impostazioni
- **Epic 08** (i18n) — tutte le label IT/EN
- **Epic 09** (Layout) — rimozione 5° tab, route dedicate
- **Epic 10** (Attività) — entry point nelle macro-sezioni
- **Epic 11** (Gym) — il contenuto della scheda Palestra

---

## Stories

- `story-14-01` — Schema DB `user_cards` + migrazione da `extra_tab_enabled`
- `story-14-02` — Registro schede frontend (costante `AVAILABLE_CARDS`)
- `story-14-03` — Entry point schede nella pagina Attività (card + bottom sheet anteprima)
- `story-14-04` — Pagina dedicata Scheda Palestra (`/cards/gym`)
- `story-14-05` — Pagina dedicata Proiezione Finanze (`/cards/finance`)
- `story-14-06` — Sezione Schede in Settings (toggle per ogni scheda)
- `story-14-07` — Suggerimento schede nel flusso di creazione area
- `story-14-08` — Aggiornamento CTA Home per navigazione a pagina scheda
