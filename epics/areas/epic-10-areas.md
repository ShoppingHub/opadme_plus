# Epic 10 — Attività (Sezione Aree)

## Obiettivo
Offrire all'utente una visione organizzata delle proprie aree di vita raggruppate per macro-categoria, con accesso rapido alla creazione di nuove aree e alla visualizzazione del dettaglio.

> **Nota architetturale:** Questa è la tab Attività (Tab 2). Route: `/activities` (aggiornata da `/areas`). Il nome "Attività" riflette meglio il ruolo di layer strutturale del sistema — qui l'utente gestisce le proprie aree di osservazione.

---

## Behavior

La schermata Areas mostra le 4 macro-aree come sezioni distinte. All'interno di ogni sezione compaiono le aree che l'utente ha creato per quella categoria. Se non esistono aree per una macro-categoria, la sezione mostra un micro empty state con CTA per aggiungere.

Le macro-aree sono sempre visibili — anche se vuote — per aiutare l'utente a capire come è strutturata la sua osservazione.

---

## Macro-aree

| Tipo | Label IT | Label EN | Icona Lucide |
|---|---|---|---|
| `health` | Salute | Health | `Heart` |
| `study` | Studio | Study | `BookOpen` |
| `reduce` | Riduci | Reduce | `TrendingDown` |
| `finance` | Finanze | Finance | `Wallet` |

---

## Layout

```
┌─────────────────────────────┐
│ Aree / Areas          [+ ]  │  ← Header con CTA globale
├─────────────────────────────┤
│                             │
│  ♥ Salute / Health          │  ← Macro-area header
│  ┌─────────────────────┐    │
│  │ Palestra     →      │    │  ← Area card (tap → Area Detail)
│  │ Camminata    →      │    │
│  └─────────────────────┘    │
│  [+ Aggiungi]               │  ← CTA add area di questo tipo
│                             │
│  📖 Studio / Study          │
│  (vuoto)                    │
│  [+ Aggiungi]               │
│                             │
│  📉 Riduci / Reduce         │
│  ┌─────────────────────┐    │
│  │ Social media  →     │    │
│  └─────────────────────┘    │
│  [+ Aggiungi]               │
│                             │
│  💰 Finanze / Finance       │
│  ┌─────────────────────┐    │
│  │ Risparmio    →      │    │
│  └─────────────────────┘    │
│  [+ Aggiungi]               │
│                             │
├─────────────────────────────┤
│  [Nav]                      │
└─────────────────────────────┘
```

---

## Specifiche Area Card (nella lista)

| Proprietà | Valore |
|---|---|
| Layout | Riga orizzontale: nome area a sinistra, chevron a destra |
| Background | `bg-[#1F4A50] rounded-lg` |
| Altezza riga | min 48px (touch target) |
| Tap | Naviga ad Area Detail |
| Stato archiviato | Non mostrato (filtro lato query) |

### Variante: area Reduce quantitativa (`tracking_mode = quantity_reduce`)

Le aree `quantity_reduce` nella sezione Riduci / Reduce mostrano un badge secondario con il totale di oggi, accanto al nome area.

| Proprietà | Valore |
|---|---|
| Layout | Nome area a sinistra + badge `"N oggi"` (IT) / `"N today"` (EN) + chevron a destra |
| Badge | Testo piccolo `text-[#B9C0C1]` — secondario rispetto al nome area |
| Tap | Naviga ad Area Detail (dove l'utente può interagire col QuantityCounter completo) |

> Il badge è solo informativo — non interattivo. Il quick-add principale è in Home (se `show_quick_add_home = true`).

---

## CTA Aggiungi

- Label IT: `+ Aggiungi` / Label EN: `+ Add`
- Stile: link testuale piccolo, `text-[#7DA3A0]`
- Tap: naviga ad Add Area (Epic 05) con tipo pre-selezionato

## CTA Header (globale)

- Icona `+` in alto a destra nell'header
- Tap: naviga ad Add Area senza pre-selezione del tipo

---

## Stati UI

### Macro-area senza aree
```
Nessuna card — solo la CTA "+ Aggiungi / + Add"
```

### Macro-area con aree
```
Lista card + CTA in fondo alla sezione
```

### Loading
```
Skeleton animate-pulse per ogni sezione
```

---

## Edge Case

- Utente senza nessuna area → tutte le sezioni mostrano il micro empty state con CTA
- Area archiviata → non compare nella lista (filtrata in query)
- Nome area molto lungo → troncato con `text-ellipsis overflow-hidden`
- Più di 5 aree per macro-categoria → scroll verticale naturale della pagina
- Area `quantity_reduce` senza dati oggi → il badge mostra `"0 oggi"` / `"0 today"`

---

## Acceptance Criteria

- [x] Le 4 macro-aree sono sempre visibili anche se vuote
- [x] Ogni macro-area mostra le aree dell'utente di quel tipo
- [x] Le aree archiviate non compaiono
- [x] La CTA "+ Aggiungi / + Add" apre Add Area con tipo pre-selezionato
- [x] Il tap su un'area card naviga ad Area Detail
- [x] La CTA header apre Add Area senza pre-selezione
- [x] Il loading state mostra skeleton animate-pulse
- [x] Le label delle macro-aree seguono la lingua selezionata (Epic 08)

---

## Stato implementazione

**Completato (da aggiornare route)** — componenti esistenti in `ShoppingHub/project-spark`.

| Componente | File | Stato |
|---|---|---|
| Pagina Attività | `src/pages/Areas.tsx` → da rinominare `Activities.tsx` | Da rinominare |
| Route | `/areas` → `/activities` | Da aggiornare |
| Tab label | `"Aree"` → `"Attività"` (IT) / `"Activities"` (EN) | Da aggiornare |

### Dettagli implementazione esistente (da preservare)
- Filtro `.is("archived_at", null)` applicato correttamente
- CTA per tipo con query param `?type=<type>` su `/areas/new` → aggiornare a `/activities/new`
- CTA globale header con icona `+` → `/activities/new` senza pre-selezione
- Area card: `bg-[#1F4A50] rounded-lg`, nome + chevron, min-h 48px

---

## Dipendenze

- Epic 05 (Add/Edit Area) — per la navigazione al form
- Epic 04 (Area Detail) — per la navigazione al dettaglio
- Epic 08 (i18n) — per le label in IT/EN

---

## Stories

- `story-10-01` — Layout sezione Aree con 4 macro-categorie e liste area — **completata**
- `story-10-02` — CTA aggiungi per tipo + CTA globale header — **completata**
- `story-10-03` — Stati empty, loading e area archiviata — **completata**
- `story-10-04` — Rename route `/areas` → `/activities`, label "Attività", back nav da Area Detail — **da fare**
- `story-10-05` — Badge totale giornaliero nelle card Reduce quantitative — **da fare**
