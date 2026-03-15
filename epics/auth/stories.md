# Stories — Epic 00 — Autenticazione

## Sequenza di implementazione

```
story-00-01 → Welcome screen                                              ✅ completata (aggiornata: email/password)
story-00-02 → Email/password login + signup (sostituisce magic link)        ✅ completata
story-00-03 → Google OAuth                                                  ✅ completata
story-00-04 → Session persistence + route guard                             ✅ completata
story-00-05 → Logout                                                        ✅ completata
story-00-06 → Delete account                                                ✅ completata
story-00-07 → Password reset flow                                           ✅ completata (extra)
story-00-08 → Demo mode (accesso senza auth)                                ✅ completata (extra)
```

> **Dipendenze:** Richiede scaffold iniziale completato (route guard base, connessione Supabase).

---

## story-00-01 — Welcome screen ✅

opad.me è un'app di osservazione del benessere personale. Costruisci la Welcome screen (route `/login`).

**Cosa mostra:**
- Logo/wordmark "opad.me" centrato (`opad` bianco, `.me` terracotta `#B5453A`)
- Tagline: `"Observe your direction."` (colore secondario)
- Tab switcher: `"Login"` / `"Sign up"`
- Input email (placeholder: `"your@email.com"`) — validazione Zod
- Input password (con toggle visibilità, min 6 caratteri)
- CTA primario full-width: `"Sign in"` (login) / `"Create account"` (signup)
- Divider con label `"or"`
- Bottone secondario: `"Continue with Google"` con logo Google SVG
- Link `"Forgot password?"` → schermata reset (story-00-07)
- Bottone `"Try demo"` / `"Prova demo"` → demo mode (story-00-08)

**Behavior:**
- CTA disabilitata se email vuota/invalida o password < 6 chars
- Se l'utente è già autenticato → redirect a `/`
- Doppio submit bloccato da loading state

**Note:**
- Il logo Google usa i colori originali
- Testi localizzati IT/EN

---

## story-00-02 — Email/password login + signup ✅

Continua l'Epic 00 di opad.me. Implementa il flusso email/password.

**Login:**
- Tap su `"Sign in"` → `supabase.auth.signInWithPassword({ email, password })`
- Successo → redirect a `/` (o `/onboarding` se primo accesso)
- Errore credenziali → messaggio inline: `"Invalid email or password."`

**Signup:**
- Tap su `"Create account"` → `supabase.auth.signUp({ email, password })`
- Successo → schermata conferma email:
  - Icona Mail (Lucide), 48px
  - Titolo: `"Check your email."`
  - Sottotitolo: `"We sent a confirmation link to"` + email
  - Link: `"Didn't get it? Send again"`

**Callback auth (route `/auth/callback`):**
- Gestisce il ritorno da email di conferma e OAuth
- Se primo accesso (nessuna area in `areas`) → redirect a `/onboarding/areas`
- Se account esistente → redirect a `/`

**Errore generico Supabase** (es. rate limit): `"Something went wrong. Please try again."` — messaggio inline, non modal

**Nota:** Il flusso magic link originale è stato sostituito da email/password nell'implementazione. Il signup richiede conferma email.

---

## story-00-03 — Google OAuth

Continua l'Epic 00 di opad.me. Aggiungi il flusso Google OAuth.

**Behavior:**
- Tap su `"Continue with Google"` → Supabase apre il flusso OAuth Google (popup o redirect)
- L'utente seleziona/conferma il proprio account Google
- Supabase autentica la sessione
- Se primo accesso → redirect a `/onboarding`
- Se account esistente → redirect a `/`

**Stati bottone Google durante il flusso:**
- Loading: opacity ridotta + spinner
- Cancellato dall'utente → ritorno alla Welcome screen, nessun messaggio di errore
- Errore OAuth → messaggio inline: `"Google login failed. Please try again."`

**Note:**
- Se l'utente ha già un account magic link con la stessa email → Supabase collega le due identità automaticamente (identity linking), nessun duplicato
- Se il popup è bloccato dal browser → Supabase usa il redirect-based OAuth come fallback

---

## story-00-04 — Persistenza sessione e route guard

