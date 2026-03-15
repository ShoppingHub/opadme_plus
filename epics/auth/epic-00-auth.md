# Epic 00 — Autenticazione & Gestione Account

## Obiettivo

Gestire l'intero ciclo di vita dell'identità utente: creazione account, login, logout, recupero accesso e cancellazione — con supporto sia per magic link (email) che per social login Google. Nessuna password. Nessun friction inutile.

---

## Behavior

L'autenticazione è gestita interamente da **Supabase Auth**. L'app supporta due metodi di accesso:

1. **Magic link** (email) — OTP via email, nessuna password
2. **Google OAuth** — login con un tap tramite provider Google

I due metodi sono equivalenti. Un utente può avere lo stesso account accessibile con entrambi (se usa la stessa email).

Le schermate di autenticazione usano lo stesso design system del resto dell'app — sfondo `#0F2F33`, tono osservativo, zero pressione.

---

## Flussi

---

### Flusso 00-A — Registrazione via Magic Link (nuovo utente)

1. L'utente apre l'app per la prima volta (non autenticato)
2. Vede la schermata di benvenuto con due opzioni: magic link e Google
3. Inserisce la propria email nel campo input
4. Tappa **"Send me a link"**
5. Schermata statica di conferma: *"Check your email. We sent you a magic link."*
6. L'utente apre l'email e clicca il link
7. Deep link / redirect all'app — Supabase autentica la sessione
8. Se è la prima volta → redirect all'Onboarding Step 2 (selezione aree)
9. Se l'account esiste già → redirect diretto alla Dashboard

> **Nota:** Supabase crea automaticamente il record `auth.users` al primo accesso con magic link. Non esiste una schermata di "creazione account" separata.

---

### Flusso 00-B — Login via Magic Link (utente esistente)

Identico al Flusso 00-A. Supabase gestisce internamente la distinzione nuovo/esistente. L'utente non deve scegliere tra "sign up" e "sign in" — esiste solo un'unica entry point.

---

### Flusso 00-C — Registrazione / Login via Google OAuth

1. L'utente tappa **"Continue with Google"**
2. Supabase apre il flusso OAuth Google (popup o redirect, a seconda della piattaforma)
3. L'utente seleziona o conferma il proprio account Google
4. Google restituisce il token a Supabase
5. Supabase autentica la sessione e crea/aggiorna il record utente
6. Se è la prima volta → redirect all'Onboarding Step 2
7. Se l'account esiste già → redirect alla Dashboard

> **Nota:** Se l'utente ha già un account magic link con la stessa email, Supabase collega automaticamente le due identità (identity linking).

---

### Flusso 00-D — Recupero Accesso (magic link scaduto o non ricevuto)

