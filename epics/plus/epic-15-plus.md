# Epic 15 — Plus (Abbonamento Premium)

## Obiettivo

Introdurre un livello di abbonamento **Plus** che sblocca funzionalità avanzate dell'app, mantenendo un'esperienza base completa e funzionale per gli utenti free. Il modello è freemium: l'app è utile senza pagare, Plus la rende più ricca.

---

## Concetto

```
Free                                Plus
─────────────────                   ─────────────────
Check-in binario (Fatto)            Tutto il Free +
Aree standard (illimitate)          Schede (Gym, Finance, future)
Home + Attività + Progress          Tracking quantity_reduce (riduzione)
Impostazioni                        Temi extra (Ocean, Sunset, Forest)
Tema Teal (default)
Grafico traiettoria base
```

L'utente free ha accesso completo al **core dell'osservazione**: aree, check-in binario, grafici traiettoria, Progress aggregato. Le funzionalità Plus estendono l'osservazione con strumenti specialistici e personalizzazione estetica.

---

## Funzionalità Plus

| Feature | Descrizione | Epic collegato |
|---|---|---|
| **Schede** | Tutti i moduli specialistici (Gym Card, Finance Projection, future) | Epic 14 |
| **Tracking riduzione** | Il `tracking_mode: quantity_reduce` per aree come fumo, alcol, social | Epic 03/05 |
| **Temi extra** | Palette Ocean, Sunset, Forest (Teal resta free) | Epic 07 |

### Dettaglio: Schede (Plus)

- L'entry point delle schede in Attività mostra un **badge Plus** (icona lucchetto o label "Plus")
- Tap su una scheda bloccata → bottom sheet con anteprima + CTA upgrade
- In Settings, la sezione Schede mostra i toggle disabilitati con badge Plus
- Con Plus attivo: comportamento identico a Epic 14 (tutto sbloccato)

### Dettaglio: Tracking riduzione (Plus)

- Quando l'utente crea un'area e seleziona `tracking_mode: quantity_reduce` → avviso che è una feature Plus
- Nella Home, il `QuantityCounter` (–/+1) è visibile solo per utenti Plus
- Per utenti free: l'area esiste ma funziona come check-in binario (Fatto/Non fatto)
- Con Plus attivo: counter inline nella Home, grafico quantità nell'Area Detail

### Dettaglio: Temi extra (Plus)

- In Settings, le palette Ocean, Sunset, Forest mostrano un badge Plus
- L'utente free vede l'anteprima del tema ma non può attivarlo
- Tap su un tema bloccato → CTA upgrade inline
- Teal + dark/light/system restano free

---

## Banner Plus nella Home

Quando l'utente **non ha Plus attivo**, nella Home appare un **banner in basso** (sopra la bottom nav), con call to action per l'upgrade.

### Comportamento

| Proprietà | Valore |
|---|---|
| Posizione | Fisso in basso nella Home, sopra la bottom nav (56px) |
| Visibilità | Solo nella Home, solo per utenti free |
| Dismissable | Sì — l'utente può chiuderlo con un tap sulla X |
| Riapparizione | Il banner riappare dopo **7 giorni** dalla chiusura |
| Persistenza dismiss | Salvata in `localStorage` con timestamp |

### Layout

```
┌─────────────────────────────────────┐
│  ✦  Osserva di più con Plus    [X] │
│      Sblocca schede, temi e         │
│      riduzione abitudini            │
│      [Scopri Plus]                  │
└─────────────────────────────────────┘
```

### Stile

| Proprietà | Valore |
|---|---|
| Background | `bg-[#1F4A50]` con bordo `border border-[#7DA3A0]/30` |
| Border radius | `rounded-xl` |
| Margine | `mx-4 mb-3` (sopra la bottom nav) |
| Icona | `Sparkles` Lucide 16px `#7DA3A0` (o simile, neutro) |
| Titolo | `text-[#EAEAEA] text-sm font-medium` |
| Sottotitolo | `text-[#B9C0C1] text-xs` |
| CTA | `text-[#7DA3A0] text-sm font-medium` → naviga a pagina Plus |
| X chiusura | `text-[#B9C0C1]` 16px, angolo in alto a destra |

