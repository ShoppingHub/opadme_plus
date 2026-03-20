# Stories — Epic 15 — Plus (Abbonamento Premium)

## Sequenza di implementazione

```
story-15-01 → Schema DB: colonne Plus nella tabella users                          ⏳ da fare
story-15-02 → Banner Plus nella Home (layout, dismiss, riapparizione 7gg)          ⏳ da fare
story-15-03 → Pagina /plus (presentazione feature + CTA)                           ⏳ da fare
story-15-04 → Gating schede: badge Plus in Attività e Settings                     ⏳ da fare
story-15-05 → Gating tracking riduzione: fallback binario per utenti free          ⏳ da fare
story-15-06 → Gating temi: badge Plus sulle palette extra in Settings              ⏳ da fare
story-15-07 → Voce "Plus" nella sezione Account di Settings                        ⏳ da fare
```

> **Dipendenze:** story-15-01 è prerequisito per tutte le altre. story-15-04 richiede Epic 14 (Schede). story-15-05 richiede Epic 03/05 (Check-in / Add-Edit Area). story-15-06 richiede story-07-08 (Theme system). story-15-07 richiede story-15-03.

---

## story-15-01 — Schema DB: colonne Plus nella tabella users

opad.me è un'app di osservazione del benessere. Aggiungi le colonne per gestire lo stato dell'abbonamento Plus nella tabella `users`.

**Colonne da aggiungere a `users`:**

| Colonna | Tipo | Default | Note |
|---|---|---|---|
| `plus_active` | BOOLEAN | `false` | Stato corrente dell'abbonamento |
| `plus_activated_at` | TIMESTAMPTZ | NULL | Data di prima attivazione |
| `plus_expires_at` | TIMESTAMPTZ | NULL | Data di scadenza (NULL = nessuna scadenza) |
| `plus_provider` | TEXT | NULL | CHECK IN ('stripe', 'apple', 'google', 'manual') |

**RLS:**
- SELECT: l'utente può leggere il proprio `plus_active`
- UPDATE su colonne Plus: solo `service_role` (edge function / backend)
- L'utente NON può modificare il proprio stato Plus dal frontend

**Hook `usePlusStatus`:**

Espone:
- `isPlusActive`: booleano — `user.plus_active === true`
- `plusExpiresAt`: data di scadenza (o null)
- `isFeatureLocked(feature)`: check rapido per feature specifiche

```typescript
type PlusFeature = 'cards' | 'quantity_reduce' | 'themes_extra'

function isFeatureLocked(feature: PlusFeature): boolean {
  return !isPlusActive
}
```

> Per l'MVP tutte le feature Plus sono bloccate/sbloccate insieme. In futuro si potrebbe differenziare per tier.

---

## story-15-02 — Banner Plus nella Home

Continua Epic 15 di opad.me. Aggiungi un banner promozionale Plus nella Home per utenti che non hanno l'abbonamento attivo.

**Dove appare:**
- Nella Home (Tab 1), in fondo al contenuto, sopra la bottom nav
- Solo se `user.plus_active === false`
- Solo se non è stato chiuso dall'utente negli ultimi 7 giorni

**Layout del banner:**

```
┌─────────────────────────────────────┐
│  ✦  Osserva di più con Plus    [X] │
│      Sblocca schede, temi e         │
│      riduzione abitudini            │
│      [Scopri Plus]                  │
└─────────────────────────────────────┘
```

**Stile:**
- Background: `bg-[#1F4A50]` con `border border-[#7DA3A0]/30 rounded-xl`
- Margini: `mx-4 mb-3` (sopra la bottom nav)
- Icona: `Sparkles` Lucide 16px `#7DA3A0`
- Titolo: `"Osserva di più con Plus"` (IT) / `"Observe more with Plus"` (EN) — `text-[#EAEAEA] text-sm font-medium`
- Sottotitolo: `"Sblocca schede, temi e riduzione abitudini"` (IT) / `"Unlock cards, themes and habit reduction"` (EN) — `text-[#B9C0C1] text-xs`
- CTA: `"Scopri Plus"` (IT) / `"Discover Plus"` (EN) — `text-[#7DA3A0] text-sm font-medium` → naviga a `/plus`
- X chiusura: `X` icon 16px `text-[#B9C0C1]`, angolo in alto a destra

**Behavior dismiss:**
- Tap sulla X → il banner scompare con animazione fade-out
- Salva in `localStorage`: `plus_banner_dismissed_at = Date.now()`
- Al prossimo caricamento della Home:
  - Se `Date.now() - plus_banner_dismissed_at < 7 * 24 * 60 * 60 * 1000` → banner nascosto
  - Altrimenti → banner visibile di nuovo

