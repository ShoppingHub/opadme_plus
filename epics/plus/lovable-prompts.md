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

## Prompt 2b — Banner Plus fisso (variante esame)

> **Variante alternativa del Prompt 2.** Banner non dismissable — usare al posto del Prompt 2 se si vuole il banner sempre visibile.

Continua opad.me. Aggiungi un **banner promozionale Plus** fisso nella Home per utenti che non hanno Plus attivo.

**Dove appare:** nella pagina Home (`Index.tsx`), in fondo alla lista delle attività, sopra la bottom nav. Visibile **sempre** se `isPlusActive === false`. Non è possibile chiuderlo — scompare solo quando l'utente attiva Plus.

**Contenuto del banner:**
- Icona: `Sparkles` (Lucide)
- Titolo IT: `"Osserva di più con Plus"` — EN: `"Observe more with Plus"`
- Sottotitolo IT: `"Sblocca schede, temi e riduzione abitudini"` — EN: `"Unlock cards, themes and habit reduction"`
- CTA IT: `"Scopri Plus"` — EN: `"Discover Plus"` → naviga a `/plus`

Nessun bottone X, nessuna logica di dismiss, nessun localStorage.

**Stile:** card con background card surface, bordo sottile accent tenue, rounded-xl. Tono neutro, coerente col brand.

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

**CTA primario** (full-width, accent background): IT `"Attiva Plus — 2,50 €/mese"` — EN `"Activate Plus — €2.50/month"`.
Tap → chiama la Edge Function `create-checkout-session` (vedi Prompt 8) e redirige a Stripe Checkout.

**Campo codice sconto** (opzionale, sopra il CTA): un campo di testo con placeholder IT `"Codice sconto"` — EN `"Discount code"`. Se compilato, il codice viene passato alla Edge Function che lo applica come `discounts` nella sessione Stripe.

**Link sotto il CTA:** IT `"Già Plus? Ripristina acquisto"` — EN `"Already Plus? Restore purchase"` — verifica lo stato dell'abbonamento Stripe e aggiorna `plus_active` di conseguenza.

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

---

## Prompt 8 — Pagamento Stripe Checkout

Continua opad.me. Integra **Stripe Checkout** (hosted) per attivare l'abbonamento Plus.

**Piano:** abbonamento mensile ricorrente, **2,50 €/mese**.

### Edge Function `create-checkout-session`

Crea una Supabase Edge Function `create-checkout-session` che:

1. Riceve `{ userId, email, promoCode? }` dal frontend (POST, autenticato)
2. Crea una sessione Stripe Checkout (`stripe.checkout.sessions.create`) con:
   - `mode: 'subscription'`
   - `line_items`: 1 item con il Price ID dell'abbonamento Plus (salvato in env var `STRIPE_PLUS_PRICE_ID`)
   - `customer_email`: email dell'utente
   - `client_reference_id`: userId
   - `success_url`: `{APP_URL}/plus?success=true`
   - `cancel_url`: `{APP_URL}/plus?canceled=true`
   - Se `promoCode` è presente: `discounts: [{ promotion_code: promoCode }]` — dove `promoCode` è l'ID della promotion code Stripe (il frontend manda il codice testuale, la Edge Function cerca la promotion code attiva con `stripe.promotionCodes.list({ code: promoCode, active: true })` e usa il primo risultato)
3. Ritorna `{ url }` — l'URL della sessione Stripe Checkout

**Env vars necessarie** (da configurare in Supabase Dashboard > Edge Functions > Secrets):
- `STRIPE_SECRET_KEY` — chiave segreta Stripe
- `STRIPE_PLUS_PRICE_ID` — ID del Price per l'abbonamento Plus mensile
- `APP_URL` — URL base dell'app (es. `https://opad.me`)

### Edge Function `stripe-webhook`

Crea una Supabase Edge Function `stripe-webhook` che:

1. Riceve i webhook Stripe (POST, verifica firma con `STRIPE_WEBHOOK_SECRET`)
2. Gestisce questi eventi:
   - `checkout.session.completed` → setta `plus_active = true`, `plus_activated_at = now()`, `plus_provider = 'stripe'` per l'utente con `id = client_reference_id`
   - `customer.subscription.deleted` → setta `plus_active = false`, `plus_expires_at = now()` per l'utente associato al customer Stripe
   - `invoice.payment_failed` → (opzionale) logga l'evento, non disattiva subito

**Env vars:** `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`

### Frontend — Pagina `/plus`

Aggiorna la pagina `/plus`:

1. Il CTA `"Attiva Plus"` al tap:
   - Se presente un codice nel campo sconto, lo include nella request
   - Chiama `create-checkout-session` passando userId, email e eventuale promoCode
   - Redirige a `response.url` (Stripe Checkout)
   - Durante il caricamento: bottone in stato loading (spinner + testo IT `"Reindirizzamento..."` — EN `"Redirecting..."`)

2. Al ritorno da Stripe:
   - Se URL contiene `?success=true`: mostra messaggio di conferma IT `"Plus attivato! Benvenuto."` — EN `"Plus activated! Welcome."` (badge accent, auto-dismiss dopo 5s). Ricarica lo stato Plus dal DB.
   - Se URL contiene `?canceled=true`: mostra messaggio neutro IT `"Pagamento annullato."` — EN `"Payment canceled."` (auto-dismiss dopo 3s)

### Setup Stripe (manuale, non nel prompt)

> **Nota per lo sviluppatore:** Prima di testare, creare su Stripe Dashboard:
> 1. Un **Product** "opad.me Plus" con un **Price** di €2,50/mese (recurring, monthly)
> 2. Un **Coupon** del 100% → creare una **Promotion Code** con codice `MCAI2026`
> 3. Configurare il **Webhook endpoint** puntando alla Edge Function `stripe-webhook` con gli eventi: `checkout.session.completed`, `customer.subscription.deleted`, `invoice.payment_failed`
> 4. Salvare le env vars nelle Secrets di Supabase

Aggiungi le chiavi di traduzione necessarie.
