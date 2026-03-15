# Stories — Epic 10 — Attività (Sezione Aree)

## Sequenza di implementazione

```
story-10-01 → Layout sezione Aree con 4 macro-categorie e liste area         ✅ completata
story-10-02 → CTA aggiungi per tipo + CTA globale header                     ✅ completata
story-10-03 → Stati empty, loading e aree archiviate                          ✅ completata
story-10-04 → Rename route /areas → /activities, label "Attività", back nav  ⏳ da fare
story-10-05 → Badge totale giornaliero nelle card Reduce quantitative          ⏳ da fare
```

> **Dipendenze:** Richiede Epic 04 (Area Detail) e Epic 05 (Add/Edit Area) per la navigazione. Epic 08 (i18n) per i label.

---

## story-10-01 — Layout sezione Aree con 4 macro-categorie

BetonMe è un'app di osservazione del benessere. Sostituisci il placeholder della pagina Areas (route `/areas`) con il layout completo.

**Struttura:**
- Header: `"Aree" / "Areas"` + icona `+` in alto a destra (CTA globale)
- 4 sezioni verticali — una per macro-area — sempre visibili anche se vuote:

**Sezione Salute / Health** (icona `Heart`)
- Lista delle aree dell'utente con `type = "health"` e `archived_at IS NULL`
- Ogni area: riga con nome a sinistra + chevron `>` a destra, background `#1F4A50`, min-height 48px, rounded
- Tap su riga → naviga ad Area Detail (`/areas/:id`)

**Sezione Studio / Study** (icona `BookOpen`)
- Stessa struttura, filtra `type = "study"`

**Sezione Riduci / Reduce** (icona `TrendingDown`)
- Stessa struttura, filtra `type = "reduce"`

**Sezione Finanze / Finance** (icona `Wallet`)
- Stessa struttura, filtra `type = "finance"`

**Label macro-aree (seguono lingua utente):**
- IT: `Salute · Studio · Riduci · Finanze`
- EN: `Health · Study · Reduce · Finance`

---

## story-10-02 — CTA aggiungi

Continua Epic 10 di BetonMe. Aggiungi le CTA per creare nuove aree.

**CTA per tipo (in fondo a ogni sezione):**
- Label IT: `+ Aggiungi` / EN: `+ Add`
- Stile: link testuale piccolo, colore `#7DA3A0`
- Tap → naviga a `/areas/new` con il tipo della sezione pre-selezionato come query param (es. `?type=health`)

**CTA globale (icona `+` nell'header):**
- Tap → naviga a `/areas/new` senza pre-selezione del tipo

---

## story-10-03 — Stati empty, loading e aree archiviate

Continua Epic 10 di BetonMe. Gestisci gli stati speciali della sezione Aree.

**Macro-area senza aree (empty per quel tipo):**
- Non mostrare card — mostrare solo la CTA `+ Aggiungi / + Add`
- Nessun testo aggiuntivo, nessuna icona

**Loading state:**
- Skeleton `animate-pulse` per ogni sezione: 1-2 righe placeholder per macro-area
- Il layout a 4 sezioni rimane visibile con i titoli

**Aree archiviate:**
- Non mostrate nella lista (filtrate dalla query: `archived_at IS NULL`)
- Le aree archiviate rimangono accessibili solo tramite link diretto (se necessario)

**Nome area lungo:**
- Troncato con `text-ellipsis overflow-hidden` — nessun wrapping

---

## story-10-04 — Rename route e label

Continua Epic 10 di BetonMe. Aggiorna la route e il label della sezione Aree per allinearlo all'architettura di navigazione v2.

**Dipende da:** story-09-04 (refactor nav con `useNavConfig`)

**Cosa cambia:**
- Route `/areas` → `/activities`
- Route `/areas/new` → `/activities/new`
- Route `/areas/:id` → `/activities/:id`
- Route `/areas/:id/edit` → `/activities/:id/edit`
- File `src/pages/Areas.tsx` → rinomina in `src/pages/Activities.tsx`
- Tab label IT: `"Aree"` → `"Attività"` / EN: `"Areas"` → `"Activities"`

**Back navigation da Area Detail:**
- Il back button in Area Detail deve tornare a `/activities` (non a `/` come ora)
- Aggiorna l'header di Area Detail: `← Attività` / `← Activities`

**Aggiorna tutti i redirect interni** da `/areas*` a `/activities*` (form, CTA, nav link).

**Comportamento invariato:** tutto il resto (4 sezioni, card, CTA, filtri) rimane identico.

---

## story-10-05 — Badge totale giornaliero nelle card Reduce quantitative ⏳

Continua Epic 10 di BetonMe. Aggiorna le card delle aree `quantity_reduce` nella sezione Riduci / Reduce per mostrare il totale di oggi accanto al nome.

**Condizione:**
- L'area ha `tracking_mode = 'quantity_reduce'`

**Cosa mostra nella card (variante rispetto alla card standard):**
- Riga: nome area a sinistra + badge `"N oggi"` (IT) / `"N today"` (EN) al centro-destra + chevron `>` a destra
- Il badge è testo piccolo `text-[#B9C0C1]` — secondario, non invadente
- Tap → naviga ad Area Detail (invariato)

**Logica del badge:**
- Legge il record `habit_quantity_daily` WHERE `area_id = X` AND `date = oggi`
- Se nessun record → mostra `"0 oggi"` / `"0 today"`
- Se record presente → mostra il valore `quantity`

**Cosa NON fare:**
- Il badge non è interattivo (non è un bottone +1)
- Non aggiungere colori di valutazione al badge (nessun rosso/verde per "meglio/peggio")
- Non mostrare baseline o confronti nel badge