**Animazione:**
- Entrata: slide-up + fade-in, `300ms ease-in-out`
- Uscita (dismiss): fade-out, `200ms ease-in-out`

**Non mostrare il banner se:**
- `user.plus_active === true`
- L'utente ha chiuso il banner meno di 7 giorni fa

---

## story-15-03 — Pagina /plus

Continua Epic 15 di opad.me. Crea la pagina dedicata alla presentazione dell'abbonamento Plus.

**Route:** `/plus`

**Header:** `"Plus"` + back button → torna alla pagina precedente

**Contenuto:**

1. **Titolo:** `"opad.me Plus"` — `text-[#EAEAEA] text-xl font-medium`
2. **Sottotitolo:** `"Estendi la tua osservazione con strumenti avanzati."` (IT) / `"Extend your observation with advanced tools."` (EN) — `text-[#B9C0C1] text-base`

3. **Feature card — Schede:**
   - Icona: `LayoutGrid` Lucide 24px `#7DA3A0`
   - Titolo: `"Schede"` (IT) / `"Cards"` (EN)
   - Desc: `"Palestra, Finanze e moduli futuri."` (IT) / `"Gym, Finance and future modules."` (EN)
   - Background: `bg-[#1F4A50] rounded-xl p-4`

4. **Feature card — Riduzione:**
   - Icona: `TrendingDown` Lucide 24px `#7DA3A0`
   - Titolo: `"Riduzione abitudini"` (IT) / `"Habit reduction"` (EN)
   - Desc: `"Traccia la riduzione giornaliera di ciò che vuoi osservare."` (IT) / `"Track the daily reduction of what you want to observe."` (EN)
   - Background: `bg-[#1F4A50] rounded-xl p-4`

5. **Feature card — Temi:**
   - Icona: `Palette` Lucide 24px `#7DA3A0`
   - Titolo: `"Temi"` (IT) / `"Themes"` (EN)
   - Desc: `"Ocean, Sunset, Forest — personalizza l'aspetto."` (IT) / `"Ocean, Sunset, Forest — customize the look."` (EN)
   - Background: `bg-[#1F4A50] rounded-xl p-4`

6. **CTA primario:** `"Attiva Plus"` (IT) / `"Activate Plus"` (EN) — `bg-[#7DA3A0] text-[#0F2F33] rounded-xl min-h-[44px] w-full font-medium`
   - Per MVP: il tap mostra un messaggio inline `"Disponibile a breve."` (IT) / `"Coming soon."` (EN) — il pagamento non è ancora integrato
   - In futuro: connette a Stripe Checkout o In-App Purchase

7. **Link restore:** `"Già Plus? Ripristina acquisto"` (IT) / `"Already Plus? Restore purchase"` (EN) — `text-[#B9C0C1] text-sm text-center`
   - Per MVP: stessa logica del CTA (placeholder)

**Se l'utente è già Plus:**
- Il CTA è sostituito da un badge: `"Plus attivo"` (IT) / `"Plus active"` (EN) — `text-[#7DA3A0] text-sm`
- Le feature card restano visibili (conferma di cosa include Plus)

---

## story-15-04 — Gating schede: badge Plus in Attività e Settings

Continua Epic 15 di opad.me. Aggiungi il gating Plus alle schede (Epic 14) — sia negli entry point in Attività che nei toggle in Settings.

**Dipende da:** story-15-01 (hook `usePlusStatus`) + Epic 14 (Schede)

**In Attività (entry point schede):**

- Se `isPlusActive === false`:
  - La card scheda mostra un badge `"Plus"` accanto al nome — `text-[#7DA3A0] text-xs`
  - Tap sulla card → bottom sheet modificato:
    - Stessa anteprima (icona, nome, descrizione)
    - Testo aggiuntivo: `"Questa scheda è disponibile con Plus."` (IT) / `"This card is available with Plus."` (EN) — `text-[#B9C0C1] text-sm`
    - CTA cambia da `"Apri"` a `"Scopri Plus"` (IT) / `"Discover Plus"` (EN) → naviga a `/plus`
  - Il badge stato (`"Configurata"` / `"Da configurare"`) non appare — sostituito dal badge Plus

- Se `isPlusActive === true`:
  - Comportamento identico a Epic 14 (nessun badge Plus, bottom sheet con "Apri")

**In Settings (sezione Schede):**

- Se `isPlusActive === false`:
  - I toggle delle schede sono **disabilitati** (opacity-50, non interattivi)
  - Accanto a ogni nome scheda: badge `"Plus"` — `text-[#7DA3A0] text-xs`
  - Sotto la sezione: link `"Scopri Plus"` → `/plus`

- Se `isPlusActive === true`:
  - Toggle funzionanti come da Epic 14

---

## story-15-05 — Gating tracking riduzione: fallback binario

