# Stories — Epic 04 — Area Detail (Scheda Attività)

## Sequenza di implementazione

```
story-04-01 → Layout base: header, back nav, struttura sezioni, rimozione check-in e grafico
story-04-02 → Sezione giorni programmati: tabella + chip picker + auto-save
story-04-03 → Sezione cronologia appunti: lista read-only
story-04-04 → Impatto su Home: filtro attività per giorno programmato
```

> **Dipendenze:** Richiede Epic 05 (Add/Edit Area) per la navigazione al form. Epic 08 (i18n) per le label.

---

## story-04-01 — Layout Area Detail: header, struttura sezioni, rimozione check-in

opad.me è un'app di osservazione del benessere personale. Stiamo riscrivendo la schermata Area Detail (route `/activities/:id`).

**Cosa fa questa schermata:**
La schermata Area Detail è una schermata di configurazione e memoria — non di log. Il check-in si fa solo dalla Home.

**Rimuovi completamente:**
- Il bottone check-in (qualsiasi `<CheckInButton>`, `handleCheckIn`, `handleAutoCheckIn`)
- Il `<CalendarHeatmap>` (migrato in Progress)
- Il `<TimeRangeSelector>` e qualsiasi grafico

**Mantieni:**
- Il `<GymCard>` per aree gym/palestra — resta in Area Detail come punto di accesso alla scheda
- Auto check-in al primo esercizio completato nella sessione gym

**Header da costruire:**
- `←` back navigation verso `/activities`
- Nome area come titolo (`text-[18px] font-semibold`)
- `<AreaTypePill>` accanto al nome
- Link testuale a destra: `"Modifica"` (IT) / `"Edit"` (EN) → naviga a `/activities/:id/edit`

**Struttura schermata (3 sezioni verticali, per ora con placeholder):**
1. Sezione `"Giorni programmati"` / `"Scheduled days"` — placeholder, verrà costruita in story-04-02
2. Sezione `"Appunti"` / `"Notes"` — placeholder, verrà costruita in story-04-03

**Loading state:**
- Skeleton: 2 rettangoli `bg-card animate-pulse rounded-xl h-24`

**Area non trovata:**
- Testo `"Attività non trovata"` / `"Activity not found"` centrato, `text-muted-foreground`

---

## story-04-02 — Sezione giorni programmati

Continua Epic 04 di opad.me. Costruisci la sezione **"Giorni programmati" / "Scheduled days"** nella schermata Area Detail.

**Crea la tabella `area_scheduled_days`:**
- `id` (uuid, PK)
- `area_id` (uuid, FK → areas.id, CASCADE DELETE)
- `day_of_week` (integer: 1=Lunedì, 2=Martedì, …, 7=Domenica — standard ISO)
- `user_id` (uuid, FK → auth.users)
- `created_at` (timestamp)
- Unique constraint su `(area_id, day_of_week)`
- RLS: l'utente vede solo i propri record

**UI — chip picker:**
- 7 chip orizzontali, uno per giorno della settimana
- Label chip IT: `Lun Mar Mer Gio Ven Sab Dom`
- Label chip EN: `Mon Tue Wed Thu Fri Sat Sun`
- Chip attivo (giorno assegnato): `bg-primary/20 border-primary text-primary`
- Chip inattivo: `border-border text-muted-foreground bg-transparent`
- Chip disabilitato (limite raggiunto, non ancora selezionato): `opacity-40` non interattivo

**Logica:**
- Il numero massimo di chip attivi = `area.frequency_per_week`
- Se `frequency_per_week = 7`: tutti i chip sono attivi e non interattivi (nessun tap)
- Tap su chip inattivo + sotto il limite → inserisce riga in `area_scheduled_days` (auto-save ottimistico)
- Tap su chip attivo → elimina la riga corrispondente (auto-save ottimistico)
- Se si tenta di selezionare oltre il limite: il chip è visivamente disabilitato, tap non ha effetto

**Subtitolo sezione:**
- Quanti giorni rimangono da selezionare: `"Seleziona ancora {N} giorni"` / `"Select {N} more days"`
- Se tutti i giorni sono già assegnati: `"Tutti i giorni assegnati"` / `"All days assigned"`
- Se `frequency_per_week = 7`: nessun subtitolo

**Loading state:**
- 7 rettangoli skeleton `animate-pulse rounded-full`

---

## story-04-03 — Sezione cronologia appunti

Continua Epic 04 di opad.me. Costruisci la sezione **"Appunti" / "Notes"** nella schermata Area Detail.

**Dati:**
- Leggi dalla tabella `activity_notes` dove `area_id = :id AND user_id = :user_id`
- Ordina per `date DESC` (più recente in cima)

**Layout ogni voce:**
- Card `bg-card rounded-xl p-4`
- Prima riga: data formattata nella lingua utente (es. IT: "12 mar 2026" / EN: "Mar 12, 2026") — `text-sm text-muted-foreground`
- Seconda riga: testo dell'appunto — `text-base text-foreground`
- Le note sono **read-only** — nessuna azione di modifica o eliminazione qui

**Empty state (nessuna nota per questa area):**
- IT: `"Nessun appunto ancora. Gli appunti si aggiungono dalla Home durante il log."`
- EN: `"No notes yet. Notes are added from Home when logging."`
- Stile: testo piccolo centrato, `text-muted-foreground`

**Loading state:**
- 2 card skeleton `bg-card animate-pulse rounded-xl h-20`

---

## story-04-04 — Impatto su Home: filtro per giorno programmato

Continua Epic 04 di opad.me. Aggiorna la **Home** (`src/pages/Index.tsx`) per rispettare i giorni programmati.

**Logica di filtraggio nella Home:**
- Al caricamento e al cambio del giorno selezionato nel WeekSelector, determina il `day_of_week` del giorno selezionato (1=Lun … 7=Dom)
- Per ogni area attiva:
  - Se l'area ha righe in `area_scheduled_days` → mostra la card **solo** se il `day_of_week` del giorno selezionato è tra i giorni assegnati
  - Se l'area **non** ha righe in `area_scheduled_days` → mostra la card ogni giorno (backward compat — comportamento attuale)
- Fetch ottimizzato: una sola query per tutti i giorni programmati delle aree dell'utente al caricamento

**Empty state giorno senza attività:**
- Se per il giorno selezionato nessuna area è programmata:
  - IT: `"Nessuna attività programmata per questo giorno."`
  - EN: `"No activities scheduled for this day."`
  - Stile: testo piccolo centrato, `text-muted-foreground`
  - Diverso dall'empty state "nessuna area" che ha la CTA verso Attività

**Note:**
- Non cambia nulla alla logica del check-in — se un'area ha check-in registrati per un giorno non programmato (dati storici), non vengono toccati
- Il WeekSelector non cambia — mostra sempre i 7 giorni
