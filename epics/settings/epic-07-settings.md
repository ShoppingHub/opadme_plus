# Epic 07 — Settings

## Obiettivo
Offrire all'utente un'unica schermata minimale per gestire le preferenze dell'app e il proprio account, senza clutter e senza opzioni superflue.

---

## Behavior

La schermata Settings è intenzionalmente semplice. Contiene solo ciò che è strettamente necessario per l'MVP. Nessuna sezione espandibile, nessun accordion, nessuna lista di opzioni avanzate.

I salvataggi sono silenziosi — nessun toast, nessun messaggio di conferma per le azioni non distruttive.

---

## Impostazioni Disponibili

| Setting | Tipo | Default | Note |
|---|---|---|---|
| Lingua / Language | Segmented control IT/EN | Rilevata da browser | Epic 08 |
| Show trajectory score | Toggle | OFF | Mostra il punteggio numerico nell'Area Detail |
| Notifications | Toggle | ON | Logica push rinviata al post-MVP |
| Schede / Cards | Sezione con toggle | — | Abilita/disabilita moduli specialistici — Epic 14 |
| Account email | Testo (read-only) | — | Non modificabile |
| Sign out | Bottone | — | |
| Delete account | Bottone distruttivo | — | Richiede conferma |

---

## Flussi

### Selezione Lingua
1. L'utente tappa il segmented control `Italiano · English`
2. La UI si aggiorna immediatamente nella nuova lingua (senza reload)
3. La preferenza `language` viene salvata su Supabase (silenzioso)
4. Vedi Epic 08 per il comportamento completo

### Sezione Schede
1. L'utente vede la lista delle schede disponibili con un toggle per ciascuna
2. Attivando un toggle → la scheda viene abilitata in `user_cards` (INSERT/UPDATE)
3. L'entry point della scheda appare immediatamente nella pagina Attività
4. Disattivando → la scheda viene disabilitata, l'entry point scompare
5. Salvataggio silenzioso — vedi Epic 14 per i dettagli

### Toggle "Show trajectory score"
1. L'utente attiva il toggle
2. La preferenza `settings_score_visible` viene aggiornata su Supabase
3. Salvataggio silenzioso — nessun feedback visivo
4. Il punteggio numerico diventa visibile nell'Area Detail

### Toggle "Notifications"
1. L'utente attiva/disattiva il toggle
2. La preferenza `settings_notifications` viene aggiornata su Supabase
3. Salvataggio silenzioso
4. La logica di push notification è fuori scope MVP

### Sign Out
1. L'utente tappa "Sign out"
2. La sessione Supabase viene terminata
3. Redirect alla schermata di onboarding (Step 1 — magic link)

### Delete Account
1. L'utente tappa "Delete account"
2. Appare un modale di conferma con il copy specifico
3. L'utente tappa "Delete permanently"
4. Account e dati vengono eliminati (cascade su tutte le tabelle)
5. Redirect alla schermata iniziale non autenticata
6. Nessun messaggio di addio

---

## Layout

```
┌─────────────────────────────┐
│ Impostazioni / Settings     │  ← Header
├─────────────────────────────┤
│                             │
│  Preferenze                 │  ← Label sezione
│  Lingua  [Italiano│English] │  ← Segmented control
│  Punteggio traiettoria  [○] │  ← Toggle (default OFF)
│  Notifiche              [●] │  ← Toggle (default ON)
│                             │
│  Schede / Cards             │  ← Label sezione
│  🏋️ Scheda Palestra    [○]  │  ← Toggle per scheda (Epic 14)
│  📈 Proiezione Finanze [○]  │
│                             │
│  Account                    │  ← Label sezione
│  user@email.com             │  ← Email read-only, text-[#B9C0C1]
│                             │
│  [Esci / Sign out]          │  ← Bottone standard
│                             │
│  [Elimina account /         │  ← Bottone distruttivo
│   Delete account]           │
│                             │
├─────────────────────────────┤
│  [Nav]                      │
└─────────────────────────────┘
```

---

## Modale Delete Account

```
Titolo:  "Delete account"
Corpo:   "This will permanently delete all your observation history.
          This cannot be undone."
CTA:     "Delete permanently"  ← text-[#E24A4A]
Cancel:  "Cancel"              ← testo neutro, chiude il modale
```

> ⚠️ Non usare "Are you sure you want to leave?" o altre frasi generiche.

---

## Stati UI

| Stato | Comportamento |
|---|---|
| Toggle OFF | Stile inattivo standard shadcn |
| Toggle ON | Stile attivo con colore `#7DA3A0` |
| Salvataggio toggle | Silenzioso, nessun feedback |
| Sign out loading | Bottone opacity-50 durante la chiamata |
| Modale delete | Overlay semi-trasparente su background `#0F2F33` |
| Delete in corso | CTA "Delete permanently" opacity-50 + spinner |

---

## Edge Case

- Utente tappa "Delete permanently" e la connessione cade → errore non bloccante, nessuna azione eseguita
- Toggle notifications attivato → nessuna push notification reale (fuori scope MVP)
- Email molto lunga → troncata con `text-ellipsis overflow-hidden`

