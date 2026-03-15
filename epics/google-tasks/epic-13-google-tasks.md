# Epic 13 — Google Tasks Integration

## Obiettivo

Permettere all'utente di collegare il proprio account Google e sincronizzare le attività di opad.me con Google Tasks, in modo che il check-in possa avvenire anche dall'ecosistema Google (Calendar, Tasks app, Assistant).

---

## Contesto

L'integrazione è **opzionale e opt-in**: l'utente decide se collegare Google e, per ogni area, se attivare la sincronizzazione. Non è una feature core — è un'estensione di comodità per chi usa già Google Tasks nella propria routine.

---

## Flusso principale

### 1. Collegamento account Google (Settings)

1. Utente va in **Impostazioni → Google Tasks**
2. Stato iniziale: "Non collegato" con CTA **"Collega Google Tasks"**
3. Tap → Edge Function `google-oauth-start` genera URL OAuth Google con scope `tasks` e `email`
4. Redirect a Google consent screen
5. Dopo autorizzazione → callback su Edge Function `google-oauth-callback`
6. Callback salva `refresh_token`, `access_token`, `token_expires_at`, `google_email` in `google_oauth_tokens`
7. Redirect all'app → Settings mostra stato **"Collegato"** con email Google visibile
8. CTA diventa **"Scollega"** (rosso, con conferma)

### 2. Attivazione sync per area (Add/Edit Area)

1. Nel form di creazione/modifica area, **solo per aree con `tracking_mode = binary`**, appare un toggle: **"Sincronizza con Google Tasks"**
2. Il toggle è visibile **solo se** l'utente ha un account Google collegato (record in `google_oauth_tokens` con status attivo)
3. Toggle ON → `areas.google_tasks_sync = true`
4. Toggle OFF → `areas.google_tasks_sync = false`

### 3. Sincronizzazione bidirezionale

#### Push (opad.me → Google Tasks)
- Quando l'utente fa check-in su Home per un'area con `google_tasks_sync = true`:
  - Fire-and-forget call a Edge Function `sync-google-tasks`
  - La function crea/aggiorna un task nella lista Google Tasks dell'utente
  - Task completato in Google = check-in fatto

#### Pull (Google Tasks → opad.me)
- Edge Function `pull-google-tasks` interroga Google Tasks API
- Recupera task completati dall'ultima sync
- Crea/aggiorna `checkins` corrispondenti in Supabase
- Trigger: manuale da Settings o automatico al caricamento di Home

---

## Database

### Tabella: `google_oauth_tokens`

| Colonna | Tipo | Note |
|---------|------|------|
| `id` | UUID PK | gen_random_uuid() |
| `user_id` | UUID FK → auth.users | UNIQUE (un collegamento per utente) |
| `google_email` | TEXT | Email dell'account Google collegato |
| `refresh_token` | TEXT | Token per refresh automatico |
| `access_token` | TEXT | Token di accesso corrente |
| `token_expires_at` | TIMESTAMPTZ | Scadenza access token |
| `connected_at` | TIMESTAMPTZ | Data collegamento |
| `status` | TEXT | Stato connessione (active, revoked, error) |
| `created_at` | TIMESTAMPTZ | DEFAULT now() |
| `updated_at` | TIMESTAMPTZ | Auto-update via trigger |

**RLS:** SELECT, INSERT, UPDATE, DELETE filtrati su `auth.uid() = user_id`.

### Colonna aggiunta: `areas.google_tasks_sync`

| Colonna | Tipo | Note |
|---------|------|------|
| `google_tasks_sync` | BOOLEAN | DEFAULT false — abilita sync per questa area |

---

## Edge Functions

### `google-oauth-start`
- **Metodo:** GET
- **Input:** JWT utente (da header Authorization)
- **Output:** Redirect URL verso Google OAuth consent screen
- **Scope richiesti:** `https://www.googleapis.com/auth/tasks`, `email`
- **Parametri OAuth:** `access_type=offline`, `prompt=consent` (per ottenere refresh_token)

### `google-oauth-callback`
- **Metodo:** GET
- **Input:** `code` e `state` da query params (Google OAuth redirect)
- **Azioni:**
  1. Scambia `code` per `access_token` + `refresh_token` via Google token endpoint
  2. Recupera email Google via userinfo endpoint
  3. Upsert in `google_oauth_tokens` (user_id dal state token)
  4. Redirect all'app su `/settings` con parametro di successo

### `sync-google-tasks`
- **Metodo:** POST
- **Input:** `{ area_id, date, completed }` + JWT utente
- **Azioni:**
  1. Verifica che l'area abbia `google_tasks_sync = true`
  2. Recupera token OAuth dell'utente
  3. Refresh token se scaduto
  4. Crea o aggiorna task in Google Tasks API
  5. Task title = nome area, due date = data check-in
- **Errori:** Token revocato → aggiorna status a "revoked", non blocca check-in

### `pull-google-tasks`
- **Metodo:** POST
- **Input:** JWT utente
- **Azioni:**
  1. Recupera tutte le aree dell'utente con `google_tasks_sync = true`
  2. Recupera token OAuth
  3. Interroga Google Tasks API per task completati
  4. Per ogni task completato, upsert in `checkins` se non già presente
  5. Aggiorna `updated_at` del token
- **Errori:** Token scaduto → tenta refresh; revocato → aggiorna status

### `calculate-score`
- **Metodo:** POST
- **Input:** `{ area_id }` + JWT utente
- **Azioni:**
  1. Recupera check-in dell'area
  2. Calcola `daily_score` e `cumulative_score` secondo logica scoring (vedi Epic 03)
  3. Upsert in `score_daily`

### `delete-account`
- **Metodo:** POST (o DELETE)
- **Input:** JWT utente
- **Azioni:**
  1. Cancella tutti i dati utente (cascade via FK)
  2. Revoca token Google se collegato
  3. Elimina record auth.users via `service_role`

### `seed-test-data`
- **Metodo:** POST
- **Solo dev/staging** — genera dati di test per un utente

---

## Stati UI in Settings

| Stato | UI | Azione disponibile |
|-------|----|----|
| **Non collegato** | Testo "Non collegato" + CTA "Collega Google Tasks" | Tap → OAuth flow |
| **Collegamento in corso** | Spinner / loading | — |
| **Collegato** | Email Google + badge "Attivo" + CTA "Scollega" (rosso) | Tap "Scollega" → conferma → disconnect |
| **Errore/Revocato** | Messaggio "Connessione scaduta" + CTA "Ricollega" | Tap → nuovo OAuth flow |

---

## Edge case

- **Token revocato da Google:** L'utente revoca accesso da Google Account → al prossimo sync, la function rileva errore 401 → aggiorna status a "revoked" → Settings mostra stato errore
- **Area archiviata:** Se un'area con sync attiva viene archiviata, la sync si interrompe (query filtra `archived_at IS NULL`)
- **Nessuna lista Google Tasks:** La function crea automaticamente una lista "opad.me" se non esiste
- **Conflitto date:** Se un check-in esiste già per (area_id, date), l'upsert non crea duplicati (UNIQUE constraint)
- **Demo mode:** Google Tasks section nascosta in demo mode
- **Offline/errore rete:** Il check-in locale non viene bloccato — sync è fire-and-forget

---

## Scope escluso (MVP)

- Sync automatica periodica (cron) — solo trigger manuale o al load di Home
- Scelta della lista Google Tasks (usa lista dedicata "opad.me")
- Sync per aree `quantity_reduce` (solo binary)
- Notifiche Google Tasks → opad.me in real-time
