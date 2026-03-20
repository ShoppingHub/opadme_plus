# Prompt Lovable — Epic 15 — Plus (Abbonamento Premium)

> **Istruzioni:** Eseguire i prompt in ordine. Ogni prompt è un'iterazione Lovable autonoma. Il PRD, il brand system e l'epic-15-plus.md sono allegati come contesto di progetto.

---

## Prompt 1 — Schema DB + hook Plus

opad.me è un'app di osservazione del benessere personale. Stiamo aggiungendo un abbonamento **Plus** che sblocca funzionalità avanzate (schede, tracking riduzione, temi extra).

Aggiungi 4 colonne alla tabella `users`:

| Colonna | Tipo | Default | Note |
|---|---|---|---|
| `plus_active` | BOOLEAN | `false` | Stato abbonamento |
| `plus_activated_at` | TIMESTAMPTZ | NULL | Data prima attivazione |
| `plus_expires_at` | TIMESTAMPTZ | NULL | Scadenza (NULL = nessuna) |
| `plus_provider` | TEXT | NULL | CHECK IN ('stripe', 'apple', 'google', 'manual') |

RLS: l'utente può leggere il proprio `plus_active`. Solo il service role può modificare le colonne Plus.

Crea un nuovo hook `usePlusStatus` come context provider (stessa struttura di `useAuth` o `useTheme`). Espone:

- `isPlusActive` — booleano, legge `plus_active` dalla tabella users per l'utente corrente
- `isFeatureLocked(feature)` — accetta `'cards' | 'quantity_reduce' | 'themes_extra'`, ritorna `true` se `!isPlusActive`

Wrappa l'app con il nuovo provider (in `App.tsx`, dentro `AuthProvider`). In demo mode, `isPlusActive = false`.

---

## Prompt 2 — Banner Plus nella Home

Continua opad.me. Aggiungi un **banner promozionale Plus** nella Home per utenti che non hanno Plus attivo.

**Dove appare:** nella pagina Home (`Index.tsx`), in fondo alla lista delle attività, sopra la bottom nav. Visibile solo se `isPlusActive === false` e se l'utente non l'ha chiuso negli ultimi 7 giorni.

**Contenuto del banner:**
- Icona: `Sparkles` (Lucide)
- Titolo IT: `"Osserva di più con Plus"` — EN: `"Observe more with Plus"`
- Sottotitolo IT: `"Sblocca schede, temi e riduzione abitudini"` — EN: `"Unlock cards, themes and habit reduction"`
- CTA IT: `"Scopri Plus"` — EN: `"Discover Plus"` → naviga a `/plus`
- Bottone X per chiudere (angolo in alto a destra)

**Stile:** card con background card surface, bordo sottile accent tenue, rounded-xl. Non deve essere invadente — tono neutro, coerente col brand.

**Behavior dismiss:**
- Tap sulla X → banner scompare con fade-out
- Salva in localStorage: `plus_banner_dismissed_at` con timestamp corrente
- Al prossimo caricamento: se meno di 7 giorni dal dismiss → banner nascosto, altrimenti riappare
- Animazione entrata: slide-up + fade-in

Aggiungi le chiavi di traduzione necessarie a `translations.ts`.

---

## Prompt 3 — Pagina /plus

Continua opad.me. Crea una **pagina dedicata** per presentare l'abbonamento Plus.

**Route:** `/plus` — aggiungila in `App.tsx` dentro le route protette (con `AppLayout`).

**Contenuto:**
- Header con back button + titolo `"Plus"`
- Titolo grande: `"opad.me Plus"`
- Sottotitolo IT: `"Estendi la tua osservazione con strumenti avanzati."` — EN: `"Extend your observation with advanced tools."`

Tre card feature (background card surface, rounded-xl, con icona + titolo + descrizione):

1. **Schede** — icona `LayoutGrid`, IT: `"Palestra, Finanze e moduli futuri."` — EN: `"Gym, Finance and future modules."`
2. **Riduzione abitudini** — icona `TrendingDown`, IT: `"Traccia la riduzione giornaliera di ciò che vuoi osservare."` — EN: `"Track the daily reduction of what you want to observe."`
3. **Temi** — icona `Palette`, IT: `"Ocean, Sunset, Forest — personalizza l'aspetto."` — EN: `"Ocean, Sunset, Forest — customize the look."`

**CTA primario** (full-width, accent background): IT `"Attiva Plus"` — EN `"Activate Plus"`.
Per ora il tap mostra un messaggio inline sotto il bottone: IT `"Disponibile a breve."` — EN `"Coming soon."` (il pagamento non è ancora integrato).

**Link sotto il CTA:** IT `"Già Plus? Ripristina acquisto"` — EN `"Already Plus? Restore purchase"` — stessa logica placeholder.

**Se l'utente è già Plus:** il CTA è sostituito da un badge: IT `"Plus attivo"` — EN `"Plus active"` (colore accent). Le card feature restano visibili.

Aggiungi tutte le chiavi di traduzione necessarie.

---

## Prompt 4 — Gating schede (Attività + Settings + Nav)

Continua opad.me. Aggiungi il **gating Plus** alle schede — in Attività, Settings e nella navigazione.

**In Attività (`CardEntryPoints`):**