> Il banner non deve essere invadente. Tono osservativo, non promozionale. Niente colori brillanti, niente urgenza.

---

## Pagina Plus

Route: `/plus`

Una pagina dedicata che presenta le funzionalità Plus e permette l'attivazione.

### Layout

```
┌─────────────────────────────┐
│ ← Plus                      │  ← Header con back button
├─────────────────────────────┤
│                             │
│  ✦ opad.me Plus             │  ← Titolo
│                             │
│  Estendi la tua osservazione│  ← Sottotitolo
│  con strumenti avanzati.    │
│                             │
│  ┌───────────────────────┐  │
│  │ 🏋️ Schede             │  │  ← Feature card 1
│  │ Palestra, Finanze     │  │
│  │ e moduli futuri       │  │
│  └───────────────────────┘  │
│                             │
│  ┌───────────────────────┐  │
│  │ 📉 Riduzione abitudini│  │  ← Feature card 2
│  │ Traccia la riduzione  │  │
│  │ giornaliera           │  │
│  └───────────────────────┘  │
│                             │
│  ┌───────────────────────┐  │
│  │ 🎨 Temi               │  │  ← Feature card 3
│  │ Ocean, Sunset, Forest │  │
│  └───────────────────────┘  │
│                             │
│  [Attiva Plus]              │  ← CTA primario
│                             │
│  Già Plus? [Ripristina]     │  ← Link restore purchase
│                             │
└─────────────────────────────┘
```

### Copy UI pagina Plus

| Elemento | IT | EN |
|---|---|---|
| Header | `"Plus"` | `"Plus"` |
| Titolo | `"opad.me Plus"` | `"opad.me Plus"` |
| Sottotitolo | `"Estendi la tua osservazione con strumenti avanzati."` | `"Extend your observation with advanced tools."` |
| Feature 1 titolo | `"Schede"` | `"Cards"` |
| Feature 1 desc | `"Palestra, Finanze e moduli futuri."` | `"Gym, Finance and future modules."` |
| Feature 2 titolo | `"Riduzione abitudini"` | `"Habit reduction"` |
| Feature 2 desc | `"Traccia la riduzione giornaliera di ciò che vuoi osservare."` | `"Track the daily reduction of what you want to observe."` |
| Feature 3 titolo | `"Temi"` | `"Themes"` |
| Feature 3 desc | `"Ocean, Sunset, Forest — personalizza l'aspetto."` | `"Ocean, Sunset, Forest — customize the look."` |
| CTA primario | `"Attiva Plus"` | `"Activate Plus"` |
| Restore | `"Già Plus? Ripristina acquisto"` | `"Already Plus? Restore purchase"` |

> La pagina è informativa. Il CTA "Attiva Plus" connette al sistema di pagamento (fuori scope MVP — vedi note tecniche).

---

## Stato Abbonamento

### Database

Colonne aggiunte alla tabella `users`:

```
plus_active          BOOLEAN    DEFAULT false
plus_activated_at    TIMESTAMPTZ  nullable
plus_expires_at      TIMESTAMPTZ  nullable
plus_provider        TEXT         nullable  CHECK (plus_provider IN ('stripe', 'apple', 'google', 'manual'))
```

> Per l'MVP, l'attivazione può essere manuale (`plus_provider = 'manual'`). L'integrazione con provider di pagamento (Stripe, App Store, Play Store) è fuori scope iniziale ma la struttura è pronta.

### RLS

- L'utente può leggere il proprio stato `plus_active`
- Solo il backend (edge function / service role) può modificare `plus_active`

### Hook di verifica

Ovunque una feature Plus venga richiesta, il controllo è:

```
IF user.plus_active = true → accesso concesso
ELSE → mostra badge Plus / CTA upgrade
```

---

