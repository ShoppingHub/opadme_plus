# Stories — Epic 00 — Autenticazione

## Sequenza di implementazione

```
story-00-01 → Welcome screen
story-00-02 → Magic link flow (invio + conferma + link scaduto)
story-00-03 → Google OAuth
story-00-04 → Session persistence + route guard
story-00-05 → Logout
story-00-06 → Delete account
```

> **Dipendenze:** Richiede scaffold iniziale completato (route guard base, connessione Supabase).

---

## story-00-01 — Welcome screen

BetonMe è un'app di osservazione del benessere personale. Costruisci la Welcome screen (route `/login`).

**Cosa mostra:**
- Logo/wordmark "BetonMe" centrato
- Tagline: `"Observe your direction."` (colore secondario)
- Input per l'email (placeholder: `"your@email.com"`)
- Bottone primario full-width: `"Send me a link"`
- Divider con label `"or"`
- Bottone secondario full-width: `"Continue with Google"` con logo Google SVG (colori originali Google, non re-colorizzato)
- Caption sotto i bottoni: `"No password needed."`

**Behavior:**
- La CTA `"Send me a link"` è disabilitata se l'email è vuota
- Validazione client-side: l'email deve contenere `@` e un dominio. Se non valida → errore inline sotto l'input: `"Please enter a valid email address."`
- Se l'utente è già autenticato e accede a `/login` → redirect a `/`

**Note:**
- Non usare le label "Sign up", "Create account", "Register" — c'è un solo entry point
- Il logo Google usa i colori originali, non viene re-colorizzato con il brand BetonMe

---

## story-00-02 — Magic link flow

Continua l'Epic 00 di BetonMe. Costruisci il flusso completo del magic link.

**Schermata di conferma email inviata:**
- Appare dopo il tap su `"Send me a link"` (chiamata Supabase riuscita)
- Icona Mail (Lucide), 48px
- Titolo: `"Check your email."`
- Sottotitolo: `"We sent you a magic link to"` + email inserita
- Link testuale: `"Didn't get it? Send again"` → re-invia il magic link alla stessa email e rimane sulla schermata di conferma
- CTA `"Send me a link"` in stato loading (spinner inline + opacity ridotta) durante l'invio

**Schermata link scaduto / non valido:**
- Si mostra quando Supabase restituisce errore sul link (token scaduto o già usato)
- Icona AlertCircle (Lucide), 48px, colore warning
- Titolo: `"This link has expired or is invalid."`
- Input email (pre-compilata con l'email dall'URL se disponibile)
- CTA: `"Send a new link"` → stesso flusso del magic link

**Callback magic link:**
- Supabase autentica la sessione al click del link nell'email
- Se primo accesso (nessuna area in tabella `areas`) → redirect a `/onboarding`
- Se account esistente → redirect a `/`

**Errore generico Supabase** (es. rate limit): `"Something went wrong. Please try again."` — messaggio inline, non modal

---

## story-00-03 — Google OAuth

Continua l'Epic 00 di BetonMe. Aggiungi il flusso Google OAuth.

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

Continua l'Epic 00 di BetonMe. Implementa la persistenza della sessione e il route guard su tutte le schermate autenticate.

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

Continua l'Epic 00 di BetonMe. Aggiungi il logout dalla schermata Settings.

**Behavior:**
- Bottone `"Sign out"` nella schermata `/settings`
- Tap → Supabase invalida la sessione (`supabase.auth.signOut()`)
- Redirect immediato a `/login`
- Nessun modale di conferma. Nessun messaggio di addio.

**Stato loading:**
- Il bottone mostra opacity ridotta durante la chiamata Supabase

---

## story-00-06 — Delete account

Continua l'Epic 00 di BetonMe. Aggiungi la cancellazione account da Settings.

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