Se `isPlusActive === false`:
- Ogni card scheda mostra un badge `"Plus"` accanto al nome (piccolo, colore accent)
- Tap sulla card → il bottom sheet (Drawer) mostra la stessa anteprima (icona, nome, descrizione) ma:
  - Aggiunge sotto la descrizione: IT `"Questa scheda è disponibile con Plus."` — EN `"This card is available with Plus."` (testo secondario)
  - Il bottone CTA cambia da `"Apri"` a `"Scopri Plus"` → naviga a `/plus`
- Il badge stato (Configurata / Da configurare) non appare — sostituito dal badge Plus

Se `isPlusActive === true`: nessun cambiamento, funziona come ora.

**In Settings (sezione Schede):**

Se `isPlusActive === false`:
- Il toggle delle schede è disabilitato (opacity ridotta, non interattivo)
- Accanto al label appare badge `"Plus"` (colore accent)
- Sotto la sezione: link testuale `"Scopri Plus"` → `/plus`

Se `isPlusActive === true`: toggle funzionante come ora.

**Nella navigazione (`useNavConfig`):**

La tab Cards nella bottom nav deve essere visibile solo se `isPlusActive === true` E ci sono schede abilitate. Attualmente la condizione è solo `anyCardEnabled` — aggiungi `isPlusActive`.

Aggiungi le chiavi di traduzione necessarie.

---

## Prompt 5 — Gating tracking riduzione

Continua opad.me. Aggiungi il **gating Plus** al tracking `quantity_reduce` — le aree di riduzione funzionano in modalità binaria per utenti free.

**IMPORTANTE — refactoring:** Attualmente in `ActivityCard.tsx` il `QuantityCounter` appare se `anyCardEnabled && area.tracking_mode === "quantity_reduce"`. Questo accoppiamento va rimosso. La condizione deve diventare `isPlusActive && area.tracking_mode === "quantity_reduce"`. Stessa modifica nella pagina `Activities.tsx` dove si mostra la quantità odierna.

**Nella Home:**

Se `isPlusActive === false` e l'area ha `tracking_mode = 'quantity_reduce'`:
- Il `QuantityCounter` non viene mostrato
- Al suo posto appare il check-in binario standard (bottone done, come le aree binary)
- Sotto il nome area: badge inline IT `"Tracciamento avanzato con Plus"` — EN `"Advanced tracking with Plus"` (testo piccolo, colore accent)
- Tap sul badge → naviga a `/plus`

Se `isPlusActive === true`: QuantityCounter visibile come ora.

**Nella creazione area (`AreaForm`):**

Quando l'utente seleziona `tracking_mode = 'quantity_reduce'` e `isPlusActive === false`:
- Mostra un avviso inline sotto la selezione: IT `"Il tracciamento quantitativo è disponibile con Plus. L'area funzionerà in modalità standard."` — EN `"Quantitative tracking is available with Plus. The area will work in standard mode."`
- L'area viene creata comunque con `tracking_mode = 'quantity_reduce'` nel DB (i dati restano pronti per quando l'utente attiverà Plus)

Se `isPlusActive === true`: flusso completo senza avviso.

**Nell'Area Detail:**

Se `isPlusActive === false` e area `quantity_reduce`:
- Il grafico mostra il trend EWMA binario (non il grafico quantità)
- Badge inline: IT `"Grafico quantità disponibile con Plus"` — EN `"Quantity chart available with Plus"`

I dati `habit_quantity_daily` non vengono mai cancellati — tornano visibili quando Plus viene attivato.

Aggiungi le chiavi di traduzione necessarie.

---

## Prompt 6 — Gating temi extra

Continua opad.me. Aggiungi il **gating Plus** ai temi extra nella schermata Settings.

**In Settings (sezione palette colori):**

Il tema **Teal** resta sempre selezionabile. I temi **Ocean**, **Sunset**, **Forest**:

Se `isPlusActive === false`:
- Ogni cerchio/chip di palette extra mostra un piccolo badge `"Plus"` sotto il nome
- Tap → non cambia tema. Mostra un messaggio inline sotto il selettore: IT `"Disponibile con Plus"` — EN `"Available with Plus"` (testo secondario piccolo)
- Link accanto: `"Scopri Plus"` → `/plus`

Se `isPlusActive === true`: tutti i temi selezionabili senza restrizioni.

La selezione dark / light / system resta **sempre free**.

**Reset automatico:** se l'utente ha una palette extra attiva e `isPlusActive` diventa false (o all'avvio, se la palette salvata in localStorage non è `teal` e `isPlusActive === false`), il tema torna automaticamente a `teal`. Nessun messaggio.

Aggiungi le chiavi di traduzione necessarie.

---

## Prompt 7 — Voce Plus in Settings

Continua opad.me. Aggiungi una **voce "Plus"** nella sezione Account della schermata Settings.

**Posizione:** nella sezione Account, sopra il bottone "Esci / Sign out".

**Layout:** riga con label `"Plus"` a sinistra, badge di stato + chevron a destra. Tap → naviga a `/plus`.

**Badge stato:**
- Se `isPlusActive === false`: IT `"Non attivo"` — EN `"Not active"` (testo secondario)
- Se `isPlusActive === true`: IT `"Attivo"` — EN `"Active"` (colore accent)

Tap naviga a `/plus` sia per utenti free che Plus.

Aggiungi le chiavi di traduzione necessarie.
