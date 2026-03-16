# Stories — Epic 13: Google Tasks Integration

> Riferimento: `epics/google-tasks/epic-13-google-tasks.md`

---

## Story 13-01 — Tabella google_oauth_tokens + colonna areas.google_tasks_sync ✅

**Come** sistema
**voglio** avere le tabelle e colonne necessarie per Google Tasks
**così che** i token OAuth e le preferenze di sync siano persistiti in modo sicuro.

### Dettaglio

- Creare tabella `google_oauth_tokens` con: id (UUID PK), user_id (UUID FK UNIQUE), google_email (TEXT), refresh_token (TEXT), access_token (TEXT), token_expires_at (TIMESTAMPTZ), connected_at (TIMESTAMPTZ), status (TEXT), created_at, updated_at
- RLS abilitata: SELECT, INSERT, UPDATE, DELETE filtrati su `auth.uid() = user_id`
- Trigger auto-update su `updated_at`
- Aggiungere colonna `google_tasks_sync BOOLEAN NOT NULL DEFAULT false` alla tabella `areas`

### Acceptance criteria

- [ ] Tabella `google_oauth_tokens` esiste con RLS attiva
- [ ] Vincolo UNIQUE su `user_id` (un solo collegamento per utente)
- [ ] 4 policy RLS (SELECT, INSERT, UPDATE, DELETE) su `auth.uid() = user_id`
- [ ] Colonna `areas.google_tasks_sync` esiste con default `false`
- [ ] Trigger `updated_at` funziona su ogni UPDATE

---

## Story 13-02 — Edge Function google-oauth-start ✅

**Come** utente
**voglio** avviare il collegamento con Google Tasks
**così che** possa autorizzare opad.me ad accedere ai miei task.

### Dettaglio

- Edge Function `google-oauth-start` (GET)
- Riceve JWT utente da header Authorization
- Genera URL OAuth Google con:
  - `scope`: `https://www.googleapis.com/auth/tasks` + `email`
  - `access_type`: `offline`
  - `prompt`: `consent`
  - `state`: token cifrato contenente user_id
  - `redirect_uri`: URL della callback function
- Risponde con redirect 302 verso Google consent screen

### Acceptance criteria

- [ ] Utente autenticato riceve redirect a Google consent
- [ ] Scope include `tasks` e `email`
- [ ] `access_type=offline` per ottenere refresh_token
- [ ] State contiene user_id in modo sicuro
- [ ] Utente non autenticato riceve 401

---

## Story 13-03 — Edge Function google-oauth-callback ✅

**Come** sistema
**voglio** gestire il ritorno da Google dopo l'autorizzazione
**così che** i token vengano salvati e l'utente torni in Settings.

### Dettaglio

- Edge Function `google-oauth-callback` (GET)
- Riceve `code` e `state` da query params
- Scambia `code` per `access_token` + `refresh_token` via Google token endpoint
- Recupera `google_email` via Google userinfo endpoint
- Upsert in `google_oauth_tokens`:
  - `user_id` dal state decodificato
  - `google_email`, `refresh_token`, `access_token`, `token_expires_at`
  - `status = 'active'`, `connected_at = now()`
- Redirect a `/settings?google_connected=true`

### Acceptance criteria

- [ ] Token scambiato correttamente con Google
- [ ] Record creato/aggiornato in `google_oauth_tokens`
- [ ] Email Google salvata
- [ ] Redirect a Settings con parametro di successo
- [ ] Errore OAuth → redirect a Settings con parametro di errore

---

## Story 13-04 — UI Google Tasks in Settings ✅

**Come** utente
**voglio** vedere lo stato del collegamento Google Tasks in Impostazioni
**così che** possa collegare, verificare o scollegare il mio account Google.

### Dettaglio

Sezione "Google Tasks" in SettingsPage con 4 stati:

1. **Non collegato:** Testo secondario + CTA "Collega Google Tasks" (stile primary)
2. **Collegamento in corso:** Spinner con testo "Collegamento..."
3. **Collegato:** Email Google visibile + badge "Attivo" verde + CTA "Scollega" (stile destructive, con AlertDialog di conferma)
4. **Errore/Revocato:** Messaggio "Connessione scaduta" + CTA "Ricollega"

- Scollega: DELETE da `google_oauth_tokens` + reset `google_tasks_sync = false` su tutte le aree dell'utente
- Nascosto in demo mode

### Acceptance criteria

- [ ] 4 stati visuali distinti
- [ ] CTA "Collega" avvia OAuth flow (chiama `google-oauth-start`)
- [ ] Stato "Collegato" mostra email Google
- [ ] "Scollega" richiede conferma tramite AlertDialog
- [ ] Dopo scollegamento, tutte le aree tornano a `google_tasks_sync = false`
- [ ] Sezione nascosta in demo mode
- [ ] Testi localizzati IT/EN

