# BetonMe — Architettura Navigazione v2
### Navigation Redesign · Marzo 2026

---

## 1. Current State Assessment

### Struttura navigazione attuale

| Posizione | Voce | Tipo | Route |
|---|---|---|---|
| 1 | Home | Fissa | `/` |
| 2 | Aree | Fissa | `/areas` |
| 3 | [Slot custom 1] | Configurabile | `/finance` o `/areas/{gymId}` |
| 4 | [Slot custom 2] | Configurabile | `/finance` o `/areas/{gymId}` |
| Ultima | Impostazioni | Fissa | `/settings` |

**Componenti implementati:**
- `src/components/BottomNav.tsx` — nav mobile
- `src/components/DesktopSidebar.tsx` — nav desktop
- `src/hooks/useMenuConfig.tsx` — gestione slot custom
- `src/pages/Index.tsx` — Dashboard (grafico aggregato)
- `src/pages/Areas.tsx` — Sezione Aree (4 macro-categorie)
- `src/pages/SettingsPage.tsx` — Impostazioni

---

### Problemi strutturali rilevati

**1. Home = schermo di osservazione, non hub giornaliero**
L'attuale Dashboard (Epic 02) mostra un grafico aggregato e un selettore macro-area. È uno schermo di trend analysis, non di interazione quotidiana. L'utente che apre l'app per fare il check-in deve navigare ad Aree → Area Detail → Check-in. Tre tap prima dell'azione principale.

**2. Nessuna sezione Progress dedicata**
La funzione di osservazione dei trend vive in Home. Home funge contemporaneamente da hub giornaliero e da osservatorio storico — due ruoli incompatibili. Quando si vuole aggiungere una vera sezione Progress non c'è uno slot architetturale per essa.

**3. Custom slot = navigazione instabile**
I 2 slot configurabili in Settings creano:
- Navigazione non uniforme tra utenti (non si può ragionare su "cosa vede l'utente nella nav")
- Palestra come scorciatoia verso `/areas/{id}` — navigation shortcut a un'area specifica, non a una sezione
- Finance come sezione separata (`/finance`) sovrapposta alla categoria Finance già presente in `/areas`
- Logica condizionale nel hook `useMenuConfig` per rilevare se esiste un'area gym

**4. Finance è in due luoghi**
Finance è una macro-categoria in `/areas` (struttura) E una route separata `/finance` (proiezione). L'utente che crea un'area Finance non sa se la gestisce in Aree o in Finance nav.

**5. TrajectoryCard è dead code**
`TrajectoryCard.tsx` e `TrajectoryCardSkeleton.tsx` non sono usati dalla Dashboard attuale. Indicano una discontinuità architetturale tra la spec originale (card per area) e l'implementazione (grafico aggregato).

**6. Check-in accessibile solo da Area Detail**
Epic 03 descrive il check-in come accessibile dalla Dashboard, ma le TrajectoryCard sono dead code. In pratica oggi il check-in è accessibile solo dall'Area Detail. Questo va contro il principio "< 3 secondi".

---

### Da preservare

- Sistema bottom nav mobile / sidebar desktop (Epic 09 — completato, ben implementato)
- 4 macro-categorie (Salute, Studio, Riduci, Finanze) — tassonomia solida
- Meccanismo check-in binario (Epic 03 — spec corretta, da implementare)
- Area Detail chart + heatmap (Epic 04)
- Gym Card come estensione di Area Detail (Epic 11)
- Sistema i18n IT/EN (Epic 08 — completato)
- Score visibility toggle (Epic 07)

---

## 2. Proposed Navigation Architecture

### Struttura definitiva: 4 tab fisse

| Posizione | Tab | Route | Ruolo |
|---|---|---|---|
| 1 | **Home** | `/` | Hub giornaliero |
| 2 | **Attività** | `/activities` | Layer strutturale |
| 3 | **Progress** | `/progress` | Layer osservazione |
| 4 | **Impostazioni** | `/settings` | Preferenze e account |

Nessuno slot custom. Nessuna voce configurabile. Nav uguale per tutti gli utenti.

### Principi di coerenza nav

- Back navigation da Area Detail → torna ad Attività (non a Home)
- Back navigation da Progress > Area Trend → torna a Progress
- Impostazioni sempre ultima, separata da divider su desktop
- Nav fissa su mobile (bottom), fissa su desktop (sidebar) — nessuna variazione

### Optional 5th tab

