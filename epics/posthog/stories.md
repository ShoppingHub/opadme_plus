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

### Step 1 — Snippet in index.html

Inserire il seguente snippet nel `<head>` di `index.html`, **prima** di qualsiasi altro script:

```html
<script>
    !function(t,e){var o,n,p,r;e.__SV||(window.posthog=e,e._i=[],e.init=function(i,s,a){function g(t,e){var o=e.split(".");2==o.length&&(t=t[o[0]],e=o[1]),t[e]=function(){t.push([e].concat(Array.prototype.slice.call(arguments,0)))}}(p=t.createElement("script")).type="text/javascript",p.async=!0,p.src=s.api_host.replace(".i.posthog.com","-assets.i.posthog.com")+"/static/array.js",(r=t.getElementsByTagName("script")[0]).parentNode.insertBefore(p,r);var u=e;for(void 0!==a?u=e[a]=[]:a="posthog",u.people=u.people||[],u.toString=function(t){var e="posthog";return"posthog"!==a&&(e+="."+a),t||(e+=" (stub)"),e},u.people.toString=function(){return u.toString(1)+".people (stub)"},o="init capture register register_once register_for_session unregister opt_out_capturing has_opted_out_capturing opt_in_capturing reset isFeatureEnabled getFeatureFlag getFeatureFlagPayload reloadFeatureFlags group identify setPersonProperties setPersonPropertiesForFlags resetPersonPropertiesForFlags setGroupPropertiesForFlags resetGroupPropertiesForFlags resetGroups onFeatureFlags addFeatureFlagsHandler onSessionId getSurveys getActiveMatchingSurveys renderSurvey canRenderSurvey getNextSurveyStep".split(" "),n=0;n<o.length;n++)g(u,o[n]);e._i.push([i,s,a])},e.__SV=1)}(document,window.posthog||[]);
    posthog.init('phc_8B5fL5BsMjtGg0GJ8AB0220jGklzgYqyvk8sFftnUGA', {
        api_host: 'https://eu.i.posthog.com',
        defaults: '2026-01-30'
    })
</script>
```

### Step 2 — Wrapper per il codice React

Installare `posthog-js` come dipendenza e creare un modulo helper (es. `src/lib/posthog.ts`) che esponga `posthog` per l'uso nel codice React:

```typescript
import posthog from 'posthog-js'
export default posthog
```

Lo snippet in `index.html` gestisce l'inizializzazione. Il modulo helper serve solo per avere l'import tipizzato nelle stories successive (identify, capture, reset).

### Step 3 — Configurazione aggiuntiva

Dopo l'init nello snippet, aggiungere queste opzioni di configurazione (nello snippet stesso o nel modulo helper):
- Autocapture: **disabilitato** (`autocapture: false`)
- Session recording: **disabilitato** (`disable_session_recording: true`)
- Rispetto Do Not Track: **abilitato** (`respect_dnt: true`)
- Persistence: `localStorage`

L'init nello snippet diventa:
```javascript
posthog.init('phc_8B5fL5BsMjtGg0GJ8AB0220jGklzgYqyvk8sFftnUGA', {
    api_host: 'https://eu.i.posthog.com',
    defaults: '2026-01-30',
    autocapture: false,
    disable_session_recording: true,
    respect_dnt: true,
    persistence: 'localStorage'
})
```

**Acceptance criteria:**
- [ ] Lo snippet PostHog è nel `<head>` di `index.html`
- [ ] Il pacchetto `posthog-js` è installato come dipendenza
- [ ] Un modulo helper `src/lib/posthog.ts` esporta l'istanza PostHog per l'uso nel codice React
- [ ] Autocapture disabilitato
- [ ] Session recording disabilitato
- [ ] Do Not Track rispettato
- [ ] L'app funziona normalmente — PostHog non blocca il rendering

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
  - `area_created` — nuova area creata, con proprietà:
    - `area_type`: macro-categoria (`"health"`, `"study"`, `"reduce"`, `"finance"`)
    - `tracking_mode`: `"binary"` o `"quantity_reduce"`
    - `source`: da dove viene creata l'attività:
      - `"onboarding"` — durante il flusso di onboarding
      - `"activities_tab"` — dal tab Attività (pulsante +)
      - `"empty_state"` — dal CTA dello stato vuoto di una macro-categoria
    - `category_count`: numero totale di attività attive nella stessa macro-categoria **dopo** la creazione (es. se l'utente crea la 3ª attività in "health", il valore è `3`)
  - `area_edited` — area modificata, con proprietà `area_type`
  - `area_archived` — area archiviata, con proprietà `area_type`

**Acceptance criteria:**
- [ ] Tutti e tre gli eventi vengono tracciati
- [ ] `area_created` include `area_type`, `tracking_mode`, `source` e `category_count`
- [ ] La `source` distingue correttamente onboarding vs activities_tab vs empty_state
- [ ] `category_count` riflette il numero di attività attive nella macro-categoria dopo la creazione

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