---

## Story 13-05 — Toggle Google Tasks sync in Add/Edit Area ✅

**Come** utente
**voglio** attivare la sincronizzazione Google Tasks per una singola area
**così che** i check-in di quell'area vengano sincronizzati con Google.

### Dettaglio

- Nel form AreaForm, **sotto** il selettore frequenza:
  - Toggle "Sincronizza con Google Tasks" (Switch component)
  - Visibile **solo se**:
    1. Utente ha Google collegato (record `google_oauth_tokens` con status `active`)
    2. Area ha `tracking_mode = 'binary'` (non disponibile per quantity_reduce)
  - Toggle ON → `areas.google_tasks_sync = true`
  - Toggle OFF → `areas.google_tasks_sync = false`
- In create mode: toggle OFF di default
- In edit mode: toggle mostra stato corrente dell'area

### Acceptance criteria

- [ ] Toggle visibile solo con Google collegato + tracking_mode binary
- [ ] Toggle nascosto per aree quantity_reduce
- [ ] Toggle nascosto se Google non collegato
- [ ] Valore salvato correttamente su INSERT/UPDATE dell'area
- [ ] Testo localizzato IT/EN

---

## Story 13-06 — Edge Function sync-google-tasks (push) ✅

**Come** sistema
**voglio** sincronizzare un check-in verso Google Tasks
**così che** il task risulti completato anche in Google.

### Dettaglio

- Edge Function `sync-google-tasks` (POST)
- Input: `{ area_id, date, completed }` + JWT utente
- Logica:
  1. Verifica `areas.google_tasks_sync = true` per l'area
  2. Recupera token da `google_oauth_tokens`
  3. Se token scaduto → refresh via Google token endpoint → aggiorna DB
  4. Se `completed = true` → crea/completa task in Google Tasks (lista "opad.me")
  5. Se `completed = false` (undo) → riapri task se esiste
  6. Task title = nome area, due date = data check-in
- Fire-and-forget: chiamata da Home senza attendere risposta
- Se token revocato: aggiorna status a "revoked", non bloccare check-in locale

### Acceptance criteria

- [ ] Check-in completato → task completato in Google Tasks
- [ ] Undo check-in → task riaperto in Google Tasks
- [ ] Token scaduto → refresh automatico trasparente
- [ ] Token revocato → status aggiornato, check-in locale non bloccato
- [ ] Area senza sync → function ritorna senza azione
- [ ] Lista "opad.me" creata automaticamente se non esiste

---

## Story 13-07 — Edge Function pull-google-tasks (pull) ✅

**Come** utente
**voglio** che i task completati in Google vengano riflessi in opad.me
**così che** non debba fare check-in doppio.

### Dettaglio

- Edge Function `pull-google-tasks` (POST)
- Input: JWT utente
- Logica:
  1. Recupera tutte le aree dell'utente con `google_tasks_sync = true` e `archived_at IS NULL`
  2. Recupera token OAuth
  3. Se token scaduto → refresh
  4. Interroga Google Tasks API: task completati nella lista "opad.me"
  5. Per ogni task completato, upsert in `checkins(area_id, date, completed=true)`
  6. UNIQUE constraint previene duplicati
- Trigger: chiamata al caricamento di Home (fire-and-forget)

### Acceptance criteria

- [ ] Task completati in Google → checkins creati in opad.me
- [ ] Nessun duplicato grazie a UNIQUE(area_id, date)
- [ ] Token scaduto → refresh automatico
- [ ] Token revocato → status aggiornato, nessun crash
- [ ] Aree archiviate ignorate
- [ ] Solo aree con `google_tasks_sync = true` considerate

---

## Story 13-08 — Trigger sync da Home al caricamento ✅

**Come** utente
**voglio** che all'apertura di Home i miei task Google vengano sincronizzati
**così che** veda lo stato aggiornato senza azioni manuali.

### Dettaglio

- In `Index.tsx` (Home), al mount della pagina:
  - Se utente ha Google collegato → fire-and-forget call a `pull-google-tasks`
  - Se utente ha Google collegato e fa check-in su area con sync → fire-and-forget call a `sync-google-tasks`
- Nessun loading state visibile per la sync (è background)
- Se la call fallisce, nessun errore mostrato all'utente

### Acceptance criteria

- [ ] Pull automatico al mount di Home per utenti con Google collegato
- [ ] Push automatico al check-in per aree con sync attiva
- [ ] Nessun spinner o loading state visibile per sync
- [ ] Errori sync silenziosi (non bloccano l'UX)
- [ ] Nessuna call se utente non ha Google collegato
