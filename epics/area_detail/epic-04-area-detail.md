# Epic 04 — Area Detail (Scheda Attività)

## Obiettivo

Offrire all'utente un punto unico per **configurare** un'attività: modificarne i metadati, assegnare i giorni della settimana alle ripetizioni programmate, e consultare la cronologia degli appunti scritti nel tempo.

> La schermata è gestione e memoria — non è interazione né osservazione. Il log si fa in Home (Epic 02). I trend si osservano in Progress (Epic 12).

---

## Behavior

L'utente arriva all'Area Detail tappando un'area nella tab Attività (`/activities`). Trova tre sezioni verticali:

1. **Modifica attività** — CTA nell'header per editare nome, tipo e frequenza
2. **Giorni programmati** — chip dei 7 giorni della settimana per assegnare le ricorrenze settimanali
3. **Cronologia appunti** — lista degli appunti scritti nel tempo per questa attività

Non c'è bottone di check-in. Non c'è grafico. Non c'è heatmap.

---

## Flussi

### Modifica attività

1. L'utente tappa il link/icona "Modifica" nell'header
2. Naviga ad Add/Edit Area (Epic 05) in modalità edit, route `/activities/:id/edit`

### Assegnazione giorni programmati

1. L'utente vede 7 chip dei giorni della settimana (Lun…Dom / Mon…Sun)
2. I chip già assegnati sono evidenziati
3. Tap su un chip disattivato → lo attiva (se il numero di giorni selezionati non supera `frequency_per_week`)
4. Tap su un chip attivo → lo disattiva
5. Ogni modifica viene salvata immediatamente (auto-save, aggiornamento ottimistico)
6. Se tutti e 7 i slot sono occupati (`frequency_per_week = 7`), tutti i chip sono attivi e non modificabili

### Consultazione cronologia appunti

1. La sezione "Appunti / Notes" mostra la lista degli appunti scritti in Home per questa attività
2. Ogni voce mostra: data + testo
3. Ordine cronologico inverso (più recente in cima)
4. Gli appunti sono **read-only** qui — la modifica avviene in Home per ogni singola occorrenza (area + giorno)

---

## Layout

```
┌─────────────────────────────────────────┐
│ ←  [Nome Area]  [pill tipo]   [Modifica]│  ← Header
├─────────────────────────────────────────┤
│                                         │
│  Giorni programmati                     │  ← Sezione 1
│  Seleziona N giorni                     │  ← subtitolo con slot disponibili
│  [Lun] [Mar] [Mer] [Gio] [Ven] [Sab] [Dom] │  ← chip 7 giorni
│                                         │
├─────────────────────────────────────────┤
│                                         │
│  Appunti / Notes                        │  ← Sezione 2
│  ┌─────────────────────────────────┐    │
│  │ 12 mar 2026                     │    │
│  │ Testo dell'appunto…             │    │
│  └─────────────────────────────────┘    │
│  ┌─────────────────────────────────┐    │
│  │ 8 mar 2026                      │    │
│  │ Testo dell'appunto…             │    │
│  └─────────────────────────────────┘    │
│                                         │
│  (empty state se nessun appunto)        │
│                                         │
├─────────────────────────────────────────┤
│  [Nav]                                  │
└─────────────────────────────────────────┘
```

---

## Sezione Giorni Programmati

### Regole

| Caso | Comportamento |
|---|---|
| `frequency_per_week = 7` | Tutti e 7 i chip sono attivi e non modificabili |
| `frequency_per_week = 1` | L'utente può selezionare al massimo 1 giorno |
| `frequency_per_week = N` | L'utente può selezionare al massimo N giorni |
| Nessun giorno configurato | Sezione mostra i chip tutti inattivi — il subtitolo invita a selezionare |
| N giorni già selezionati | Se si tenta di aggiungere oltre il limite, il chip è visivamente disabilitato |

### Dati

- Tabella nuova: `area_scheduled_days`
- Campi: `area_id`, `day_of_week` (integer: 1=Lun, 2=Mar, …, 7=Dom — ISO week: lunedì = 1)
- Una riga per giorno assegnato
- `ON CONFLICT` per evitare duplicati

### Impatto su Home (Epic 02)

- Se un'area ha giorni configurati → in Home compare **solo** nei giorni della settimana assegnati
- Se un'area **non** ha giorni configurati → compare ogni giorno (backward compat)
- Il confronto usa il `day_of_week` del giorno selezionato nel WeekSelector

### Chip UI

| Stato | Stile |
|---|---|
| Inattivo (non assegnato) | Bordo `border-border`, testo `text-muted-foreground`, bg trasparente |
| Attivo (assegnato) | Bg `bg-primary/20`, bordo `border-primary`, testo `text-primary` |
| Disabilitato (limite raggiunto) | `opacity-40`, non interattivo — solo per i chip non ancora selezionati |
| Non modificabile (freq = 7) | Tutti attivi, nessun tap |

### Subtitolo sezione