## Impatto sugli Epic Esistenti

### Epic 02 — Home

- Aggiunta del **banner Plus** in basso per utenti free
- Il `QuantityCounter` per aree `quantity_reduce` è visibile solo se Plus attivo
- Per utenti free con aree `quantity_reduce`: il counter è sostituito dal check-in binario standard

### Epic 03 / 05 — Check-in / Add-Edit Area

- La selezione `tracking_mode: quantity_reduce` durante la creazione area → mostra badge Plus
- L'area può essere creata comunque, ma il tracking funziona in modalità binaria senza Plus
- Con Plus attivo: funzionamento completo come da Epic 03/05

### Epic 07 — Settings

- Sezione Schede: toggle disabilitati con badge Plus per utenti free
- Sezione Temi: palette extra mostrano badge Plus
- Nuova voce: **"Plus"** nella sezione Account → link a `/plus`

### Epic 10 — Attività

- Entry point schede: badge Plus visibile se non attivo
- Tap su scheda bloccata → bottom sheet con CTA upgrade (al posto di "Apri")

### Epic 14 — Schede

- Le schede richiedono Plus per essere abilitate
- Il toggle in Settings è disabilitato senza Plus
- L'entry point in Attività mostra il badge Plus

---

## Gating UX — Principi

1. **Mai bloccare silenziosamente.** Se una feature è Plus, l'utente deve capire *perché* non è disponibile e *come* sbloccarla
2. **Mai nascondere.** Le feature Plus sono visibili (con badge), non nascoste. L'utente vede cosa potrebbe avere
3. **Mai interrompere.** Nessun popup modale che interrompe il flusso. Badge inline, CTA nel contesto
4. **Tono neutro.** "Disponibile con Plus" — non "Sblocca il potenziale!" o "Non perderti questa feature!"
5. **Degradazione elegante.** Le aree `quantity_reduce` funzionano comunque (come binarie). L'app resta funzionale

---

## Badge Plus

Un indicatore visivo consistente usato in tutta l'app per le feature Plus bloccate.

| Proprietà | Valore |
|---|---|
| Tipo | Testo `"Plus"` o icona `Sparkles` + testo |
| Colore testo | `#7DA3A0` |
| Dimensione | `text-xs` |
| Posizione | Inline, accanto all'elemento bloccato |

Varianti:
- **Badge solo testo:** `Plus` — usato nei toggle Settings, entry point schede
- **Badge con icona:** `✦ Plus` — usato nel banner Home, pagina Plus

---

## Stati UI

| Stato | Comportamento |
|---|---|
| Free, banner visibile | Banner in basso nella Home con CTA |
| Free, banner chiuso | Banner nascosto, riappare dopo 7 giorni |
| Free, tap su scheda | Bottom sheet con anteprima + CTA upgrade |
| Free, tap su tema bloccato | CTA upgrade inline |
| Free, area quantity_reduce | Counter nascosto, funziona come binario |
| Plus attivo | Banner nascosto, tutte le feature sbloccate |
| Plus scaduto | Torna a comportamento free (banner riappare) |

---

## Edge Case

- Utente attiva Plus → le schede precedentemente abilitate si sbloccano immediatamente
- Utente disattiva Plus (scadenza) → le schede restano in `user_cards` ma diventano inaccessibili; i dati `quantity_reduce` restano nel DB ma il counter non è visibile
- Utente free crea area `quantity_reduce` → l'area funziona in modalità binaria, i dati quantitativi non vengono persi se attiva Plus in futuro
- Utente Plus registra quantità → disattiva Plus → i dati storici restano nel grafico Area Detail (read-only), ma non può aggiungere nuovi dati quantitativi
- Banner dismiss → cambia dispositivo → il banner appare (localStorage è per dispositivo)
- Pagina `/plus` accessibile anche da utente Plus → mostra stato "Plus attivo" invece del CTA

---

## Acceptance Criteria

