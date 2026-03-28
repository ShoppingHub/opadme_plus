# Stories — Epic 18 — Plus Always-On (Freemium Unlock)

## Sequenza di implementazione

```
story-18-01 → usePlusStatus always-on: hardcode isPlusActive = true          ⏳ da fare
story-18-02 → Rimozione banner Plus dalla Home                                ⏳ da fare
story-18-03 → Pulizia badge Plus e CTA upgrade ovunque                       ⏳ da fare
story-18-04 → Schede sempre attive: rimozione toggle da Settings              ⏳ da fare
story-18-05 → Cleanup finale: voce Plus in Account, route /plus               ⏳ da fare
```

> **Dipendenze:** story-18-01 è prerequisita per tutte le altre. Con `isPlusActive` sempre `true`, molti badge e CTA spariscono automaticamente — le story successive gestiscono i remnants UI e le modifiche strutturali.

> **Nota implementativa:** Le stories modificano componenti esistenti nel repository `ShoppingHub/joyous-beginning`. La sezione "Stato implementazione attuale" in `epics/plus/epic-15-plus.md` elenca i file specifici con i riferimenti alle linee di codice. Se alcune story di Epic 15 non sono ancora state implementate (es. story-15-03, story-15-07), le story corrispondenti di questo epic (18-05) non hanno effetto — verificare prima di procedere.

---

## story-18-01 — usePlusStatus always-on

opad.me è un'app mobile-first di osservazione del benessere personale. Stiamo disattivando temporaneamente il gating freemium: tutte le funzionalità Plus devono essere sempre attive per tutti gli utenti.

**Modifica il hook `usePlusStatus`** in `src/hooks/usePlusStatus.tsx`:

- `isPlusActive` deve restituire sempre `true`, senza leggere `users.plus_active` dal database
- `isFeatureLocked(feature)` deve restituire sempre `false` per qualsiasi feature (`'cards'`, `'quantity_reduce'`, `'themes_extra'`)
- `plusExpiresAt` deve restituire `null`

Se il hook non esiste ancora: crearlo con queste proprietà hardcoded. Nessuna query Supabase necessaria.

**Non modificare:** la struttura delle tabelle `users` o `user_cards`. Nessuna migrazione database.

---

## story-18-02 — Rimozione banner Plus dalla Home

Continua Epic 18 di opad.me. Con `isPlusActive` sempre `true` (story-18-01), il banner Plus non dovrebbe più apparire nella Home. Verifica e assicura che sia così.

**Verifica il componente `PlusBanner`** (`src/components/home/PlusBanner.tsx`) e il punto di rendering nella Home:

- Se il banner è condizionato da `!isPlusActive`: con story-18-01 applicata sparisce automaticamente — nessuna modifica necessaria
- Se il banner persiste per qualsiasi altro motivo: rimuovere il suo rendering dalla Home completamente

**Comportamento atteso:**
- Il banner non appare mai nella Home, per nessun utente
- Rimuovendo il banner, il contenuto della Home scorre normalmente fino alla bottom nav senza spazio vuoto residuo
- Il `localStorage` `plus_banner_dismissed_at` non viene rimosso attivamente (non è necessario)

---

## story-18-03 — Pulizia badge Plus e CTA upgrade

Continua Epic 18 di opad.me. Rimuovi tutti i badge "Plus" e le CTA di upgrade dall'app. Con `isPlusActive = true`, molti potrebbero già essere nascosti — questa story gestisce i remnants visivi e le dipendenze che non passano dal hook.

**`src/components/CardEntryPoints.tsx`:**
- Rimuovere il badge "Plus" accanto al nome scheda (se visibile anche con `isPlusActive = true`)
- Il bottom sheet deve mostrare CTA "Apri" / "Open" — senza testo "Questa scheda è disponibile con Plus."
- Lo stato "Configurata" / "Da configurare" deve essere visibile come da Epic 14

**`src/pages/SettingsPage.tsx` — sezione Temi (linee 218-238):**
- Rimuovere il badge "Plus" dalle palette Ocean, Sunset, Forest
- Tap su qualsiasi tema → lo attiva senza messaggi o link di upgrade
- Rimuovere il testo "Disponibile con Plus" e il link "Scopri Plus" se presenti

**`src/pages/AreaForm.tsx`:**
- Selezionando `tracking_mode = quantity_reduce`: rimuovere l'avviso inline "Il tracciamento quantitativo è disponibile con Plus."
- Il flusso è identico per tutti i tipi di tracking, senza avvisi Plus

**`src/pages/AreaDetail.tsx`:**
- Rimuovere il badge "Grafico quantità disponibile con Plus"
- Il grafico quantità per aree `quantity_reduce` è visibile senza condizioni

---

## story-18-04 — Schede sempre attive

Continua Epic 18 di opad.me. Rendi le Schede sempre attive per tutti gli utenti e rimuovi il loro toggle dalle Impostazioni.

**`src/pages/SettingsPage.tsx` — rimuovi la sezione Schede:**
- La sezione "Schede / Cards" con i toggle per ogni scheda non deve apparire in Settings
- Le sezioni Preferenze e Account rimangono invariate

**Entry point schede in Attività — auto-enable:**

Scegli l'opzione più semplice da implementare:

- **Opzione A:** Le schede vengono mostrate sempre come entry point in Attività, indipendentemente da `user_cards.enabled`. Se l'utente non ha un record `user_cards`, la scheda appare con stato "Da configurare".
- **Opzione B:** All'accesso utente, fare upsert silenzioso in `user_cards` per tutte le schede disponibili con `enabled = true` se non esistono già.

**Comportamento invariato (da Epic 14):**
- Schede con area collegata presente → badge "Configurata"
- Schede senza area collegata → badge "Da configurare", pagina dedicata mostra wizard
- Tap su entry point → bottom sheet anteprima con CTA "Apri"
- Pagine `/cards/gym` e `/cards/finance` funzionano normalmente

**`src/components/home/ActivityCard.tsx`:**
- Il `QuantityCounter` deve essere visibile per tutte le aree `quantity_reduce` con `show_quick_add_home = true`, senza dipendenze da `anyCardEnabled` o `isPlusActive`

---

## story-18-05 — Cleanup finale: voce Plus in Account e route /plus

Continua Epic 18 di opad.me. Rimuovi la voce "Plus" dalla sezione Account di Settings e la route `/plus` dall'app.

**Nota preliminare:** se story-15-03 (pagina `/plus`) e story-15-07 (voce Plus in Account) non sono ancora state implementate nel codebase, questa story non ha effetto — verificare prima di procedere.

**`src/pages/SettingsPage.tsx` — sezione Account:**
- Rimuovere la voce "Plus" con badge stato ("Non attivo" / "Attivo") e chevron → `/plus`
- La sezione Account mostra solo: email read-only + bottone Esci + bottone Elimina account

**Route `/plus`:**
- Rimuovere la route `/plus` da `src/App.tsx` (o il file di routing principale)
- Rimuovere `src/pages/PlusPage.tsx` se esiste
- Verificare che nessun altro componente contenga link o navigate(`/plus`) e rimuoverli