- **Default:** non visibile
- **Attivazione:** toggle in Impostazioni ("Mostra tab aggiuntiva / Show extra tab")
- **Target MVP:** Finance projection (`/finance`) — unica opzione disponibile per ora
- **Posizione:** quarta posizione (sposta Impostazioni in quinta)
- **Architettura:** array `navItems[]` nel hook, il 5th item è `{ visible: false }` di default
- **Schema DB:** colonna `extra_tab_enabled BOOLEAN DEFAULT false` su `users` (replace `menu_custom_items`)

---

## 3. Screen Hierarchy

```
Onboarding (unauthenticated)
├── /login
├── /onboarding/areas
└── /onboarding/frequency

Home /
├── Calendar strip (7 giorni, oggi selezionato)
├── Lista attività di oggi
│   ├── Card area + CheckInButton per ogni area attiva
│   └── Indicatore gym day (se oggi = giorno scheda palestra)
└── Empty state (nessuna area → CTA vai ad Attività)

Attività /activities
├── Lista aree per macro-categoria (4 sezioni)
│   ├── Salute / Health  [+ entry point schede abilitate]
│   ├── Studio / Study
│   ├── Riduci / Reduce
│   └── Finanze / Finance  [+ entry point schede abilitate]
├── Area Detail /activities/:id
│   ├── Grafico traiettoria (compatto, 30d default)
│   ├── TimeRangeSelector
│   ├── CalendarHeatmap 30 giorni
│   ├── CheckInButton
│   └── Link "Modifica area"
├── Aggiungi Area /activities/new
└── Modifica Area /activities/:id/edit

Schede /cards/* (pagine dedicate — Epic 14)
├── Scheda Palestra /cards/gym
│   ├── Sessione di oggi (checklist esercizi)
│   ├── Selettore giorno
│   ├── Storico sessioni
│   ├── Modifica scheda
│   └── Wizard setup (primo accesso)
└── Proiezione Finanze /cards/finance
    ├── Grafico storico Finance
    └── Proiezione 30 giorni

Progress /progress
├── Grafico aggregato (tutte le aree)
├── TimeRangeSelector (30d / 90d / 365d)
└── MacroAreaSelector (Tutto · Salute · Studio · Riduci · Finanze)

Impostazioni /settings
├── Preferenze
│   ├── Lingua (IT/EN)
│   ├── Mostra punteggio traiettoria (toggle)
│   └── Notifiche (toggle)
├── Schede
│   ├── Scheda Palestra (toggle)
│   └── Proiezione Finanze (toggle)
└── Account
    ├── Email (read-only)
    ├── Sign out
    └── Delete account
```

---

## 4. Tab Responsibility Model

### Home

**Appartiene:**
- Data di oggi con navigazione al giorno precedente/successivo (futuro)
- Lista delle aree attive con stato check-in odierno
- Bottone check-in diretto per ogni area
- Indicatore gym day se applicabile

**Non appartiene:**
- Grafici storici e trend (→ Progress)
- Gestione aree e configurazione (→ Attività)
- Preferenze (→ Impostazioni)

**Intento utente:** *"Cosa devo registrare oggi? L'ho già fatto?"*

---

### Attività

**Appartiene:**
- Lista aree organizzata per macro-categoria
- CRUD aree (crea, modifica, archivia)
- Configurazione frequenza area
- Area Detail con grafico compatto + heatmap
- Gym Card e gestione scheda palestra

