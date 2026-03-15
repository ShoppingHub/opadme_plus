# Stories — Epic 09 — Layout Responsivo

## Sequenza di implementazione

```
story-09-01 → Sidebar desktop (≥ 1024px) con logo e voci fisse               ✅ completata
story-09-02 → Adattamento responsivo: switch automatico mobile/desktop        ✅ completata
story-09-03 → Voci custom nella sidebar da configurazione Settings            ✅ completata (da refactorare)
story-09-04 → Refactor nav: 4 tab fisse via useNavConfig                      ⏳ da fare
story-09-05 → Optional 5th tab Finance con toggle in Settings                 ⏳ da fare
```

> **Nota:** Le story 09-01/02/03 sono completate nell'implementazione attuale ma devono essere refactorate in 09-04 per allinearsi alla nuova architettura a 4 tab fisse.

---

## story-09-01 — Sidebar desktop ✅

BetonMe è un'app di osservazione del benessere. Aggiungi una sidebar di navigazione laterale che appare su viewport ≥ 1024px.

**Struttura sidebar (dall'alto verso il basso):**
- Logo / wordmark `"BetonMe"` in cima (padding top 24px)
- Voci di navigazione fisse: `Home` · `Aree / Areas`
- Divider orizzontale sottile
- `Impostazioni / Settings` in fondo

**Specifiche:**
- Larghezza fissa 240px
- Background `#0F2F33`
- Ogni voce: icona Lucide outline 20px + label testuale, altezza minima 44px, padding laterale 16px
- Voce attiva: testo e icona `#7DA3A0`, background `rgba(125,163,160,0.1)` rounded
- Voce inattiva: testo e icona `#B9C0C1`
- Il contenuto principale ha `margin-left: 240px`, padding 32px, max-width 900px centrato

---

## story-09-02 — Layout responsivo mobile/desktop ✅

Continua Epic 09 di BetonMe. Implementa lo switch automatico tra bottom nav (mobile) e sidebar (desktop).

**Behavior:**
- Viewport < 1024px → mostra solo il bottom nav fisso in basso, sidebar nascosta
- Viewport ≥ 1024px → mostra solo la sidebar laterale sinistra, bottom nav nascosto
- Il cambio avviene dinamicamente al ridimensionamento della finestra, senza reload
- Su desktop la sidebar non è collassabile (MVP — nessun toggle hamburger)

---

## story-09-03 — Voci custom nella sidebar ✅ (deprecata con 09-04)

Story completata nell'implementazione precedente. Il comportamento viene sostituito dalla story 09-04.

---

## story-09-04 — Refactor nav: 4 tab fisse

Continua Epic 09 di BetonMe. Rifai il sistema di navigazione con 4 tab fisse, rimuovendo il meccanismo degli slot custom.

**Obiettivo:** la nav è uguale per tutti gli utenti. Nessuno slot configurabile. Nessuna logica condizionale nei componenti nav.

**Crea hook `useNavConfig`** che sostituisce `useMenuConfig`. Restituisce un array di `NavItem` con `visible` boolean:
- `home` → `/` — sempre visibile
- `activities` → `/activities` — sempre visibile — label IT: `Attività` / EN: `Activities`
- `progress` → `/progress` — sempre visibile — label IT: `Progress` / EN: `Progress`
- `finance` → `/finance` — visibile solo se `extra_tab_enabled = true` (colonna su `users`)
- `settings` → `/settings` — sempre visibile, con flag `isLast: true` per il divider desktop

**Aggiorna `BottomNav.tsx`:**
- Usa `useNavConfig().visibleItems` al posto di `useMenuConfig`
- Rimuovi tutta la logica custom slots
- Mostra le voci nell'ordine dell'array

**Aggiorna `DesktopSidebar.tsx`:**
- Usa `useNavConfig().visibleItems`
- Il divider appare prima della voce con `isLast: true`
- Rimuovi tutta la logica custom slots

**Route update:** aggiorna tutti i link interni da `/areas` a `/activities` (inclusi header back nav, CTA form, redirect).

**DB migration:** aggiungi colonna `extra_tab_enabled BOOLEAN DEFAULT false` alla tabella `users`.

**Label nav (seguono lingua utente):**

| Tab | IT | EN |
|---|---|---|
| Home | `Home` | `Home` |
| Attività | `Attività` | `Activities` |
| Progress | `Progress` | `Progress` |
| Finanze (opzionale) | `Finanze` | `Finance` |
| Impostazioni | `Impostazioni` | `Settings` |

---

## story-09-05 — Optional 5th tab Finance

Continua Epic 09 di BetonMe. Implementa il toggle per abilitare il 5° tab Finance dalla schermata Impostazioni.

**Dipende da:** story-09-04 (hook `useNavConfig` con `finance.visible` già gestito)

**In Impostazioni — sezione Preferenze:**
Aggiungi toggle:
- Label IT: `Mostra sezione Finanze`
- Label EN: `Show Finance tab`
- Sottotitolo IT: `Aggiunge un accesso rapido alla proiezione finanziaria.`
- Sottotitolo EN: `Adds quick access to the finance projection.`
- Default: OFF
- Al cambio: aggiorna `extra_tab_enabled` su Supabase (silenzioso)
- Il nav si aggiorna immediatamente senza reload

**Behavior nav:**
- `extra_tab_enabled = false` (default): nav mostra Home · Attività · Progress · Impostazioni
- `extra_tab_enabled = true`: nav mostra Home · Attività · Progress · Finanze · Impostazioni

**Finanze route `/finance`:** già implementato in Epic 06. Nessuna modifica alla pagina.
