# Epic 16 — PostHog Analytics

## Obiettivo

Integrare **PostHog** come piattaforma di product analytics per opad.me. L'obiettivo è osservare come gli utenti interagiscono con l'app — quali funzionalità usano, dove si fermano, come si muovono tra le schermate — senza raccogliere dati personali sensibili.

PostHog permette di:
- Identificare gli utenti autenticati (senza dati personali)
- Tracciare eventi custom per le azioni principali
- Analizzare funnel, retention e pattern d'uso
- Prendere decisioni di prodotto basate su dati reali

---

## Concetto

```
Utente                          PostHog
─────────────────               ─────────────────
Login / Signup                  → identify (user_id, piano, lingua)
Navigazione tra tab             → $pageview (automatico)
Azioni principali               → custom event (check-in, area, scheda...)
Upgrade Plus                    → custom event (conversione)
```

L'integrazione è **invisibile all'utente**. Nessun banner, nessun toggle, nessuna UI aggiuntiva. PostHog lavora in background.

---

## Inizializzazione

### Snippet HTML

PostHog viene inizializzato tramite snippet nel `<head>` di `index.html`. Lo snippet carica l'SDK in modo asincrono e non blocca il rendering.

```html
<script>
    !function(t,e){var o,n,p,r;e.__SV||(window.posthog=e,e._i=[],e.init=function(i,s,a){function g(t,e){var o=e.split(".");2==o.length&&(t=t[o[0]],e=o[1]),t[e]=function(){t.push([e].concat(Array.prototype.slice.call(arguments,0)))}}(p=t.createElement("script")).type="text/javascript",p.async=!0,p.src=s.api_host.replace(".i.posthog.com","-assets.i.posthog.com")+"/static/array.js",(r=t.getElementsByTagName("script")[0]).parentNode.insertBefore(p,r);var u=e;for(void 0!==a?u=e[a]=[]:a="posthog",u.people=u.people||[],u.toString=function(t){var e="posthog";return"posthog"!==a&&(e+="."+a),t||(e+=" (stub)"),e},u.people.toString=function(){return u.toString(1)+".people (stub)"},o="init capture register register_once register_for_session unregister opt_out_capturing has_opted_out_capturing opt_in_capturing reset isFeatureEnabled getFeatureFlag getFeatureFlagPayload reloadFeatureFlags group identify setPersonProperties setPersonPropertiesForFlags resetPersonPropertiesForFlags setGroupPropertiesForFlags resetGroupPropertiesForFlags resetGroups onFeatureFlags addFeatureFlagsHandler onSessionId getSurveys getActiveMatchingSurveys renderSurvey canRenderSurvey getNextSurveyStep".split(" "),n=0;n<o.length;n++)g(u,o[n]);e._i.push([i,s,a])},e.__SV=1)}(document,window.posthog||[]);
    posthog.init('phc_8B5fL5BsMjtGg0GJ8AB0220jGklzgYqyvk8sFftnUGA', {
        api_host: 'https://eu.i.posthog.com',
        defaults: '2026-01-30',
        autocapture: false,
        disable_session_recording: true,
        respect_dnt: true,
        persistence: 'localStorage'
    })
</script>
```

### Wrapper React

Il pacchetto `posthog-js` viene installato come dipendenza per avere l'import tipizzato nel codice React. Un modulo helper (`src/lib/posthog.ts`) esporta l'istanza per le stories successive (identify, capture, reset).

### Configurazione

| Proprietà | Valore |
|---|---|
| API key | `phc_8B5fL5BsMjtGg0GJ8AB0220jGklzgYqyvk8sFftnUGA` |
| Host | `https://eu.i.posthog.com` (Cloud EU) |
| Autocapture | **Disabilitato** — troppo rumore, eventi non significativi |
| Session recording | **Disabilitato** (MVP) — valutare in futuro |
| Feature flags | **Disabilitato** (MVP) — valutare in futuro |
| Persistence | `localStorage` |
| Rispetto Do Not Track | Sì |

> Autocapture disabilitato è una scelta deliberata: opad.me ha un set di azioni ben definito. Eventi espliciti e nominati producono dati più puliti e interpretabili.

---

## Identificazione Utente

### Quando identificare

| Evento | Azione PostHog |
|---|---|
| **Login** (email/password o OAuth) | `posthog.identify(user_id, properties)` |
| **Signup** (fine onboarding) | `posthog.identify(user_id, properties)` |
| **Logout** | `posthog.reset()` |
| **App load con sessione attiva** | `posthog.identify(user_id, properties)` |