**Non appartiene:**
- Trend a lungo termine su più aree (→ Progress)
- Check-in stand-alone senza contesto area (check-in vive nell'Area Detail e in Home)

**Intento utente:** *"Come è strutturato il mio sistema di osservazione? Voglio gestire le mie aree."*

---

### Progress

**Appartiene:**
- Grafico aggregato di tutte le aree
- Filtro per macro-area
- Time range esteso (30d / 90d / 365d)
- Osservazione della traiettoria globale nel tempo

**Non appartiene:**
- Azioni (check-in, modifica aree)
- Finance projection (specifico di Finance, opzionale)

**Intento utente:** *"Come sta andando la mia traiettoria complessiva nel tempo?"*

---

### Impostazioni

**Appartiene:**
- Lingua
- Visibilità score
- Notifiche
- Toggle 5th tab (opzionale)
- Account (email, sign out, delete)

**Non appartiene:**
- Configurazione nav (rimosso — nav è ora fissa)
- Gestione aree (→ Attività)

**Intento utente:** *"Voglio configurare l'app secondo le mie preferenze."*

---

## 5. Schede — Moduli con Pagine Dedicate (Epic 14)

> Il concetto di 5° tab opzionale è stato **rimosso** e sostituito dal sistema Schede.

### Cosa sono le Schede

Le Schede sono moduli specialistici separati dalle attività ma collegati alle macro-sezioni. Ogni scheda ha una pagina dedicata con la propria route (`/cards/*`) e non appare nella navigazione principale.

### Schede disponibili (MVP)

| ID | Nome IT | Route | Sezione |
|---|---|---|---|
| `gym` | Scheda Palestra | `/cards/gym` | Salute |
| `finance_projection` | Proiezione Finanze | `/cards/finance` | Finanze |

### Come si accede alle Schede

1. **Da Attività:** entry point nella macro-sezione corrispondente (card con border-dashed) → bottom sheet anteprima → CTA "Apri" → pagina dedicata
2. **Da Home:** CTA "Apri scheda" per area Gym → `/cards/gym`
3. **Da Settings:** sezione "Schede" con toggle per abilitare/disabilitare

### Schema DB

```sql
-- Sostituisce extra_tab_enabled e menu_custom_items
CREATE TABLE user_cards (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  card_type TEXT NOT NULL CHECK (card_type IN ('gym', 'finance_projection')),
  area_id UUID REFERENCES areas(id) ON DELETE SET NULL,
  enabled BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(user_id, card_type)
);
```

### Navigazione invariata

La nav resta fissa a 4 tab. Le schede non aggiungono voci alla nav — vivono come pagine dedicate accessibili dall'ecosistema Attività.

```
navItems: NavItem[] = [
  { id: 'home',       route: '/',           visible: true,  fixed: true },
  { id: 'activities', route: '/activities', visible: true,  fixed: true },
  { id: 'progress',   route: '/progress',   visible: true,  fixed: true },
  { id: 'settings',   route: '/settings',   visible: true,  fixed: true, isLast: true },
]
// Nessun tab opzionale. Le route /cards/* sono registrate nel router ma non nella nav.
```

---

## 6. Primary User Flows

### Flow 1 — Open app e registra oggi (< 3 secondi)
```
1. Apri app → Home
2. Vedi lista aree di oggi con stato check-in
3. Tap su CheckIn per "Palestra" → Done
4. Il bottone passa a "Observed"
```

### Flow 2 — Apri Attività e configura un'area
```
1. Tap tab Attività
2. Vedi 4 sezioni macro-categoria
3. Tap "+ Aggiungi" sotto Salute
4. Form: nome area + frequenza
5. Torna alla lista Attività
```

### Flow 3 — Crea attività Gym con scheda
```
1. Attività → Salute → "+ Aggiungi"
2. Nome: "Palestra", tipo: health
3. Salva → suggerimento: "Vuoi configurare anche la Scheda Palestra?"
4. Tap "Configura" → naviga a /cards/gym
5. Wizard: quanti giorni? → nomi giorni → Crea scheda
6. Aggiungi gruppi muscolari ed esercizi per ogni giorno
```

### Flow 4 — Osserva traiettoria
```
1. Tap tab Progress
2. Vedi grafico aggregato 30d
3. Tap "Studio" nel filtro macro-area
4. Il grafico mostra solo le aree Studio
5. Cambia time range a 90d
```

### Flow 5 — Abilita scheda Finance
```
1. Tap tab Impostazioni
2. Nella sezione "Schede", attiva toggle "Proiezione Finanze"
3. L'entry point appare nella sezione Finanze di Attività
4. Attività → Finanze → tap entry point → bottom sheet → "Apri" → /cards/finance
```

---

## 7. Epic e Stories — Mappa degli aggiornamenti

### Epics da aggiornare

| Epic | Modifica principale |
|---|---|
| **Epic 02** (Dashboard → Home) | Rewrite completo: hub giornaliero con lista attività di oggi e check-in diretto |
| **Epic 07** (Settings) | Rimuovi sezione Menu custom; aggiungi sezione Schede generica (Epic 14) |
| **Epic 09** (Layout) | Da 3-fisse + 2-custom → 4-fisse senza tab opzionali |
| **Epic 10** (Aree → Attività) | Rename tab/route; aggiorna ruolo e back navigation |

### Nuovi epics

| Epic | Contenuto |
|---|---|
| **Epic 12** (Progress) | Sezione Progress dedicata: grafico aggregato estratto da Home + filtri |
| **Epic 14** (Schede) | Moduli specialistici con pagine dedicate — Gym Card e Finance Projection |

### Stories da modificare

| Story | Modifica |
|---|---|
| 02-01..02-05 | Rewrite: nuova Home con lista attività di oggi (non il grafico) |
| 07-04 | Rimuovi: configurazione menu custom — non più necessaria |
| 09-01..09-03 | Update: 4 tab fisse, rimosso hook useMenuConfig, nuovo useNavConfig |
| 10-01..10-03 | Update: rinomina route `/areas` → `/activities`, aggiorna label "Attività" |

### Nuove stories

| Story | Contenuto |
|---|---|
| 02-01 (nuovo) | Layout Home: header data, lista attività oggi con CheckInButton |
| 02-02 (nuovo) | Stato check-in per ciascuna area nella Home (caricamento + aggiornamento) |
| 02-03 (nuovo) | Indicatore gym day nella Home |
| 02-04 (nuovo) | Empty state Home (nessuna area) |
| 12-01 | Layout Progress: grafico aggregato + time range selector |
| 12-02 | MacroAreaSelector in Progress: filtro per macro-area |
| 12-03 | Empty state e loading state Progress |
| 09-04 (nuovo) | Refactor nav: 4 tab fisse via useNavConfig, rimozione useMenuConfig |
| 14-01..14-08 (nuove) | Sistema Schede: DB, registro, entry point, pagine dedicate, Settings, suggerimento post-creazione |

---

## 8. Implementation Notes per Codebase Lovable

### Componenti da refactorare

**`useMenuConfig.tsx` → `useNavConfig.tsx`**
- Rimuovere: logica gym detection, `menu_custom_items` array, max 2 logic
- Aggiungere: array `navItems[]` con `visible` boolean per ogni tab
- Semplificare: solo `extra_tab_enabled` da Supabase

**`BottomNav.tsx` e `DesktopSidebar.tsx`**
- Rimuovere: ogni riferimento a `menuConfig.customItems`
- Aggiungere: render da `navConfig.visibleItems` (array filtrato)
- Nessuna altra modifica strutturale necessaria

**`src/pages/Index.tsx` (Dashboard)**
- Repurpose completo: da grafico aggregato → lista aree di oggi con check-in
- Il grafico aggregato si sposta in un nuovo `src/pages/Progress.tsx`
- `MacroAreaSelector` si sposta in `Progress.tsx`

**`src/pages/Areas.tsx` → `src/pages/Activities.tsx`**
- Rename file
- Aggiorna route da `/areas` a `/activities`
- Aggiorna tab label da "Aree" a "Attività"
- Comportamento invariato

**Dead code da rimuovere:**
- `src/components/TrajectoryCard.tsx`
- `src/components/TrajectoryCardSkeleton.tsx`

### Nuovi componenti

**`src/pages/Progress.tsx`**
- Grafico aggregato (estratto da `Index.tsx`)
- `TimeRangeSelector` (già esiste, riusabile)
- `MacroAreaSelector` (già inline in `Index.tsx`, da estrarre in componente)

**`src/components/TodayActivityList.tsx`**
- Lista delle aree attive con data + CheckInButton per ciascuna
- Stato check-in caricato da Supabase (query: checkins di oggi per user_id)
- Usato nella nuova `Home`

**`src/components/GymDayIndicator.tsx`** (piccolo)
- Mostra il nome del giorno scheda previsto per oggi
- Visibile solo se esiste area gym con scheda configurata e sessione non ancora fatta

### Route changes

| Vecchio | Nuovo |
|---|---|
| `/areas` | `/activities` |
| `/areas/new` | `/activities/new` |
| `/areas/:id` | `/activities/:id` |
| `/areas/:id/edit` | `/activities/:id/edit` |
| `/` | `/` (invariato, ma comportamento completamente diverso) |
| (non esiste) | `/progress` (nuovo) |
| `/finance` (5th tab) | `/cards/finance` (pagina dedicata scheda) |
| (non esiste) | `/cards/gym` (pagina dedicata scheda) |

### Migration DB

```sql
-- Rimuovere colonne obsolete
ALTER TABLE users DROP COLUMN IF EXISTS menu_custom_items;
ALTER TABLE users DROP COLUMN IF EXISTS extra_tab_enabled;

-- Nuova tabella per il sistema Schede (Epic 14)
-- Vedi epic-14-cards.md per lo schema completo di user_cards
```

### Ordine di implementazione raccomandato

1. Refactor nav (Epic 09 aggiornato) — `useNavConfig` + 4 tab fisse — prerequisito per tutto
2. Progress section (Epic 12) — riusa quasi tutto il codice del Dashboard attuale
3. Home rewrite (Epic 02 nuovo) — lista attività + check-in
4. Settings update (Epic 07) — rimuovi menu sezione, aggiungi sezione Schede
5. Rename Attività (Epic 10) — rename route, invariato per il resto
6. Sistema Schede (Epic 14) — DB, registro, entry point, pagine dedicate

---

*BetonMe Navigation Architecture v2 · Marzo 2026*
