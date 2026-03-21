# Stories — Epic 16: PostHog Analytics

## Sequenza di implementazione

```
story-16-01  Setup SDK
story-16-02  Identify utente
story-16-03  Eventi onboarding + auth
story-16-04  Eventi check-in
story-16-05  Eventi aree
story-16-06  Eventi schede + gym
story-16-07  Eventi navigazione
story-16-08  Eventi Plus
story-16-09  Eventi settings
story-16-10  Aggiornamento proprietà utente
```

---

## story-16-01 — Setup PostHog SDK

**Contesto:** opad.me è un'app React di osservazione del benessere personale. Vogliamo integrare PostHog per la product analytics.

**Cosa fare:**
- Installare il pacchetto `posthog-js`
- Creare un modulo di inizializzazione PostHog che venga eseguito all'avvio dell'app
- Configurare PostHog con:
  - API key da variabile d'ambiente (`VITE_POSTHOG_API_KEY`)
  - Host da variabile d'ambiente (`VITE_POSTHOG_HOST`)
  - Autocapture: disabilitato
  - Pageview automatico: abilitato
  - Session recording: disabilitato
  - Persistence: `localStorage`
  - Rispetto Do Not Track: abilitato
- Se la API key non è presente (es. ambiente di sviluppo), PostHog non si inizializza e l'app funziona normalmente

**Acceptance criteria:**
- [ ] PostHog SDK installato e inizializzato all'avvio
- [ ] Autocapture disabilitato
- [ ] Pageview automatico abilitato
- [ ] API key e host in variabili d'ambiente
- [ ] L'app funziona senza errori anche senza le variabili PostHog configurate

---

## story-16-02 — Identificazione utente

**Contesto:** PostHog è inizializzato (story-16-01). Ora dobbiamo collegare le sessioni agli utenti autenticati.

**Cosa fare:**
- Al **login** (email/password o Google OAuth): chiamare `posthog.identify(user_id)` con le proprietà utente:
  - `plan`: `"free"` o `"plus"` (da `plus_active`)
  - `language`: lingua selezionata
  - `areas_count`: numero aree attive
  - `cards_enabled`: lista schede abilitate
  - `created_at`: data creazione account
- Al **signup** (fine onboarding): stessa identify call
- Al **caricamento app con sessione attiva**: ri-identificare l'utente
- Al **logout**: chiamare `posthog.reset()` per disconnettere la sessione
- In **demo mode**: non chiamare identify (l'utente resta anonimo)
- Mai inviare email, nomi o altri dati personali

**Acceptance criteria:**
- [ ] Login → `posthog.identify()` con proprietà corrette
- [ ] Signup → `posthog.identify()` con proprietà corrette
- [ ] Reload con sessione → `posthog.identify()`
- [ ] Logout → `posthog.reset()`
- [ ] Demo mode → nessun identify
- [ ] Nessun dato personale inviato

---

## story-16-03 — Eventi onboarding e autenticazione

**Contesto:** L'utente è identificato (story-16-02). Tracciamo gli eventi di onboarding e autenticazione.

**Cosa fare:**
- Tracciare i seguenti eventi:
  - `onboarding_started` — quando l'utente inizia l'onboarding
  - `onboarding_step_completed` — quando completa uno step, con proprietà `step` (1, 2, 3)
  - `onboarding_completed` — quando completa l'onboarding, con proprietà `areas_count`
  - `auth_login` — login riuscito, con proprietà `method` (`"email"` o `"google"`)
  - `auth_signup` — signup riuscito, con proprietà `method` (`"email"` o `"google"`)
  - `auth_logout` — al logout

**Acceptance criteria:**
- [ ] Tutti gli eventi vengono tracciati nei punti corretti del flusso
- [ ] Le proprietà sono corrette per ogni evento
- [ ] Gli eventi di onboarding coprono l'intero funnel (start → step → complete)

---

## story-16-04 — Eventi check-in

**Contesto:** Tracciamo le interazioni con i check-in — l'azione principale dell'app.

**Cosa fare:**
- Tracciare i seguenti eventi:
  - `checkin_completed` — quando l'utente completa un check-in (dalla Home o dall'Area Detail), con proprietà:
    - `area_type`: tipo dell'area (`"health"`, `"study"`, `"reduce"`, `"finance"`)
    - `tracking_mode`: `"binary"` o `"quantity_reduce"`
    - `source`: `"home"` o `"area_detail"` (da dove viene fatto il check-in)
  - `checkin_undo` — quando l'utente annulla un check-in, con proprietà `area_type`