### Proprietà utente (`$set`)

| Proprietà | Tipo | Descrizione |
|---|---|---|
| `plan` | `"free"` \| `"plus"` | Piano attivo |
| `language` | `"it"` \| `"en"` | Lingua selezionata |
| `areas_count` | number | Numero di aree attive (non archiviate) |
| `cards_enabled` | string[] | Lista schede abilitate (es. `["gym", "finance_projection"]`) |
| `created_at` | ISO string | Data di creazione account |

### Proprietà escluse (mai inviare)

- Email
- Nome / cognome
- Dati di check-in specifici
- Contenuto delle aree (nomi custom)
- Dati finanziari
- Qualsiasi dato identificabile personalmente

> L'`user_id` è l'UUID di Supabase Auth — non contiene informazioni personali.

---

## Eventi Custom

### Convenzione di naming

Formato: `snake_case`, con prefisso per categoria.

```
area_created
area_archived
checkin_completed
checkin_undo
card_enabled
card_disabled
card_opened
session_gym_started
session_gym_completed
plus_page_viewed
plus_upgrade_clicked
onboarding_started
onboarding_step_completed
onboarding_completed
settings_language_changed
settings_theme_changed
settings_score_toggled
```

### Catalogo eventi

#### Onboarding

| Evento | Trigger | Proprietà |
|---|---|---|
| `onboarding_started` | L'utente inizia l'onboarding | — |
| `onboarding_step_completed` | L'utente completa uno step | `step`: number (1, 2, 3) |
| `onboarding_completed` | L'utente completa l'onboarding | `areas_count`: number |

#### Autenticazione

| Evento | Trigger | Proprietà |
|---|---|---|
| `auth_login` | Login riuscito | `method`: `"email"` \| `"google"` |
| `auth_signup` | Signup riuscito | `method`: `"email"` \| `"google"` |
| `auth_logout` | Logout | — |

#### Check-in

| Evento | Trigger | Proprietà |
|---|---|---|
| `checkin_completed` | L'utente completa un check-in | `area_type`: string, `tracking_mode`: `"binary"` \| `"quantity_reduce"`, `source`: `"home"` \| `"area_detail"` |
| `checkin_undo` | L'utente annulla un check-in | `area_type`: string |

#### Aree

| Evento | Trigger | Proprietà |
|---|---|---|
| `area_created` | Nuova area creata | `area_type`: string, `tracking_mode`: string, `source`: `"onboarding"` \| `"activities_tab"` \| `"empty_state"`, `category_count`: number |
| `area_edited` | Area modificata | `area_type`: string |
| `area_archived` | Area archiviata | `area_type`: string |

#### Schede

| Evento | Trigger | Proprietà |
|---|---|---|
| `card_enabled` | Scheda abilitata (da Settings o suggerimento) | `card_type`: string |
| `card_disabled` | Scheda disabilitata da Settings | `card_type`: string |
| `card_opened` | L'utente apre una pagina scheda | `card_type`: string |

#### Gym

| Evento | Trigger | Proprietà |
|---|---|---|
| `session_gym_started` | L'utente inizia una sessione palestra | — |
| `session_gym_completed` | L'utente completa una sessione palestra | `exercises_done`: number |

#### Navigazione

| Evento | Trigger | Proprietà |
|---|---|---|
| `tab_switched` | L'utente cambia tab nella nav | `tab`: `"home"` \| `"activities"` \| `"progress"` \| `"settings"` |
| `area_detail_viewed` | L'utente apre un Area Detail | `area_type`: string, `time_range`: string |

#### Plus

| Evento | Trigger | Proprietà |
|---|---|---|
| `plus_page_viewed` | L'utente visita la pagina Plus | `source`: `"banner"` \| `"settings"` \| `"gating"` |
| `plus_upgrade_clicked` | Tap su CTA "Attiva Plus" | — |
| `plus_banner_dismissed` | L'utente chiude il banner Plus | — |

#### Settings

| Evento | Trigger | Proprietà |
|---|---|---|
| `settings_language_changed` | Cambio lingua | `language`: `"it"` \| `"en"` |
| `settings_theme_changed` | Cambio tema o palette | `theme`: string, `palette`: string |
| `settings_score_toggled` | Toggle visibilità score | `visible`: boolean |

---

## Aggiornamento Proprietà Utente

Le proprietà utente (`$set`) vanno aggiornate quando cambiano:

| Cambiamento | Aggiornamento |
|---|---|
| Attiva/disattiva Plus | `plan`: `"free"` ↔ `"plus"` |
| Cambio lingua | `language` |
| Crea/archivia area | `areas_count` |
| Abilita/disabilita scheda | `cards_enabled` |

> Le proprietà si aggiornano con `posthog.identify(user_id, { $set: { ... } })` oppure `posthog.people.set(...)`.

---

## Privacy e Compliance

| Aspetto | Decisione |
|---|---|
| Dati personali | Mai inviati — solo UUID, piano, lingua, conteggi |
| GDPR | PostHog Cloud EU (hosting EU) — nessun trasferimento dati extra-UE |
| Do Not Track | Rispettato — se il browser ha DNT attivo, PostHog non traccia |
| Cookie banner | Non necessario — PostHog usa `localStorage`, non cookie di terze parti |
| Cancellazione dati | Alla cancellazione account, chiamare PostHog API per eliminare il profilo utente |
| Opt-out utente | Non previsto nell'MVP — l'analytics è anonimizzata e non invasiva |

---

## Demo Mode

Se l'utente è in **demo mode** (Epic 00, non autenticato):
- PostHog **non** identifica l'utente (`identify` non viene chiamato)
- Gli eventi vengono comunque tracciati come utente anonimo
- Al login/signup da demo mode → `posthog.identify()` collega la sessione anonima all'utente

---

## Edge Case

- Utente naviga in più tab del browser → PostHog gestisce automaticamente (stessa sessione via `localStorage`)
- Utente fa login su un nuovo dispositivo → PostHog crea una nuova sessione, l'identify collega all'utente esistente
- Utente cancella `localStorage` → PostHog assegna un nuovo anonymous ID, il prossimo login riallinea
- PostHog non raggiungibile (offline, errore rete) → gli eventi vengono accodati e inviati al ripristino della connessione (behavior default dell'SDK)
- Utente con ad-blocker → PostHog potrebbe essere bloccato. Nessun impatto sull'app — l'analytics è non bloccante

---

## Acceptance Criteria

- [ ] PostHog SDK è inizializzato all'avvio dell'app con autocapture disabilitato
- [ ] Al login/signup, l'utente viene identificato con `posthog.identify(user_id, properties)`
- [ ] Al logout, `posthog.reset()` viene chiamato
- [ ] Al caricamento dell'app con sessione attiva, l'utente viene ri-identificato
- [ ] Tutti gli eventi del catalogo vengono tracciati con le proprietà specificate
- [ ] Le proprietà utente vengono aggiornate quando cambiano (piano, lingua, aree, schede)
- [ ] Nessun dato personale (email, nomi, contenuti) viene inviato a PostHog
- [ ] Il Do Not Track del browser viene rispettato
- [ ] In demo mode, gli eventi sono anonimi (nessun identify)
- [ ] L'app funziona normalmente anche se PostHog è bloccato o non raggiungibile
- [ ] La API key di PostHog è in variabile d'ambiente, non hardcoded

---

## Dipendenze

- **Epic 00** (Auth) — identificazione utente al login/signup/logout
- **Epic 01** (Onboarding) — eventi onboarding
- **Epic 02** (Home) — eventi check-in dalla Home
- **Epic 03** (Check-in) — eventi check-in e undo
- **Epic 05** (Add/Edit Area) — eventi creazione/modifica area
- **Epic 07** (Settings) — eventi cambio lingua, tema, score
- **Epic 10** (Attività) — eventi navigazione
- **Epic 11** (Gym) — eventi sessione palestra
- **Epic 14** (Schede) — eventi abilitazione/apertura schede
- **Epic 15** (Plus) — eventi conversione Plus

---

## Stories

- `story-16-01` — Inizializzazione PostHog SDK (setup, variabili d'ambiente, configurazione)
- `story-16-02` — Identificazione utente (identify al login/signup, reset al logout, ri-identify al reload)
- `story-16-03` — Eventi onboarding e autenticazione
- `story-16-04` — Eventi check-in (completamento e undo)
- `story-16-05` — Eventi aree (creazione, modifica, archiviazione)
- `story-16-06` — Eventi schede e sessione gym
- `story-16-07` — Eventi navigazione (tab switch, area detail, pageview)
- `story-16-08` — Eventi Plus (page viewed, upgrade clicked, banner dismissed)
- `story-16-09` — Eventi settings (lingua, tema, score)
- `story-16-10` — Aggiornamento proprietà utente al cambio stato
