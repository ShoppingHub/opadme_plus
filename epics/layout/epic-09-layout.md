# Epic 09 — Layout Responsivo (Mobile Bottom Nav · Desktop Sidebar)

## Obiettivo
Adattare la navigazione principale al dispositivo: bottom navigation su mobile, sidebar laterale fissa su desktop — con 4 tab fisse e supporto architetturale per un 5° tab opzionale.

---

## Behavior

La navigazione è **fissa e uguale per tutti gli utenti**: 4 tab in ordine invariato. Non ci sono slot configurabili. Non ci sono tab opzionali.

Su mobile il bottom nav è fisso in basso. Su desktop la navigazione migra sulla colonna sinistra come sidebar fissa. Le voci di navigazione sono identiche su entrambi i layout.

> **Nota (v2):** Il 5° tab opzionale (Finance) è stato rimosso. Con l'introduzione del sistema Schede (Epic 14), Finance Projection è diventata una scheda con pagina dedicata (`/cards/finance`), accessibile dalla sezione Attività. La nav resta fissa a 4 tab.

---

## Layout Mobile (< 1024px)

```
┌─────────────────────────────┐
│ Header                      │
├─────────────────────────────┤
│                             │
│   Contenuto pagina          │
│                             │
├─────────────────────────────┤
│  Home · Attività · Progress · ⚙  │  ← Bottom nav fissa 56px (4 tab default)
└─────────────────────────────┘
```

---

## Layout Desktop (≥ 1024px)

```
┌──────────────┬──────────────────────────────┐
│              │ Header pagina                │
│  [Logo]      ├──────────────────────────────┤
│              │                              │
│  Home        │   Contenuto pagina           │
│  Attività    │                              │
│  Progress    │                              │
│              │                              │
│  ──────────  │                              │
│  Impostazioni│                              │
│              │                              │
└──────────────┴──────────────────────────────┘
```

### Specifiche sidebar desktop

| Proprietà | Valore |
|---|---|
| Larghezza | 240px fissa |
| Background | `#0F2F33` |
| Logo / nome app | In cima alla sidebar, `opad.me` |
| Voci nav | Icona + label testuale (su mobile solo icona) |
| Icone | Lucide outline 20px |
| Colore voce attiva | `#7DA3A0` |
| Colore voce inattiva | `#B9C0C1` |
| Impostazioni | In fondo alla sidebar, separato da divider |
| Contenuto principale | `margin-left: 240px`, padding laterale 32px |
| Max width contenuto | 900px centrato |

---

## Struttura tab

| Posizione | Tab | Route | Tipo |
|---|---|---|---|
| 1 | Home | `/` | Fissa |
| 2 | Attività / Activities | `/activities` | Fissa |
| 3 | Progress | `/progress` | Fissa |
| Ultima | Impostazioni / Settings | `/settings` | Fissa, sempre separata da divider su desktop |

Le 4 tab fisse non sono mai nascoste, mai configurabili. Non ci sono tab opzionali.

> Le schede (Gym, Finance Projection) hanno le proprie route dedicate (`/cards/gym`, `/cards/finance`) ma non appaiono nella nav — sono accessibili dalla pagina Attività (Epic 14).

---

## Hook `useNavConfig`

Sostituisce il vecchio `useMenuConfig`. Fornisce ai componenti nav la lista delle voci visibili.

```typescript
interface NavItem {
  id: string
  route: string
  icon: LucideIcon
  labelIT: string
  labelEN: string
  visible: boolean
  isLast?: boolean       // per il divider su desktop
}

// Voci di default
const DEFAULT_NAV: NavItem[] = [
  { id: 'home',       route: '/',           icon: Home,         labelIT: 'Home',          labelEN: 'Home',     visible: true },
  { id: 'activities', route: '/activities', icon: LayoutGrid,   labelIT: 'Attività',      labelEN: 'Activities', visible: true },
  { id: 'progress',   route: '/progress',   icon: TrendingUp,   labelIT: 'Progress',      labelEN: 'Progress', visible: true },
  { id: 'settings',   route: '/settings',   icon: Settings,     labelIT: 'Impostazioni',  labelEN: 'Settings', visible: true, isLast: true },
]
// Nota: la voce Finance è stata rimossa dalla nav.
// Finance Projection è ora una scheda con pagina dedicata /cards/finance (Epic 14).
```

