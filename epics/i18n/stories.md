# Stories — Epic 08 — Lingua (IT / EN)

## Sequenza di implementazione — ✅ tutte completate

```
story-08-01 → Selettore lingua in Settings + salvataggio su Supabase          ✅
story-08-02 → Rilevamento lingua browser al primo accesso                     ✅
story-08-03 → Traduzione completa IT/EN di tutti i testi dell'app             ✅
```

> **Dipendenze:** Richiede Epic 07 (Settings) per l'UI del selettore. Le traduzioni impattano tutti gli altri epic.

---

## story-08-01 — Selettore lingua in Settings

Implementata in `story-07-02` — vedi `epics/settings/stories.md`.

---

## story-08-02 — Rilevamento lingua browser

BetonMe è un'app di osservazione del benessere. Implementa il rilevamento automatico della lingua al primo accesso.

**Behavior:**
- All'apertura dell'app, se l'utente non è ancora autenticato → rileva `navigator.language` del browser
- Se inizia con `"it"` → usa `"it"` come lingua per onboarding e login
- Qualsiasi altro valore → usa `"en"`
- Al termine dell'onboarding → salva la preferenza `language` nella tabella `users`
- Se l'utente è già autenticato → usa la preferenza salvata nel suo profilo

---

## story-08-03 — Traduzione IT/EN completa

BetonMe è un'app di osservazione del benessere. Implementa la traduzione completa di tutti i testi dell'app in italiano e inglese.

**Scope della traduzione (tutte le schermate):**

| Schermata | Elementi da tradurre |
|---|---|
| Login / Auth | Label campi, placeholder, messaggi errore, CTA |
| Onboarding | Titoli step, label aree preset, placeholder, CTA |
| Dashboard | Empty state, MacroAreaSelector labels |
| Areas | Header, label macro-aree, CTA aggiungi |
| Area Detail | Edit link, empty state, label heatmap, label score |
| Add/Edit Area | Label form, placeholder, tipo area options, CTA |
| Finance | Label sezione, label grafico |
| Settings | Tutte le label (vedi Epic 07 Copy UI) |
| Gym Card | Tutte le label (vedi Epic 11 Copy UI) |
| Bottom nav / Sidebar | Label voci navigazione |
| Modali | Titolo, corpo, CTA conferma, CTA annulla |
| Empty state globali | Tutti i testi |
| Toast / errori | Messaggi di errore non bloccanti |

**Implementazione:**
- Usa un sistema di traduzioni centralizzato (es. oggetto i18n o libreria `i18next`)
- La lingua attiva viene letta dalla preferenza utente in tempo reale
- Nessun testo hardcodato in inglese rimane quando la lingua è `"it"`
- Fallback a `"en"` se una stringa IT non è disponibile (silenzioso)