1. L'utente tenta di accedere con magic link ma il link è scaduto (o non lo ha ricevuto)
2. Viene reindirizzato a una schermata di errore con messaggio: *"This link has expired or is invalid."*
3. La schermata mostra un campo email pre-compilato (se disponibile dall'URL) e il bottone **"Send a new link"**
4. L'utente tappa il bottone → nuovo magic link inviato
5. Ritorna alla schermata di conferma standard

---

### Flusso 00-E — Logout

1. L'utente è nella schermata Settings
2. Tappa **"Sign out"**
3. Supabase invalida la sessione (`supabase.auth.signOut()`)
4. Redirect immediato alla schermata di autenticazione (Welcome screen)
5. Nessun messaggio di addio. Nessun modale di conferma.

---

### Flusso 00-F — Cancellazione Account

1. L'utente è nella schermata Settings
2. Tappa **"Delete account"**
3. Appare un modale di conferma con copy specifico (vedi sezione Copy UI)
4. L'utente tappa **"Delete permanently"**
5. Supabase elimina l'account (`supabase.auth.admin.deleteUser()` via Edge Function) con cascade su tutte le tabelle: `areas`, `checkins`, `score_daily`, `users`
6. Sessione invalidata
7. Redirect alla Welcome screen
8. Nessun messaggio di addio

> **Nota tecnica:** La cancellazione richiede una Supabase Edge Function con permessi `service_role` per eliminare l'utente da `auth.users`. Il frontend non può farlo direttamente.

---

### Flusso 00-G — Sessione Persistente (app reload)

1. L'utente riapre l'app dopo averla chiusa
2. Supabase controlla il token di sessione in localStorage / secure storage
3. Se la sessione è valida → redirect diretto alla Dashboard (nessun login richiesto)
4. Se la sessione è scaduta o assente → redirect alla Welcome screen

---

### Flusso 00-H — Accesso Protetto (route guard)

Ogni route dell'app principale (`/dashboard`, `/areas`, `/finance`, `/settings`) è protetta.

1. Se l'utente non è autenticato e tenta di accedere a una route protetta → redirect automatico alla Welcome screen
2. Dopo l'autenticazione → redirect alla route originalmente richiesta (o alla Dashboard se non specificata)

---

## Layout

### Welcome Screen

```
┌─────────────────────────────┐
│                             │
│                             │
│          BetonMe            │  ← Logo / wordmark, centrato
│   "Observe your direction." │  ← Tagline, text-secondary
│                             │
│                             │
│  ┌───────────────────────┐  │
│  │  your@email.com       │  │  ← Input email
│  └───────────────────────┘  │
│  ┌───────────────────────┐  │
│  │   Send me a link      │  │  ← CTA primaria
│  └───────────────────────┘  │
│                             │
│  ─────────── or ───────────  │  ← Divider
│                             │
│  ┌───────────────────────┐  │
│  │  [G]  Continue with   │  │  ← Google OAuth button
│  │       Google          │  │
│  └───────────────────────┘  │
│                             │
│  "No password needed."      │  ← Caption, text-secondary
│                             │
└─────────────────────────────┘
```

### Schermata Conferma Email Inviata

```
┌─────────────────────────────┐
│                             │
│         ✉                   │  ← Icona Mail (Lucide), 48px, #7DA3A0
│                             │
│  "Check your email."        │  ← Titolo
│  "We sent you a magic link" │  ← Sottotitolo, text-secondary
│  "to your@email.com."       │
│                             │
│  "Didn't get it?"           │  ← Link testuale, text-secondary
│  "Send again"               │  ← Tappabile, rimanda al flusso
│                             │
└─────────────────────────────┘
```

### Schermata Link Scaduto / Errore

```
┌─────────────────────────────┐
│                             │
│         ⚠                   │  ← Icona AlertCircle (Lucide), 48px, #BFA37A
│                             │
│  "This link has expired     │
│   or is invalid."           │
│                             │
│  ┌───────────────────────┐  │
│  │  your@email.com       │  │  ← Input email (pre-compilata se disponibile)
│  └───────────────────────┘  │
│  ┌───────────────────────┐  │
│  │   Send a new link     │  │  ← CTA
│  └───────────────────────┘  │
│                             │
└─────────────────────────────┘
```

---

## Specifiche Componenti

### Input Email

```
bg-[#1F4A50]
border border-[#7DA3A0]/30
rounded-xl px-4
min-h-[48px]
text-[#EAEAEA] text-[16px]
placeholder: text-[#B9C0C1]
focus: border-[#7DA3A0] outline-none
```

### CTA Primaria — "Send me a link"

```
bg-[#7DA3A0]
text-[#0F2F33]
font-medium text-[16px]
w-full rounded-xl min-h-[48px]
Disabled: opacity-40 cursor-not-allowed (se email vuota o non valida)
Loading: spinner inline + opacity-60
```

### Google OAuth Button

```
bg-[#1F4A50]
border border-[#7DA3A0]/30
text-[#EAEAEA] text-[16px] font-medium
w-full rounded-xl min-h-[48px]
Logo Google SVG inline (colori originali), 20px
```

### Divider "or"

```
Linea orizzontale: border-[#1F4A50]
Label: text-xs text-[#B9C0C1] px-3 bg-[#0F2F33]
```

---

## Validazione Email

- L'email deve contenere `@` e un dominio valido
- Validazione client-side prima del submit (nessuna chiamata API con email malformata)
- Errore inline sotto l'input: `text-[#E24A4A] text-sm`
- Messaggi di errore:
  - Email vuota → CTA disabilitata (nessun messaggio)
  - Formato non valido → *"Please enter a valid email address."*
  - Errore Supabase (rate limit, ecc.) → *"Something went wrong. Please try again."*

---

## Stati UI

| Stato | Comportamento |
|---|---|
| Input vuoto | CTA "Send me a link" disabilitata |
| Email valida inserita | CTA abilitata |
| Loading (invio magic link) | CTA: spinner + opacity-60, input disabilitato |
| Email inviata | Transizione a schermata conferma |
| Google OAuth loading | Bottone Google: opacity-60 + spinner |
| Google OAuth successo | Redirect (Onboarding o Dashboard) |
| Google OAuth cancellato dall'utente | Ritorno alla Welcome screen, nessun errore |
| Google OAuth errore | Messaggio inline: *"Google login failed. Please try again."* |
| Link scaduto | Schermata errore con CTA per nuovo link |
| Sessione valida al riavvio | Skip Welcome screen → Dashboard |

---

## Edge Case

- Utente usa Google con email diversa da quella del magic link → due account separati (comportamento Supabase default)
- Utente usa Google con la stessa email del magic link → Supabase collega le identità automaticamente (`identity linking`)
- Email non ricevuta → link "Didn't get it? Send again" visibile nella schermata di conferma
- Utente tappa il magic link su un device diverso da quello dove ha avviato il flusso → autenticazione funziona (token-based, non device-bound)
- Popup Google bloccato dal browser → fallback a redirect-based OAuth
- Tentativo di accesso a route protetta senza sessione → redirect a Welcome screen con URL originale salvato per post-login redirect
- Sessione scaduta durante l'uso dell'app → Supabase client rinnova automaticamente il token se il refresh token è valido; altrimenti redirect silenzioso alla Welcome screen
- Rate limiting Supabase sul magic link (max X email/ora per stesso indirizzo) → errore inline generico senza esporre dettagli tecnici
- Account cancellato tenta il login → Supabase restituisce errore → Welcome screen (comportamento standard)

---

## Sicurezza

| Aspetto | Implementazione |
|---|---|
| Token magic link | One-time use, scadenza configurabile in Supabase (default: 1 ora) |
| Google OAuth | Gestito interamente da Supabase + Google, nessun token maneggiato lato client |
| Sessione persistente | Supabase gestisce refresh token in localStorage / secure storage |
| Cancellazione account | Solo via Edge Function con `service_role` key — mai lato client |
| Route protection | Middleware/guard su ogni route autenticata |
| RLS Supabase | Abilitato su tutte le tabelle — ogni query è scoped all'`auth.uid()` corrente |

---

## Acceptance Criteria

- [ ] La Welcome screen mostra input email, CTA magic link e bottone Google
- [ ] La CTA "Send me a link" è disabilitata con email vuota o non valida
- [ ] L'invio del magic link mostra la schermata di conferma con l'email usata
- [ ] Il magic link autentica l'utente e redirige correttamente (Onboarding se nuovo, Dashboard se esistente)
- [ ] Il link scaduto mostra la schermata di errore con CTA per nuovo link
- [ ] "Continue with Google" avvia il flusso OAuth e redirige correttamente
- [ ] Utente esistente che ri-accede con Google (stessa email) → nessun duplicato, identità collegata
- [ ] Al riavvio dell'app con sessione valida, l'utente accede direttamente alla Dashboard
- [ ] Route protette redirigono alla Welcome screen se non autenticato
- [ ] "Sign out" in Settings invalida la sessione e redirige alla Welcome screen
- [ ] "Delete account" + "Delete permanently" elimina account e dati con cascade e redirige alla Welcome screen
- [ ] Nessun messaggio motivazionale o valutativo in nessuna schermata di questo flusso

---

## Copy UI

| Elemento | Stringa |
|---|---|
| Tagline Welcome | `"Observe your direction."` |
| Placeholder email | `"your@email.com"` |
| CTA magic link | `"Send me a link"` |
| Caption sotto i bottoni | `"No password needed."` |
| Divider | `"or"` |
| Bottone Google | `"Continue with Google"` |
| Titolo conferma email | `"Check your email."` |
| Sottotitolo conferma | `"We sent you a magic link to"` + email |
| Link re-invio | `"Didn't get it? Send again"` |
| Titolo link scaduto | `"This link has expired or is invalid."` |
| CTA link scaduto | `"Send a new link"` |
| Errore email non valida | `"Please enter a valid email address."` |
| Errore generico | `"Something went wrong. Please try again."` |
| Errore Google OAuth | `"Google login failed. Please try again."` |
| Titolo modale delete | `"Delete account"` |
| Corpo modale delete | `"This will permanently delete all your observation history. This cannot be undone."` |
| CTA modale confirm | `"Delete permanently"` |
| CTA modale cancel | `"Cancel"` |

---

## Note UI

- Nessuna menzione di "Create account" o "Sign up" — il flusso è unificato
- Nessun tono motivazionale nelle schermate di errore
- Il logo Google nel bottone OAuth usa i colori ufficiali Google (SVG inline) — non re-colorizzato
- Brand tokens: `brand-system/betonme_brand_system_lovable.md`
- Anti-pattern da rispettare: sezione 10 del brand system

---

## Stories

- `story-00-01` — Welcome screen con input email, CTA magic link e bottone Google
- `story-00-02` — Flusso magic link: invio, schermata conferma, gestione link scaduto
- `story-00-03` — Flusso Google OAuth: avvio, callback, gestione errori
- `story-00-04` — Persistenza sessione e route guard su schermate protette
- `story-00-05` — Logout da Settings
- `story-00-06` — Delete account con Edge Function e cascade