- IT: `"Seleziona {N} giorni"` dove N = `frequency_per_week` meno il numero già selezionato
- EN: `"Select {N} days"`
- Se N = 0 (tutti i giorni già assegnati): `"Tutti i giorni assegnati"` / `"All days assigned"`
- Se `frequency_per_week = 7`: nessun subtitolo — sezione informativa

---

## Sezione Cronologia Appunti

### Dati

- Fonte: tabella `activity_notes` (`area_id`, `date`, `content`, `user_id`)
- Query: tutti i record dove `area_id = :id AND user_id = :user_id`, ordinati per `date DESC`

### Layout voce

| Elemento | Stile |
|---|---|
| Data | `text-sm text-muted-foreground` — formato localizzato (es. "12 mar 2026" / "Mar 12, 2026") |
| Testo | `text-base text-foreground` |
| Card | `bg-card rounded-xl p-4` |

### Empty state

- IT: `"Nessun appunto ancora. Gli appunti si aggiungono dalla Home durante il log."`
- EN: `"No notes yet. Notes are added from Home when logging."`
- Stile: testo piccolo centrato, `text-muted-foreground`

---

## Header

| Elemento | Dettaglio |
|---|---|
| Back nav `←` | Torna a `/activities` |
| Nome area | `text-[18px] font-semibold` |
| Pill tipo | `<AreaTypePill>` esistente |
| CTA Modifica | Link testuale `"Modifica"` (IT) / `"Edit"` (EN), a destra nell'header |

---

## Stati UI

### Loading

```
Skeleton chip giorni: 7 rettangoli animate-pulse rounded-full
Skeleton appunti: 2 card bg-card animate-pulse rounded-xl h-20
```

### Area non trovata

```
Testo: "Attività non trovata" / "Activity not found"
Testo small muted, centrato
```

---

## Edge Case

- Area con `frequency_per_week = 7` → tutti i chip attivi, nessuna interazione possibile
- Area appena creata senza giorni configurati → sezione giorni mostra chip tutti inattivi
- Area con 0 appunti → empty state sezione appunti
- Back navigation dopo edit area → ritorna a `/activities/:id` (non a `/activities`)
- Area Gym con scheda → la GymCard (Epic 11) NON compare in Area Detail — la scheda è accessibile solo dalla Home via CTA "Apri scheda"
- Aree `quantity_reduce` → la sezione giorni programmati funziona esattamente come per le aree binary

---

## Acceptance Criteria

- [ ] Nessun bottone check-in in questa schermata
- [ ] Nessun grafico o heatmap in questa schermata
- [ ] Il link "Modifica" nell'header naviga a `/activities/:id/edit`
- [ ] La sezione giorni mostra 7 chip con i giorni selezionati evidenziati
- [ ] Tap su chip non selezionato → lo assegna (se sotto il limite)
- [ ] Tap su chip selezionato → lo rimuove
- [ ] Il numero massimo di giorni selezionabili = `frequency_per_week`
- [ ] Con `frequency_per_week = 7` i chip sono tutti attivi e non interattivi
- [ ] Le modifiche ai giorni si salvano immediatamente (auto-save ottimistico)
- [ ] La sezione appunti mostra le note per questa area in ordine cronologico inverso
- [ ] Ogni nota mostra data e testo
- [ ] Se nessuna nota: empty state con copy corretto
- [ ] La back nav torna a `/activities`
- [ ] Il subtitolo della sezione giorni indica quanti slot rimangono
- [ ] Label dei chip seguono la lingua utente (IT/EN)

---

## Note UI

- Nessun grafico — i trend vivono in Progress (Epic 12)
- Nessun check-in — il log è solo in Home (Epic 02)
- La GymCard non compare qui — accessibile dalla Home
- Brand tokens: `brand-system/brand_system.md`

---

## Dipendenze

- Epic 02 (Home) — impatto sul filtro giorni nel WeekSelector
- Epic 05 (Add/Edit Area) — navigazione al form di modifica
- Epic 08 (i18n) — label IT/EN chip giorni + copy sezioni
- Epic 11 (Gym) — la GymCard non compare qui (rimozione)
- Epic 12 (Progress) — grafico e heatmap migrati lì

---

## Stato implementazione

**Da riscrivere** — l'attuale `src/pages/AreaDetail.tsx` va refactorizzato:

| Da rimuovere | Motivo |
|---|---|
| Bottone check-in (`handleCheckIn`, `handleAutoCheckIn`) | Log solo in Home |
| `<CalendarHeatmap>` | Migrato in Progress |
| `<GymCard>` in Area Detail | Accessibile solo da Home |

| Da aggiungere | Componente |
|---|---|
| Chip picker giorni | `<ScheduledDaysPicker>` |
| Lista appunti | `<NotesHistory>` |
| Tabella DB | `area_scheduled_days` |

---

## Stories

- `story-04-01` — Layout Area Detail: header + modifica + struttura sezioni — **da riscrivere**
- `story-04-02` — Sezione giorni programmati: chip picker + auto-save — **nuova**
- `story-04-03` — Sezione cronologia appunti: lista read-only — **nuova**
- `story-04-04` — Impatto su Home: filtro attività per giorno della settimana — **nuova**
