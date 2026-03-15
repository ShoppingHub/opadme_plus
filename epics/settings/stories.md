# Stories — Epic 07 — Settings

## Sequenza di implementazione

```
story-07-01 → Layout Settings: sezioni Preferenze, Menu, Account              ✅ completata
story-07-02 → Selettore lingua IT/EN con aggiornamento UI immediato           ✅ completata
story-07-03 → Toggle score: aggiornamento settings_score_visible su Supabase  ✅ completata
story-07-04 → Sezione Menu: voci fisse + max 2 slot custom configurabili      ✅ completata (deprecata)
story-07-05 → Sign out con redirect                                           ✅ completata
story-07-06 → Delete account con modale e cascade                             ✅ completata
story-07-07 → Toggle "Mostra sezione Finanze" + extra_tab_enabled su Supabase ⏳ da fare
```

> **Dipendenze:** story-07-07 richiede story-09-04 (hook `useNavConfig`) già implementata.

---

## story-07-01 — Layout Settings

BetonMe è un'app di osservazione del benessere. Aggiorna la schermata Settings (route `/settings`) con il nuovo layout a 3 sezioni.

**Struttura dall'alto verso il basso:**

**Sezione "Preferenze / Preferences":**
- Segmented control Lingua: `Italiano · English`
- Toggle: `"Mostra punteggio / Show trajectory score"` — default OFF
- Toggle: `"Notifiche / Notifications"` — default ON

**Sezione "Menu":**
- 3 righe fisse non modificabili con etichetta `(fisso)`: Home · Aree/Areas · Impostazioni/Settings

**Sezione "Account":**
- Email dell'utente (read-only, colore secondario, troncata con ellipsis)
- Bottone `"Esci / Sign out"`
- Bottone `"Elimina account / Delete account"` (testo colore errore)

**Note:**
- I toggle usano shadcn `<Switch>` con override colore attivo `#7DA3A0`
- Tutti i label seguono la lingua selezionata dall'utente
- Salvataggio silenzioso — nessun toast per le azioni non distruttive

---

## story-07-02 — Selettore lingua IT/EN

Continua Epic 07 di BetonMe. Implementa il selettore lingua.

**Behavior:**
- Il segmented control mostra `Italiano · English`
- Al tap su un'opzione → la UI dell'intera app si aggiorna immediatamente nella nuova lingua (senza reload di pagina)
- La preferenza `language` (`"it"` o `"en"`) viene salvata silenziosamente nella tabella `users`
- Al riavvio dell'app, la lingua salvata è applicata prima del rendering

**Lingua di default:**
- Se il browser è in `it-*` → default `"it"`
- Qualsiasi altra lingua browser → default `"en"`

---

## story-07-03 — Toggle score

Continua Epic 07 di BetonMe. Implementa la logica del toggle "Mostra punteggio / Show trajectory score".

**Behavior:**
- Al cambio del toggle → aggiornamento ottimistico di `settings_score_visible` nella tabella `users`
- Se `true` → il punteggio numerico diventa visibile nell'Area Detail
- Salvataggio in background; in caso di errore → toggle torna al valore precedente, nessun messaggio

---

## story-07-04 — Sezione Menu configurabile *(deprecata)*

Story completata nell'implementazione precedente. Sostituita dalla story-07-07 con la nuova architettura nav.

---

## story-07-05 — Sign out

Continua Epic 07 di BetonMe. Implementa il logout.

**Behavior:**
- Tap `"Esci / Sign out"` → `supabase.auth.signOut()`
- Bottone in opacity ridotta durante la chiamata
- Sessione invalidata → redirect a `/login`
- Nessun modale di conferma, nessun messaggio

---

## story-07-06 — Delete account

Continua Epic 07 di BetonMe. Implementa la cancellazione account.

**Behavior:**
- Tap `"Elimina account / Delete account"` → si apre un modale di conferma

**Modale (label segue lingua):**

IT:
- Titolo: `"Elimina account"`
- Corpo: `"Eliminerà definitivamente tutto il tuo storico. Questa azione è irreversibile."`
- CTA conferma: `"Elimina definitivamente"` (testo `#E24A4A`)
- CTA annulla: `"Annulla"`

EN:
- Titolo: `"Delete account"`
- Corpo: `"This will permanently delete all your observation history. This cannot be undone."`
- CTA conferma: `"Delete permanently"` (testo `#E24A4A`)
- CTA annulla: `"Cancel"`

**Al tap su conferma:**
1. CTA mostra spinner + opacity ridotta
2. Chiamata all'Edge Function Supabase che elimina l'account con cascade su `areas`, `checkins`, `score_daily`, `users`, `gym_sessions`, `gym_exercises`
3. Sessione invalidata → redirect a `/login`
4. Nessun messaggio di addio

**Edge case:**
- Errore di rete → messaggio inline sotto la CTA, nessuna azione eseguita, modale rimane aperto

---

## story-07-07 — Toggle "Mostra sezione Finanze"

Continua Epic 07 di BetonMe. Aggiungi il toggle per abilitare il 5° tab Finance nella navigazione.

**Dipende da:** story-09-04 (hook `useNavConfig` con `extra_tab_enabled` già gestito)

**Cosa aggiungere in Settings — sezione Preferenze**, dopo i toggle esistenti:

- Toggle: `"Mostra sezione Finanze"` (IT) / `"Show Finance tab"` (EN) — default OFF
- Sottotitolo IT: `"Aggiunge un accesso rapido alla proiezione finanziaria."`
- Sottotitolo EN: `"Adds quick access to the finance projection."`

**Behavior:**
- Al cambio del toggle → aggiornamento ottimistico di `extra_tab_enabled` (BOOLEAN) nella tabella `users`
- Se `true` → il tab "Finanze / Finance" appare immediatamente nella nav (bottom nav su mobile, sidebar su desktop), come 4a voce prima di Impostazioni
- Se `false` → il tab scompare dalla nav immediatamente
- Salvataggio silenzioso — nessun toast

**DB:** la colonna `extra_tab_enabled BOOLEAN DEFAULT false` deve esistere nella tabella `users`. Se non esiste, crearla tramite migrazione Supabase.

**Edge case:**
- Utente riapre l'app con `extra_tab_enabled = true` → il tab Finance è già visibile nella nav al caricamento
- Errore di salvataggio → toggle torna al valore precedente, nessun messaggio
