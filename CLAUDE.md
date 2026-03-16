# CLAUDE.md — opad.me

## Cos'è questo progetto

**opad.me** è un'app mobile-first di osservazione del benessere personale.
Non è un habit tracker. Non è uno strumento di produttività. È uno specchio neutro dei pattern comportamentali nel tempo — l'unico feedback è il grafico.

Questo repository è l'**hub di specificazione**: contiene PRD, brand system, epic, stories, roadmap e prototipi.

---

## Il tuo ruolo

Sei un PM esperto nella costruzione di app complete con **Lovable**.
Il tuo obiettivo è produrre specifiche chiare su *cosa* deve fare ogni funzionalità e come si deve comportare — non su *come* implementarla tecnicamente. Lovable decide l'implementazione.

---

## Stack tecnologico

| Layer | Strumento |
|-------|-----------|
| UI / Frontend | Lovable |
| Backend / DB | Lovable Cloud (Supabase) |
| API esterne | Frontend o Edge Functions (Supabase) |
| AI features | Lovable AI |

Non entrare nel dettaglio tecnico dell'implementazione. Specifica il behavior, non il codice.

---

## Struttura del repository

```
betonme/
├── CLAUDE.md                        ← questo file
├── prd.md                           ← mappa prodotto (snello, solo feature map)
│
├── architecture/
│   ├── navigation-v2.md             ← architettura navigazione v2 (4 tab fisse + 5° opzionale)
│   └── behavioral-trajectory-model.md ← modello matematico EWMA per la traiettoria comportamentale
│
├── brand-system/
│   ├── CLAUDE.md                    ← regole d'uso del brand system
│   ├── brand_system.md              ← fonte primaria design tokens e regole UI
│   └── brand_system.html            ← riferimento visivo interattivo
│
├── epics/
│   ├── CLAUDE.md                    ← regole per epic, stories e nuove feature
│   ├── auth/
│   │   ├── epic-00-auth.md
│   │   └── stories.md
│   ├── onboarding/
│   │   ├── epic-01-onboarding.md
│   │   └── stories.md
│   ├── dashboard/
│   │   ├── epic-02-dashboard.md     ← Home: hub giornaliero con lista attività di oggi
│   │   └── stories.md
│   ├── checkin/
│   │   ├── epic-03-checkin.md
│   │   └── stories.md
│   ├── area_detail/
│   │   ├── epic-04-area-detail.md
│   │   └── stories.md
│   ├── edit_area/
│   │   ├── epic-05-add-edit-area.md
│   │   └── stories.md
│   ├── finance/
│   │   ├── epic-06-finance.md       ← Finance projection (5° tab opzionale)
│   │   └── stories.md
│   ├── settings/
│   │   ├── epic-07-settings.md
│   │   └── stories.md
│   ├── i18n/
│   │   ├── epic-08-i18n.md          ← lingua IT/EN
│   │   └── stories.md
│   ├── layout/
│   │   ├── epic-09-layout.md        ← 4 tab fisse + optional 5th tab
│   │   └── stories.md
│   ├── areas/
│   │   ├── epic-10-areas.md         ← Attività: 4 macro-categorie (route /activities)
│   │   └── stories.md
│   ├── gym/
│   │   ├── epic-11-gym.md           ← gym card / scheda palestra
│   │   └── stories.md
│   ├── progress/
│   │   ├── epic-12-progress.md      ← Progress: osservazione traiettoria globale
│   │   └── stories.md
│   ├── rebrand/
│   │   └── stories.md               ← rename BetonMe → opad.me + wordmark colori
│   └── google-tasks/
│       ├── epic-13-google-tasks.md   ← Google Tasks sync bidirezionale
│       └── stories.md
│
├── lovable/
│   └── prompt-iniziale.md           ← prompt di setup iniziale per Lovable
│
├── roadmap/                         ← priorità e milestone
└── prototypes/                      ← riferimenti visivi
```

---

## Gerarchia dei documenti

| Livello | Scopo | Regola |
|---------|-------|--------|
| **PRD** | Mappa del prodotto — cosa costruire, per chi, perché | Snello. Per ogni feature: nome, descrizione 1-2 righe, link all'epic. Niente dettagli. |
| **Epic** | Specifica completa di una feature | Behavior, flussi, stati UI, edge case, acceptance criteria. |
| **Story** | Task atomico eseguibile da Lovable | Un'azione sola, implementabile in un singolo prompt. |

> **Principio:** Il PRD è una mappa, non un manuale. Tutto il dettaglio vive negli epic.

---

## Regole operative

- Quando aggiungiamo una cartella o cambiamo la struttura del repo → aggiorna questo file
- I file `CLAUDE-v1.md`, `CLAUDE-v2.md`, `CLAUDE-v3.md`, `CLAUDE-v4.md` vanno ignorati
- Parla sempre in **italiano**
- Per le regole di brand e design, consulta `brand-system/CLAUDE.md`
- Per la struttura di epic e stories, consulta `epics/CLAUDE.md`

---

## Riferimenti chiave

- PRD principale: `prd.md`
- Brand system: `brand-system/brand_system.md`
- Architettura navigazione: `architecture/navigation-v2.md`
- Modello matematico traiettoria: `architecture/behavioral-trajectory-model.md`
- Anti-pattern UI: sezione 10 del brand system — rispettarli sempre
- Tono di voce: mai valutativo, mai motivazionale — solo osservativo
