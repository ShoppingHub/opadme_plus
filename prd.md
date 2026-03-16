# opad.me — Product Requirements Document
### MVP · Personal Wellbeing Trajectory App · v2.0
### Target: **Lovable** · March 2026 · `CONFIDENTIAL`

---

## ⚡ TL;DR

opad.me è una **mobile-first React web app** di osservazione del benessere personale.
Gli utenti registrano un'azione giornaliera per area di vita. Il grafico è il prodotto.
Tono: calmo, neutro, riflessivo. Un journal personale, non una fitness app.

---

## 1. Prodotto

opad.me rende **visibile la traiettoria del benessere personale nel tempo** tramite grafici longitudinali.

Non è un habit tracker. Non è un goal-setting tool. È un **sistema di osservazione** — uno specchio neutro dei pattern comportamentali.

---

## 2. Target User

- **Età:** 25–55
- **Profilo:** Interessati alla crescita personale, già sperimentano abitudini di benessere
- **Pain:** Esauriti dalla cultura della produttività e dalla pressione delle performance
- **JTBD:** *"Voglio osservare la direzione delle mie azioni nel tempo, senza sentirmi valutato."*

---

## 3. Tech Stack

| Layer | Tecnologia |
|---|---|
| Framework | React 18 + TypeScript |
| Styling | Tailwind CSS |
| Components | shadcn/ui |
| Charts | Recharts |
| Animations | Framer Motion |
| Backend | Supabase (auth + database + edge functions) |
| Auth | Email/password + Google OAuth + demo mode |
| State | Zustand o React Context |
| Mobile breakpoint | 375px primary |

---

## 4. Architettura

```
App
├── Onboarding (unauthenticated)          → epic-01-onboarding.md
│   ├── Step 1: Account (email/password + OAuth)
│   ├── Step 2: Scegli le aree
│   ├── Step 3: Imposta frequenza
│   └── → Home
│
└── Main App (authenticated)
    ├── [Tab 1] Home (hub giornaliero)     → epic-02-dashboard.md
    ├── [Tab 2] Attività                   → epic-10-areas.md / epic-04-area-detail.md / epic-05-add-edit-area.md
    ├── [Tab 3] Progress                   → epic-12-progress.md
    ├── [Tab 5°] Finance (opzionale)       → epic-06-finance.md · abilitabile da Impostazioni
    └── [Ultima] Impostazioni              → epic-07-settings.md
```

### Navigazione adattiva
- **Mobile (< 1024px):** Bottom nav fissa, 56px, background `#0F2F33` — solo icone
- **Desktop (≥ 1024px):** Sidebar laterale sinistra, 240px — icone + label
- **4 tab fisse:** Home · Attività · Progress · Impostazioni — uguali per tutti, non configurabili
- **5° tab opzionale:** Finance — attivabile da Impostazioni (toggle), non visibile di default
- Icone Lucide outline 24px — active `#7DA3A0`, inactive `#B9C0C1`
- Vedi `epic-09-layout.md` per il comportamento responsivo completo
- Vedi `architecture/navigation-v2.md` per la razionale architetturale completa

---

## 5. Feature Map

| # | Feature | Epic | Priorità |
|---|---|---|---|
| 01 | Onboarding (auth + setup aree) | `epic-01-onboarding.md` | P0 |
| 02 | Home — hub giornaliero con lista attività di oggi e check-in | `epic-02-dashboard.md` | P0 |
| 03 | Check-in giornaliero | `epic-03-checkin.md` | P0 |
| 04 | Area Detail — analisi traiettoria | `epic-04-area-detail.md` | P1 |
| 05 | Add / Edit Area | `epic-05-add-edit-area.md` | P0 |
| 06 | Finance Projection (5° tab opzionale) | `epic-06-finance.md` | P2 |
| 07 | Settings (lingua, score, notifiche, account) | `epic-07-settings.md` | P1 |
| 08 | Lingua IT / EN | `epic-08-i18n.md` | P1 |
| 09 | Layout responsivo — 4 tab fisse + optional 5th | `epic-09-layout.md` | P1 |
| 10 | Attività — 4 macro-categorie + gestione aree | `epic-10-areas.md` | P0 |
| 11 | Gym Card — scheda palestra | `epic-11-gym.md` | P2 |
| 12 | Progress — osservazione traiettoria globale | `epic-12-progress.md` | P1 |
| 13 | Abitudini da ridurre — tracciamento quantitativo | `epics/checkin/epic-03-checkin.md` · `epics/edit_area/epic-05-add-edit-area.md` | P1 |
| 14 | Google Tasks — sync bidirezionale con Google | `epics/google-tasks/epic-13-google-tasks.md` | P2 |
| 15 | Theme system — dark/light/system + 4 palette colori | `epics/settings/stories.md` (story-07-08) | P2 |
| 16 | Demo mode — accesso senza autenticazione | `epics/auth/stories.md` (story-00-08) | P2 |