Continua Epic 15 di opad.me. Aggiungi il gating Plus al tracking `quantity_reduce` — le aree di riduzione funzionano in modalità binaria per utenti free.

**Dipende da:** story-15-01 (hook `usePlusStatus`) + Epic 03/05 (Check-in / Add-Edit Area)

**Nella Home (check-in):**

- Se `isPlusActive === false` e l'area ha `tracking_mode = 'quantity_reduce'`:
  - Il `QuantityCounter` (–/+1) **non** viene mostrato
  - Al suo posto appare il check-in binario standard: `"Fatto"` / `"Done"`
  - Un badge inline sotto la CTA: `"Tracciamento avanzato con Plus"` (IT) / `"Advanced tracking with Plus"` (EN) — `text-[#7DA3A0] text-xs`
  - Tap sul badge → naviga a `/plus`

- Se `isPlusActive === true`:
  - Comportamento completo: `QuantityCounter` visibile, tracking quantitativo attivo

**Nella creazione area (Add Area, Epic 05):**

- Quando l'utente seleziona `tracking_mode = 'quantity_reduce'`:
  - Se `isPlusActive === false`:
    - Mostra un avviso inline sotto la selezione: `"Il tracciamento quantitativo è disponibile con Plus. L'area funzionerà in modalità standard."` (IT) / `"Quantitative tracking is available with Plus. The area will work in standard mode."` (EN)
    - L'area viene creata comunque con `tracking_mode = 'quantity_reduce'` nel DB (i dati sono pronti per quando l'utente attiverà Plus)
    - Ma nella Home funzionerà come binaria finché Plus non è attivo

  - Se `isPlusActive === true`:
    - Flusso completo come da Epic 05

**Nell'Area Detail:**

- Se `isPlusActive === false` e area `quantity_reduce`:
  - Il grafico mostra il trend EWMA binario (non il grafico quantità)
  - Badge inline: `"Grafico quantità disponibile con Plus"` (IT) / `"Quantity chart available with Plus"` (EN)

- Se `isPlusActive === true`:
  - Grafico quantità giornaliera come da Epic 04

**Dati:**
- I dati `habit_quantity_daily` eventualmente registrati quando l'utente era Plus **non vengono cancellati** se Plus scade
- Tornano visibili se l'utente riattiva Plus

---

## story-15-06 — Gating temi: badge Plus sulle palette extra

Continua Epic 15 di opad.me. Aggiungi il gating Plus ai temi extra nella schermata Settings.

**Dipende da:** story-15-01 (hook `usePlusStatus`) + story-07-08 (Theme system)

**In Settings (sezione Temi):**

- Il tema **Teal** resta sempre selezionabile (free)
- I temi **Ocean**, **Sunset**, **Forest**:

  - Se `isPlusActive === false`:
    - La card/chip del tema mostra un badge `"Plus"` — `text-[#7DA3A0] text-xs`
    - Tap → non cambia tema, mostra un messaggio inline sotto il selettore:
      `"Disponibile con Plus"` (IT) / `"Available with Plus"` (EN) — `text-[#B9C0C1] text-xs`
    - Link: `"Scopri Plus"` → `/plus`

  - Se `isPlusActive === true`:
    - Tutti i temi selezionabili senza restrizioni

- La selezione **dark / light / system** resta **free** per tutte le palette

**Se l'utente aveva un tema Plus attivo e Plus scade:**
- Il tema torna automaticamente a Teal (mantenendo la modalità dark/light/system scelta)
- Nessun messaggio — il cambio è silenzioso

---

## story-15-07 — Voce "Plus" nella sezione Account di Settings

Continua Epic 15 di opad.me. Aggiungi una voce "Plus" nella sezione Account della schermata Settings.

**Dipende da:** story-15-03 (pagina `/plus`)

**Dove appare:**
- Nella sezione **Account** di Settings, sopra il bottone "Esci / Sign out"

**Layout:**

```
┌─────────────────────────────┐
│  Account                    │
│                             │
│  user@email.com             │
│                             │
│  Plus       [Non attivo] >  │  ← nuova voce
│                             │
│  [Esci / Sign out]          │
│  [Elimina account]          │
└─────────────────────────────┘
```

**Behavior:**
- Label: `"Plus"` — `text-[#EAEAEA]`
- Badge stato:
  - Se `isPlusActive === false`: `"Non attivo"` (IT) / `"Not active"` (EN) — `text-[#B9C0C1] text-sm`
  - Se `isPlusActive === true`: `"Attivo"` (IT) / `"Active"` (EN) — `text-[#7DA3A0] text-sm`
- Chevron destro → tap naviga a `/plus`
- Tap → naviga a `/plus` (sia per utenti free che Plus)
