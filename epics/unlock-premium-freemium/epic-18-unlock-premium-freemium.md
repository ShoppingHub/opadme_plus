# Epic 18 — Plus Always-On (Freemium Unlock)

## Obiettivo

Rendere **tutte le funzionalità Plus sempre attive** per tutti gli utenti, senza gating freemium. Eliminare ogni riferimento a paywall, badge di upgrade, banner promozionali e CTA a pagamento. Rendere le Schede sempre disponibili e rimuovere il loro toggle dalle Impostazioni.

Questa configurazione sospende temporaneamente il modello freemium (Epic 15) per permettere il test completo dell'app. La struttura DB rimane intatta: il ripristino del gating sarà possibile in futuro senza refactoring.

---

## Motivazione

L'app viene testata in produzione. Il gating freemium introduce attrito che ostacola la valutazione completa delle funzionalità. La scelta è di aprire tutto temporaneamente:

1. **Plus always-on:** `isPlusActive = true` per tutti (hardcoded, nessun DB check)
2. **UI pulita:** nessun badge, banner, CTA o voce che rimandi al piano Plus
3. **Schede sempre attive:** nessun toggle utente, le schede sono sempre disponibili e visibili in Attività

---

## Comportamento Target

| Funzionalità | Stato precedente (Epic 15) | Stato target |
|---|---|---|
| `isPlusActive` | Letto da `users.plus_active` | Sempre `true` (hardcoded) |
| Banner Plus in Home | Visibile per utenti free | Rimosso — non appare mai |
| Badge "Plus" in Settings, Attività | Visibile sulle feature bloccate | Rimosso completamente |
| CTA "Scopri Plus" / "Disponibile con Plus" | Presente ovunque | Rimosso completamente |
| Bottom sheet scheda bloccata | CTA upgrade | CTA "Apri" standard (come da Epic 14) |
| Pagina `/plus` | Accessibile | Non accessibile — route rimossa |
| Voce "Plus" in Settings Account | Presente con stato | Rimossa |
| Toggle Schede in Settings | Controllato dall'utente | Rimosso dalla UI |
| QuantityCounter (quantity_reduce) | Visibile solo con Plus | Sempre visibile |
| Temi Ocean / Sunset / Forest | Bloccati senza Plus | Selezionabili liberamente |
| Entry point schede in Attività | Badge Plus se non attivo | Entry point standard sempre |

---

## Modifiche per File

### `src/hooks/usePlusStatus.tsx`

Il hook `usePlusStatus` restituisce sempre `isPlusActive = true`, senza query al database:

```
isPlusActive       → sempre true
isFeatureLocked()  → sempre false (per qualsiasi PlusFeature)
plusExpiresAt      → null
```

Il tipo `PlusFeature = 'cards' | 'quantity_reduce' | 'themes_extra'` rimane per compatibilità con le chiamate esistenti. Nessuna query a `users.plus_active`.

Se il hook non esiste ancora (story-15-01 non implementata): crearlo direttamente in forma always-on.

---

### `src/components/home/PlusBanner.tsx`

Il banner non viene renderizzato nella Home, mai. Se è condizionato da `!isPlusActive`, sparisce automaticamente con story-18-01. Verificare che non persista in nessun caso e che non lasci spazio vuoto nel layout.

---

### `src/components/CardEntryPoints.tsx`

- Rimosso il badge "Plus" accanto al nome scheda
- Bottom sheet: CTA "Apri" / "Open" senza testo "Questa scheda è disponibile con Plus."
- Stato "Configurata" / "Da configurare" visibile come da Epic 14

---

### `src/pages/SettingsPage.tsx`

Tre modifiche distinte:

**1. Sezione Schede rimossa**
La sezione "Schede / Cards" con i toggle per ogni scheda viene rimossa dalla UI. Non appare in nessuna parte di Settings.

**2. Sezione Temi pulita**
Badge "Plus" rimosso dalle palette Ocean, Sunset, Forest. Tutti e 4 i temi sono selezionabili senza restrizioni. Rimosso il testo "Disponibile con Plus" e il link "Scopri Plus".

**3. Sezione Account pulita**
La voce "Plus" con badge stato ("Non attivo" / "Attivo") e chevron verso `/plus` viene rimossa. La sezione Account mostra: email + Esci + Elimina account.

---

### `src/hooks/useNavConfig.tsx`

Se la visibilità della tab Cards è condizionata da `isPlusActive`: con `isPlusActive` sempre `true`, la condizione si risolve automaticamente. Verificare che la tab sia visibile se `anyCardEnabled` (comportamento da Epic 14).

Rimuovere l'eventuale check `isPlusActive &&` dalla condizione — la visibilità dipende solo da `anyCardEnabled`.

---

### `src/pages/AreaForm.tsx`