Il componente nav esegue solo:
```typescript
navConfig.visibleItems.map(item => <NavItem key={item.id} {...item} />)
```

Nessuna logica condizionale nei componenti nav.

---

## Breakpoint

| Breakpoint | Layout |
|---|---|
| < 1024px | Bottom navigation mobile |
| ≥ 1024px | Sidebar desktop |

> Usare `lg:` breakpoint di Tailwind (1024px).

---

## Stati UI

| Stato | Mobile | Desktop |
|---|---|---|
| Voce attiva | Icona `#7DA3A0`, label sotto | Riga intera highlight, icona + label `#7DA3A0` |
| Voce inattiva | Icona `#B9C0C1` | Icona + label `#B9C0C1` |

---

## Edge Case

- Viewport ridimensionato durante la sessione → layout si adatta senza reload
- Sidebar desktop non collassabile (MVP)

---

## Acceptance Criteria

- [x] Su viewport < 1024px: solo bottom nav in basso
- [x] Su viewport ≥ 1024px: solo sidebar laterale sinistra (240px)
- [ ] La nav mostra esattamente: Home · Attività · Progress · Impostazioni (4 tab fisse, nessun tab opzionale)
- [ ] Impostazioni è sempre separata da divider su desktop
- [ ] La voce attiva è evidenziata con `#7DA3A0`
- [ ] Resize aggiorna il layout dinamicamente (no reload)
- [ ] `useNavConfig` fornisce l'array fisso; i componenti nav non contengono logica condizionale
- [ ] Le route `/cards/*` sono registrate nel router ma non nella nav

---

## Stato implementazione

**Parzialmente completato** — componenti nav esistenti (BottomNav, DesktopSidebar) da refactorare per 4 tab fisse.

| Componente | File | Stato |
|---|---|---|
| Layout shell | `src/components/AppLayout.tsx` | Completato |
| Bottom nav mobile | `src/components/BottomNav.tsx` | Da aggiornare: 4 tab fisse |
| Sidebar desktop | `src/components/DesktopSidebar.tsx` | Da aggiornare: 4 tab fisse |
| Menu config | `src/hooks/useMenuConfig.tsx` | Da sostituire con `useNavConfig` |

---

## Migration DB

```sql
-- Rimuovere colonna obsoleta
ALTER TABLE users DROP COLUMN IF EXISTS menu_custom_items;

-- extra_tab_enabled è stata rimossa e sostituita dalla tabella user_cards (Epic 14)
ALTER TABLE users DROP COLUMN IF EXISTS extra_tab_enabled;
```

> La migrazione completa (creazione `user_cards`, trasferimento dati) è definita in Epic 14 (story-14-01).

---

## Dependencies

- Epic 07 (Settings) — preferenze utente
- Epic 08 (i18n) — label IT/EN per tutte le voci nav
- Epic 14 (Schede) — route dedicate `/cards/*` registrate nel router ma non nella nav

---

## Stories

- `story-09-01` — Sidebar desktop con logo, voci fisse e settings in fondo — **completata**
- `story-09-02` — Adattamento responsivo: bottom nav su mobile, sidebar su desktop — **completata**
- `story-09-03` — Voci custom nella sidebar da configurazione Settings — **completata** *(da refactorare in 09-04)*
- `story-09-04` — Refactor nav: 4 tab fisse via `useNavConfig`, rimozione `useMenuConfig` — **da fare**
- `story-09-05` — ~~Optional 5th tab~~ → **rimossa** — sostituita dal sistema Schede (Epic 14). Le schede hanno route dedicate `/cards/*` senza tab nella nav.