Continua l'Epic 00 di opad.me. Implementa la persistenza della sessione e il route guard su tutte le schermate autenticate.

**Persistenza sessione:**
- Al riavvio dell'app, Supabase controlla il token in localStorage
- Sessione valida → accesso diretto a `/` (nessun login ripetuto)
- Sessione scaduta o assente → redirect a `/login`
- Supabase gestisce il rinnovo automatico del refresh token

**Route guard:**
- Le route `/`, `/areas`, `/finance`, `/settings` sono protette
- Tentativo di accesso senza sessione → redirect a `/login`
- Dopo il login, redirect alla route originariamente richiesta (o `/` se non specificata)

**Sessione scaduta durante l'uso:**
- Supabase client tenta il rinnovo automatico
- Se fallisce → redirect silenzioso a `/login` (nessun messaggio di errore intrusivo)

---

## story-00-05 — Logout

Continua l'Epic 00 di opad.me. Aggiungi il logout dalla schermata Settings.

**Behavior:**
- Bottone `"Sign out"` nella schermata `/settings`
- Tap → Supabase invalida la sessione (`supabase.auth.signOut()`)
- Redirect immediato a `/login`
- Nessun modale di conferma. Nessun messaggio di addio.

**Stato loading:**
- Il bottone mostra opacity ridotta durante la chiamata Supabase

---

## story-00-06 — Delete account

Continua l'Epic 00 di opad.me. Aggiungi la cancellazione account da Settings.

**Behavior:**
- Bottone `"Delete account"` nella schermata `/settings` (stile testo in colore errore `#E24A4A`)
- Tap → si apre un modale di conferma

**Modale di conferma:**
- Titolo: `"Delete account"`
- Corpo: `"This will permanently delete all your observation history. This cannot be undone."`
- CTA conferma: `"Delete permanently"` (testo in colore errore)
- CTA annulla: `"Cancel"` (chiude il modale, nessuna azione)

**Azione al confirm:**
- Chiamata a una Supabase Edge Function con `service_role` che elimina l'utente da `auth.users` con cascade su: `areas`, `checkins`, `score_daily`, `users`
- Sessione invalidata
- Redirect a `/login`
- Nessun messaggio di addio

**Stato loading:**
- CTA `"Delete permanently"` mostra spinner + opacity ridotta durante la chiamata
- Se la connessione cade durante il delete → errore inline non bloccante, nessuna azione eseguita

---

## story-00-07 — Password reset flow ✅

Continua Epic 00 di opad.me. Implementa il flusso di reset password.

**Accesso:** link `"Forgot password?"` nella schermata Login.

**Schermata forgot password:**
- Input email (pre-compilata se disponibile)
- CTA `"Send reset link"` → chiama `supabase.auth.resetPasswordForEmail()`
- Stato loading: spinner + opacity ridotta

**Schermata conferma:**
- Icona Mail, titolo `"Check your email"`
- Sottotitolo con email inserita
- Link `"Didn't get it? Send again"`

**Pagina reset password (route `/reset-password`):**
- Raggiunta dal link nell'email
- Input nuova password (con toggle visibilità)
- CTA `"Update password"` → aggiorna password via Supabase
- Redirect a `/` dopo successo

---

## story-00-08 — Demo mode ✅

Continua Epic 00 di opad.me. Aggiungi la modalità demo per accesso senza autenticazione.

**Accesso:** bottone `"Try demo"` / `"Prova demo"` nella schermata Login.

**Behavior:**
- Tap → attiva demo mode via `sessionStorage`
- Bypass del route guard (ProtectedRoute mostra contenuto anche senza sessione)
- Dati generati da `lib/demoData.ts` (deterministici, con pattern realistici)
- Tutte le feature di scrittura sono disabilitate o simulate in locale
- Google Tasks section nascosta
- Profile avatar mostra icona generica

**Uscita demo:**
- Tap su `"Sign out"` in Settings → disabilita demo mode → redirect a Login
- Chiusura tab → demo mode si perde (sessionStorage)

**Cosa NON fare:**
- Non persistere dati demo su Supabase
- Non mostrare banner "sei in demo" — l'esperienza è pulita
