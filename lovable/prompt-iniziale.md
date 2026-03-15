# BetonMe — Prompt iniziale per Lovable

> **Istruzioni per l'utente:** Allegare questo prompt su Lovable come primo messaggio, con `prd.md` e `brand-system/brand_system.md` caricati come documenti di contesto permanente del progetto.

---

## Prompt

BetonMe è una **mobile-first React web app** (breakpoint primario: 375px) per osservare la traiettoria del benessere personale nel tempo. Gli utenti registrano un'azione binaria al giorno per area di vita. Il grafico è il prodotto. Tono: neutro, osservativo — mai valutativo, mai motivazionale.

Il PRD e il brand system sono allegati come contesto di progetto. Usali come riferimento assoluto per design, colori e comportamento.

---

### Step 1 — Database Supabase

Crea le seguenti 4 tabelle con RLS abilitato su tutte:

```
users        → id, settings_score_visible (boolean, default false), settings_notifications (boolean, default true)
areas        → id, user_id, name, type (health | study | reduce | finance), frequency_per_week (integer 1–7), archived_at (timestamp nullable)
checkins     → id, area_id, user_id, date (date), completed (boolean) — UNIQUE(area_id, date)
score_daily  → id, area_id, date (date), daily_score (float), cumulative_score (float), consecutive_missed (integer)
```

---

### Step 2 — Navigation shell

Bottom navigation bar fissa (56px), visibile solo per utenti autenticati. 4 tab con icone Lucide outline 24px:

- **Home** → icona `LayoutDashboard`, route `/`
- **Areas** → icona `Layers`, route `/areas`
- **Finance** → icona `BarChart2`, route `/finance`
- **Settings** → icona `Settings`, route `/settings`

---

### Step 3 — Route guard

- Tutte le route sopra richiedono autenticazione Supabase
- Utente non autenticato → redirect automatico a `/login`
- Sessione valida al riavvio app → accesso diretto a `/` (nessun login ripetuto)

---

### Step 4 — Schermate placeholder

- `/login` → pagina vuota con testo "Login" (verrà costruita nella story successiva)
- `/`, `/areas`, `/finance`, `/settings` → tab con il nome come contenuto placeholder

---

> Questo scaffold non include contenuto visivo definitivo. Il design delle singole schermate viene costruito storia per storia a partire dalla prossima sessione.