---

## Database

Colonne richieste nella tabella `users`:
```
language           TEXT     DEFAULT 'en'   CHECK (language IN ('it', 'en'))
```

> `extra_tab_enabled` è stata rimossa — sostituita dalla tabella `user_cards` (vedi Epic 14).
> `menu_custom_items` (TEXT[]) è deprecata — da rimuovere in migrazione (vedi Epic 09, story-09-04).

Tabella aggiuntiva (vedi Epic 14):
```
user_cards → id, user_id, card_type, area_id (nullable), enabled, created_at
```

---

## Acceptance Criteria

- [x] Il selettore lingua appare come primo elemento della sezione Preferenze
- [x] Il cambio lingua aggiorna la UI immediatamente e salva `language` su Supabase
- [x] Il toggle "Show trajectory score" aggiorna `settings_score_visible` su Supabase
- [x] Il salvataggio dei toggle è silenzioso (nessun toast)
- [ ] La sezione Schede mostra un toggle per ogni scheda disponibile (Epic 14)
- [ ] L'attivazione di un toggle scheda inserisce/aggiorna un record in `user_cards`
- [ ] L'entry point della scheda appare/scompare immediatamente nella pagina Attività
- [x] "Esci / Sign out" termina la sessione e redirige al login
- [x] "Elimina account / Delete account" apre il modale con il copy esatto specificato
- [x] "Delete permanently" elimina account e dati con cascade
- [x] Il modale ha CTA distruttiva in `#E24A4A`
- [x] Dopo il delete, redirect alla schermata non autenticata senza messaggi
- [x] Tutti i label seguono la lingua selezionata

---

## Stato implementazione

**Completato** — commit su `ShoppingHub/project-spark`.

| Componente | File |
|---|---|
| Settings page | `src/pages/SettingsPage.tsx` — 3 sezioni (Preferenze, Menu, Account) |
| Menu config | `src/hooks/useMenuConfig.tsx` — load/save `menu_custom_items`, rilevamento gym, Supabase realtime listener |
| Migrazioni DB | `20260307164834` (language) + `20260307165629` (menu_custom_items) |

### Variazioni rispetto all'epic
- Label toggle score: epic `"Mostra punteggio"` → codice `"Mostra punteggio traiettoria"` (più descrittivo)
- Le voci fisse nella sezione Menu mostrano un'icona lucchetto + badge "fisso/fixed" (miglioramento UX)
- Messaggio "Max 2 custom items active" mostrato quando il limite è raggiunto (miglioramento UX)
- `useMenuConfig` include un Supabase realtime channel listener per auto-rimuovere "gym" se l'area viene eliminata

---

## Copy UI

| Elemento | IT | EN |
|---|---|---|
| Header | `"Impostazioni"` | `"Settings"` |
| Label sezione preferenze | `"Preferenze"` | `"Preferences"` |
| Label lingua | `"Lingua"` | `"Language"` |
| Label toggle score | `"Mostra punteggio"` | `"Show trajectory score"` |
| Label toggle notifiche | `"Notifiche"` | `"Notifications"` |
| Label sezione schede | `"Schede"` | `"Cards"` |
| Sub sezione schede | `"Moduli specialistici per le tue aree."` | `"Specialized modules for your areas."` |
| Label sezione account | `"Account"` | `"Account"` |
| Bottone sign out | `"Esci"` | `"Sign out"` |
| Bottone delete | `"Elimina account"` | `"Delete account"` |
| Titolo modale | `"Elimina account"` | `"Delete account"` |
| Corpo modale | `"Eliminerà definitivamente tutto il tuo storico. Questa azione è irreversibile."` | `"This will permanently delete all your observation history. This cannot be undone."` |
| CTA modale confirm | `"Elimina definitivamente"` | `"Delete permanently"` |
| CTA modale cancel | `"Annulla"` | `"Cancel"` |

---

## Note UI

- Il bottone "Delete account" ha colore testo `#E24A4A` (accent — uso consentito per azioni distruttive)
- Brand tokens: `brand-system/betonme_brand_system_lovable.md`
- Toggle usa shadcn/ui `<Switch>` con override colore active: `#7DA3A0`

---

## Dipendenze

- Epic 08 (i18n) — per la logica lingua e i label IT/EN
- Epic 14 (Schede) — la sezione Schede in Settings gestisce l'abilitazione dei moduli specialistici

---

## Stories

- `story-07-01` — Layout Settings con sezioni Preferenze e Account — **completata** *(da aggiornare: rimuovere sezione Menu)*
- `story-07-02` — Selettore lingua IT/EN con aggiornamento UI immediato — **completata**
- `story-07-03` — Logica toggle score e salvataggio su Supabase — **completata**
- `story-07-04` — Sezione Menu custom — **deprecata** *(rimossa con refactor nav)*
- `story-07-05` — Sign out con redirect — **completata**
- `story-07-06` — Delete account con modale di conferma e cascade — **completata**
- `story-07-07` — ~~Toggle "Mostra sezione Finanze"~~ → **sostituita** dalla sezione Schede generica (vedi Epic 14, story-14-06)