- [ ] La tabella `users` include `plus_active`, `plus_activated_at`, `plus_expires_at`, `plus_provider`
- [ ] Il banner Plus appare nella Home per utenti free
- [ ] Il banner è dismissable e riappare dopo 7 giorni
- [ ] Il banner non appare per utenti Plus
- [ ] La pagina `/plus` presenta le feature Plus con CTA
- [ ] Le schede (Epic 14) richiedono Plus per essere abilitate
- [ ] Il tracking `quantity_reduce` richiede Plus; senza Plus, l'area funziona in modalità binaria
- [ ] I temi Ocean, Sunset, Forest richiedono Plus
- [ ] Il tema Teal + dark/light/system resta free
- [ ] In Settings, le feature Plus bloccate mostrano un badge "Plus"
- [ ] In Attività, le schede bloccate mostrano un badge "Plus"
- [ ] Tap su feature Plus bloccata → CTA che porta a `/plus`
- [ ] L'app è completamente funzionale senza Plus (degradazione elegante)
- [ ] Il tono di tutti i copy Plus è neutro e osservativo

---

## Copy UI

| Elemento | IT | EN |
|---|---|---|
| Badge Plus | `"Plus"` | `"Plus"` |
| Banner titolo | `"Osserva di più con Plus"` | `"Observe more with Plus"` |
| Banner sottotitolo | `"Sblocca schede, temi e riduzione abitudini"` | `"Unlock cards, themes and habit reduction"` |
| Banner CTA | `"Scopri Plus"` | `"Discover Plus"` |
| CTA upgrade generico | `"Disponibile con Plus"` | `"Available with Plus"` |
| Bottom sheet scheda bloccata | `"Questa scheda è disponibile con Plus."` | `"This card is available with Plus."` |
| Bottom sheet CTA | `"Scopri Plus"` | `"Discover Plus"` |
| Settings label Plus | `"Plus"` | `"Plus"` |
| Settings stato attivo | `"Attivo"` | `"Active"` |
| Settings stato non attivo | `"Non attivo"` | `"Not active"` |
| Tema bloccato | `"Disponibile con Plus"` | `"Available with Plus"` |
| Area quantity_reduce (free) | `"Tracciamento avanzato con Plus"` | `"Advanced tracking with Plus"` |

---

## Note Tecniche

### Pagamento (fuori scope MVP)

Il CTA "Attiva Plus" nella pagina `/plus` sarà inizialmente un placeholder. Le opzioni per l'integrazione futura:

- **Web app (PWA):** Stripe Checkout / Stripe Billing
- **iOS wrapper:** In-App Purchase (Apple)
- **Android wrapper:** Google Play Billing

Per l'MVP, l'attivazione Plus è gestita manualmente dal backend (`plus_provider = 'manual'`).

### Performance

- Il controllo `plus_active` è letto dal profilo utente già caricato — nessuna query aggiuntiva
- Il banner dismiss usa `localStorage` — nessun round-trip al server
- Il gating delle feature è un semplice check booleano nel frontend

---

## Dipendenze

- **Epic 02** (Home) — banner Plus nella Home
- **Epic 03** (Check-in) — gating del `QuantityCounter`
- **Epic 05** (Add/Edit Area) — badge Plus nella creazione area `quantity_reduce`
- **Epic 07** (Settings) — badge Plus su schede e temi, voce Plus in Account
- **Epic 10** (Attività) — badge Plus su entry point schede
- **Epic 14** (Schede) — gating delle schede

---

## Stories

- `story-15-01` — Schema DB: colonne Plus nella tabella `users`
- `story-15-02` — Banner Plus nella Home (layout, dismiss, riapparizione 7gg)
- `story-15-03` — Pagina `/plus` (presentazione feature + CTA)
- `story-15-04` — Gating schede: badge Plus in Attività e Settings
- `story-15-05` — Gating tracking riduzione: fallback binario per utenti free
- `story-15-06` — Gating temi: badge Plus sulle palette extra in Settings
- `story-15-07` — Voce "Plus" nella sezione Account di Settings