Selezionando `tracking_mode = quantity_reduce`: rimosso l'avviso "Il tracciamento quantitativo è disponibile con Plus." Il flusso è identico per tutti i tipi di tracking.

---

### `src/pages/AreaDetail.tsx`

Rimosso il badge "Grafico quantità disponibile con Plus". Il grafico quantità per aree `quantity_reduce` è sempre visibile senza messaggi di upgrade.

---

### `src/components/home/ActivityCard.tsx`

Le condizioni `isQuantityReduce` e `isQuantityNoQuickAdd` non devono dipendere da `anyCardEnabled` né da `isPlusActive` (ora sempre true). Verificare che il QuantityCounter sia sempre visibile per le aree `quantity_reduce` con `show_quick_add_home = true`.

---

### Schede sempre attive — auto-enable

Le schede non hanno un toggle utente. Due opzioni (implementare la più semplice):

**Opzione A (preferita):** Le schede vengono mostrate sempre come entry point in Attività, senza filtrare per `user_cards.enabled`. Se l'utente non ha un record `user_cards` per una scheda, la scheda appare comunque con stato "Da configurare".

**Opzione B:** All'accesso utente, tutte le schede disponibili vengono inserite in `user_cards` con `enabled = true` se non esistono già (upsert silenzioso).

Il comportamento di configurazione (wizard, collegamento area, stato "Configurata" / "Da configurare") rimane invariato da Epic 14.

---

### Route `/plus` e pagina `PlusPage.tsx`

- Rimuovere la route `/plus` da `App.tsx`
- Rimuovere `src/pages/PlusPage.tsx` (se esiste)
- Rimuovere tutti i link a `/plus` presenti nell'app (banner, bottom sheet, CTA in Settings)

Se la pagina `/plus` non è ancora stata implementata (story-15-03 non completata): questa modifica non è necessaria.

---

## Cosa NON cambia

- Struttura DB: `users.plus_active`, `user_cards`, tabelle Plus — intatte, nessuna migrazione
- Comportamento schede da Epic 14: wizard configurazione, collegamento area, pagine dedicate `/cards/gym` e `/cards/finance`
- Sistema EWMA e grafico traiettoria
- Sezione Attività: le schede continuano ad apparire come entry point standard (senza badge Plus)
- Navigazione: 4 tab fisse invariate

---

## Acceptance Criteria

- [ ] Tutte le funzionalità Plus (schede, quantity_reduce, temi) accessibili senza abbonamento
- [ ] `isPlusActive` sempre `true` — verificato nel hook
- [ ] Nessun badge "Plus" visibile in nessuna parte dell'app
- [ ] Il banner Plus non appare mai nella Home
- [ ] Nessuna CTA "Scopri Plus" o "Disponibile con Plus" in nessuna schermata
- [ ] La voce "Plus" assente dalla sezione Account di Settings
- [ ] La sezione "Schede" assente da Settings (nessun toggle)
- [ ] Le schede appaiono sempre come entry point in Attività (senza badge Plus)
- [ ] Il QuantityCounter visibile per tutte le aree `quantity_reduce` senza restrizioni
- [ ] I temi Ocean, Sunset, Forest selezionabili liberamente in Settings
- [ ] La route `/plus` non è accessibile
- [ ] Struttura DB Plus intatta (nessuna migrazione di rimozione)

---

## Reversibilità

Per ripristinare il gating freemium in futuro:

1. `usePlusStatus.tsx` — ripristinare la lettura di `users.plus_active` dal DB (da story-15-01)
2. Ripristinare badge Plus e CTA dai componenti (reverting story-18-03)
3. Ripristinare il toggle Schede in Settings (reverting story-18-04)
4. Ripristinare route `/plus` e voce Account (reverting story-18-05)
5. Nessuna migrazione DB necessaria (il dato era e rimane intatto)

---

## Dipendenze

- **Epic 15** (Plus) — la feature sospesa; i file da modificare sono documentati nella sezione "Stato implementazione attuale" di epic-15-plus.md
- **Epic 14** (Schede) — il comportamento target delle schede sbloccate
- **Epic 07** (Settings) — le sezioni da modificare (Schede, Temi, Account)
- **Epic 02** (Home) — rimozione banner
- **Epic 10** (Attività) — entry point schede senza gating
- **Epic 05** (Add/Edit Area) — selezione tracking_mode senza badge Plus

---

## Stories

- `story-18-01` — `usePlusStatus` always-on: hardcode `isPlusActive = true`
- `story-18-02` — Rimozione banner Plus dalla Home
- `story-18-03` — Pulizia badge Plus e CTA upgrade (Settings temi, Attività, AreaForm, AreaDetail)
- `story-18-04` — Schede sempre attive: rimozione toggle da Settings, auto-enable
- `story-18-05` — Cleanup finale: voce Plus in Account, route `/plus`