---

## 6. Database (Supabase — schema di riferimento)

RLS abilitato su tutte le tabelle.

```
users        → id, settings_score_visible (default false), settings_notifications, extra_tab_enabled (default false)
areas        → id, user_id, name, type, frequency_per_week, archived_at,
               tracking_mode (binary | quantity_reduce), unit_label, baseline_initial, show_quick_add_home
checkins     → id, area_id, user_id, date, completed — UNIQUE(area_id, date)
score_daily  → id, area_id, date, daily_score, trajectory_state, consecutive_missed
habit_quantity_daily → id, area_id, date, quantity, source (quick_add | manual_edit | system_init),
                       created_at, updated_at — UNIQUE(area_id, date)
```

**Modello di traiettoria comportamentale per `tracking_mode = binary`:**

Il sistema usa un modello EWMA (Exponentially Weighted Moving Average) che stima la **direzione corrente** del comportamento, non il totale accumulato.

```
Segnale giornaliero (input_t):
  completato       → +1.0
  primo mancato    →  0.0
  secondo mancato  → -0.5
  terzo+ mancato   → -1.0

Aggiornamento stato traiettoria:
  state_t = state_(t-1) + α × (input_t − state_(t-1))
  α = 0.08  (memoria effettiva ≈ 25–30 giorni)
```

Il `trajectory_state` alimenta il grafico. Non viene mai mostrato come numero (default off).
Vedi specifiche complete in `architecture/behavioral-trajectory-model.md`.

**Evaluation logic per `tracking_mode = quantity_reduce` (interno, non mostrato):**
```
if history_days < 7:
    reference_qty = baseline_initial
else:
    reference_qty = moving_average(last_7_days)

delta = qty_today - reference_qty
→ improving : qty_today < reference_qty
→ stable    : qty_today ≈ reference_qty
→ worsening : qty_today > reference_qty
```

> Le aree `quantity_reduce` mostrano il grafico della quantità giornaliera — non usa `score_daily`.

---

## 6b. Database — aggiornamento

Colonne aggiunte alla tabella `users`:
```
language           TEXT     DEFAULT 'en'   CHECK (language IN ('it', 'en'))
extra_tab_enabled  BOOLEAN  DEFAULT false  -- abilita il 5° tab Finance nella nav
```

> `menu_custom_items` è deprecata. Sarà sostituita da `extra_tab_enabled` in migrazione.

Nuove tabelle per Gym Card:
```
gym_sessions   → id, area_id, user_id, date
gym_exercises  → id, session_id, name, sets, reps, weight_kg (nullable), notes (nullable), order
```

---

## 7. Scope MVP

### ✅ Incluso
- Autenticazione email/password + OAuth Google
- Lingua IT / EN (rilevamento browser + cambio in Settings)
- 4 tipi area (Health, Study, Reduce, Finance)
- Check-in binario giornaliero per aree standard
- Tracciamento quantitativo giornaliero per abitudini da ridurre (`quantity_reduce`)
- Dashboard con grafico totale aggregato + filtro per macro-area
- Sezione Aree con 4 macro-categorie
- Grafico traiettoria per area (Area Detail)
- Time range: 30d / 90d / 365d
- Calendar heatmap 30 giorni
- Proiezione lineare Finance
- Toggle visibilità score (default off)
- Toggle notifiche
- Layout responsivo: bottom nav mobile · sidebar desktop
- Menu configurabile: 3 voci fisse + max 2 slot custom
- Gym Card per area Gym/Palestra

### ❌ Escluso
- Feature social o condivisione
- Badge, punti, livelli, classifiche
- Gamificazione streak
- Push notifications (toggle presente, logica rinviata)
- Export / download dati
- Suggerimenti AI

---

## 8. Riferimenti

- Brand system completo: `brand-system/betonme_brand_system_lovable.md`
- Regole design e anti-pattern: sezione 10 del brand system
- Tono di voce: mai valutativo, mai motivazionale — solo osservativo

---

*BetonMe PRD v2.0 — Mappa prodotto. Il dettaglio vive negli epic.*