**Acceptance criteria:**
- [ ] `checkin_completed` tracciato con proprietà corrette
- [ ] `checkin_undo` tracciato con proprietà corrette
- [ ] La `source` distingue correttamente Home vs Area Detail

---

## story-16-05 — Eventi aree

**Contesto:** Tracciamo la gestione delle aree di osservazione.

**Cosa fare:**
- Tracciare i seguenti eventi:
  - `area_created` — nuova area creata, con proprietà `area_type` e `tracking_mode`
  - `area_edited` — area modificata, con proprietà `area_type`
  - `area_archived` — area archiviata, con proprietà `area_type`

**Acceptance criteria:**
- [ ] Tutti e tre gli eventi vengono tracciati
- [ ] Le proprietà riflettono il tipo e la modalità corretti

---

## story-16-06 — Eventi schede e sessione gym

**Contesto:** Tracciamo le interazioni con le schede (moduli specialistici) e le sessioni palestra.

**Cosa fare:**
- Tracciare i seguenti eventi:
  - `card_enabled` — scheda abilitata, con proprietà `card_type` (es. `"gym"`, `"finance_projection"`)
  - `card_disabled` — scheda disabilitata, con proprietà `card_type`
  - `card_opened` — l'utente apre una pagina scheda, con proprietà `card_type`
  - `session_gym_started` — inizio sessione palestra
  - `session_gym_completed` — fine sessione palestra, con proprietà `exercises_done` (numero esercizi completati)

**Acceptance criteria:**
- [ ] Eventi schede tracciati da Settings e dalla pagina Attività
- [ ] Eventi gym tracciati dalla pagina `/cards/gym`

---

## story-16-07 — Eventi navigazione

**Contesto:** Tracciamo come l'utente si muove nell'app, oltre al pageview automatico.

**Cosa fare:**
- Tracciare i seguenti eventi:
  - `tab_switched` — cambio tab nella bottom nav / sidebar, con proprietà `tab` (`"home"`, `"activities"`, `"progress"`, `"settings"`)
  - `area_detail_viewed` — apertura di un Area Detail, con proprietà `area_type` e `time_range`

**Acceptance criteria:**
- [ ] `tab_switched` tracciato ad ogni cambio tab
- [ ] `area_detail_viewed` tracciato con le proprietà corrette

---

## story-16-08 — Eventi Plus

**Contesto:** Tracciamo il funnel di conversione Plus.

**Cosa fare:**
- Tracciare i seguenti eventi:
  - `plus_page_viewed` — visita della pagina `/plus`, con proprietà `source`:
    - `"banner"` se proviene dal banner Plus nella Home
    - `"settings"` se proviene dalla voce Plus in Settings
    - `"gating"` se proviene da un CTA di una feature bloccata
  - `plus_upgrade_clicked` — tap su "Attiva Plus" nella pagina Plus
  - `plus_banner_dismissed` — chiusura del banner Plus nella Home

**Acceptance criteria:**
- [ ] La `source` del `plus_page_viewed` è corretta per ogni punto di ingresso
- [ ] `plus_upgrade_clicked` tracciato al tap sul CTA
- [ ] `plus_banner_dismissed` tracciato alla chiusura del banner

---

## story-16-09 — Eventi settings

**Contesto:** Tracciamo le modifiche alle impostazioni per capire le preferenze degli utenti.

**Cosa fare:**
- Tracciare i seguenti eventi:
  - `settings_language_changed` — cambio lingua, con proprietà `language` (`"it"` o `"en"`)
  - `settings_theme_changed` — cambio tema o palette, con proprietà `theme` (es. `"light"`, `"dark"`, `"system"`) e `palette` (es. `"teal"`, `"ocean"`)
  - `settings_score_toggled` — toggle visibilità score, con proprietà `visible` (boolean)

**Acceptance criteria:**
- [ ] Tutti gli eventi vengono tracciati al momento della modifica
- [ ] Le proprietà riflettono i valori selezionati

---

## story-16-10 — Aggiornamento proprietà utente

**Contesto:** Le proprietà utente su PostHog devono restare allineate quando cambiano (piano, lingua, aree, schede).

**Cosa fare:**
- Aggiornare le proprietà utente con `posthog.identify(user_id, { $set: { ... } })` quando:
  - L'utente attiva o disattiva Plus → aggiornare `plan`
  - L'utente cambia lingua → aggiornare `language`
  - L'utente crea o archivia un'area → aggiornare `areas_count`
  - L'utente abilita o disabilita una scheda → aggiornare `cards_enabled`

**Acceptance criteria:**
- [ ] Ogni cambiamento di stato aggiorna le proprietà utente su PostHog
- [ ] Le proprietà sono sempre coerenti con lo stato reale dell'app
