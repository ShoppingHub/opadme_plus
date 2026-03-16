# Stories — Epic 02 — Home (Hub Giornaliero)

## Sequenza di implementazione

```
story-02-01 → Layout Home: header + selettore settimanale + lista attività       ✅ completata
story-02-02 → Stato check-in per ciascuna area (caricamento + ottimistico)        ✅ completata
story-02-03 → Retroattività: marcare e annullare check-in su giorni passati       ✅ completata
story-02-04 → Note per attività: icona, testo libero, salvataggio per area+giorno ✅ completata
story-02-05 → CTA Gym: "Apri scheda" → naviga Area Detail gym                     ✅ completata
story-02-06 → Indicatore gym day nella Home                                        ✅ completata
story-02-07 → Empty state Home (nessuna area → CTA Attività)                      ✅ completata
story-02-08 → QuantityCounter in Home per aree quantity_reduce                    ✅ completata
story-02-09 → WeekBanner: dialogo calendario mensile per navigazione rapida       ✅ completata (extra)
story-02-10 → Google Tasks auto-sync al caricamento Home                          ✅ completata (extra)
```

> **Prerequisiti:** Richiede story-09-04 (refactor nav, route `/activities`) e Epic 03 (CheckInButton).

> **Story precedenti 02-01..02-05 (grafico aggregato) sono deprecate.** Il comportamento è stato spostato in Epic 12 (Progress). L'attuale `src/pages/Index.tsx` va riscritto.

---

## story-02-01 — Layout Home con selettore settimanale e lista attività

opad.me è un'app mobile-first di osservazione del benessere. Riscrivi la Home (route `/`) come hub giornaliero con selezione del giorno.

**Rimuovi** il grafico aggregato attuale (`src/pages/Index.tsx`) e i componenti `TrajectoryCard.tsx` e `TrajectoryCardSkeleton.tsx` che sono dead code.

**Header:**
- Wordmark `"opad.me"` a sinistra (`opad` in bianco, `.me` in `#B5453A`)

**Selettore settimanale:**
- Row con due frecce `‹` `›` ai lati e 7 chip al centro, uno per ogni giorno della settimana
- Ogni chip è un quadrato con bordi stondati — mostra il giorno abbreviato sopra e il numero sotto (es. `Lu` / `10`)
- Tap su una freccia → salta di 7 giorni interi (settimana precedente o successiva)
- Tap su un chip → seleziona quel giorno e aggiorna la lista sottostante
- Giorno di default all'apertura: oggi
- I chip dei giorni futuri (dopo oggi) sono grayed out e non tappabili
- Il chip di oggi ha uno stile distinto (es. bordo o background diverso)
- I chip dei giorni con almeno un check-in hanno un piccolo indicatore visivo (es. punto sotto il numero)
- Non è possibile navigare oltre la settimana corrente verso il futuro

**Lista attività:**
- Una card per ciascuna area attiva dell'utente (non archiviate) per il giorno selezionato
- Ogni card contiene:
  - Nome area (testo principale)
  - CTA `"Fatto"` (IT) / `"Done"` (EN) — bottone check-in
  - Icona note (outline) in fondo a destra della card
- Background card: `bg-[#1F4A50] rounded-xl p-4`
- Ordine: ordine di creazione area

---

## story-02-02 — Stato check-in per ciascuna area

Continua la Home di opad.me. Implementa il caricamento e l'aggiornamento dello stato check-in per ogni area, per il giorno attualmente selezionato nel selettore settimanale.

**Al caricamento o al cambio giorno:**
- Query a Supabase: `checkins` dove `user_id = currentUser` AND `date = giornoSelezionato`
- Per ogni area, determina se il check-in è completato
- Il bottone mostra lo stato corretto senza attendere l'interazione utente

**Loading state:**
- Durante la query: skeleton `animate-pulse` per ogni card (`bg-[#1F4A50] rounded-xl h-20`)

**Dopo tap su "Fatto":**
- Aggiornamento ottimistico: il bottone passa immediatamente a `"Osservato ✓"` (IT) / `"Observed ✓"` (EN)
- La scrittura avviene in background su Supabase
- Se la scrittura fallisce: rollback visivo, il bottone torna a "Fatto"

**Tutto loggato per il giorno selezionato:**
- Se tutte le aree sono in stato "Osservato":
  - IT: `"Tutto registrato per questo giorno."` — testo piccolo centrato, `text-[#B9C0C1]`
  - EN: `"All logged for this day."`

---

## story-02-03 — Retroattività: marcare e annullare check-in su giorni passati

Continua la Home di opad.me. Aggiungi la possibilità di gestire i check-in dei giorni passati.

**Marcare un'attività su un giorno passato:**
- L'utente seleziona un giorno passato nel selettore settimanale
- Vede la lista attività con lo stato reale di quel giorno
- Tap su `"Fatto"` su un'area non ancora loggata → crea il check-in con `date = giornoSelezionato`
- Aggiornamento ottimistico, stessa logica di story-02-02

**Annullare un check-in già registrato:**
- Se un'area è in stato `"Osservato ✓"` → il bottone è tappabile anche su giorni passati
- Tap su `"Osservato ✓"` → mostra una micro-conferma inline (es. `"Annullare?"` con Sì / No) oppure annulla direttamente con aggiornamento ottimistico
- Il check-in viene eliminato da Supabase (`delete` sul record `checkins`)
- Il bottone torna a stato `"Fatto"`
- La nota associata a quell'occorrenza **non viene cancellata** automaticamente

**Giorni futuri:**
- I chip dei giorni futuri rimangono non tappabili
- I CTA nella lista non sono interattivi se per qualche motivo venisse mostrata una lista di un giorno futuro

---

## story-02-04 — Note per attività

Continua la Home di opad.me. Aggiungi la possibilità di inserire note testuali libere per ogni attività del giorno selezionato.

**Icona note nella card:**
- In fondo a destra di ogni area card, un'icona note (matita o foglio — Lucide)
- Se non esiste ancora una nota per quella occorrenza (`area_id + date`): icona outline
- Se esiste già una nota: icona filled (o colore distinto) — segnala visivamente che c'è contenuto

**Interazione — apertura:**
- Tap sull'icona → espande la card inline mostrando un campo testo (textarea)
- Se esiste già una nota, il testo è precompilato
- Il campo mostra un contatore caratteri residui (es. `"1500"` → `"1487"` man mano che si scrive)
- Limite: 1500 caratteri — l'input si blocca al raggiungimento, il contatore diventa rosso

**Salvataggio:**
- Il testo viene salvato on-blur oppure con un tasto `"Salva"` / `"Save"` visibile sotto il campo
- Salvataggio su Supabase: tabella `activity_notes`, campi `area_id`, `date`, `user_id`, `content`
- Se il campo viene svuotato e salvato → il record viene eliminato (e l'icona torna outline)
- Nota vuota (nessun carattere) → non creare record

**Leggere note passate:**
- Quando l'utente naviga su un giorno passato, se esiste una nota per un'area → icona filled
- Tap sull'icona → mostra il testo in lettura (campo editabile)

---

## story-02-05 — CTA Gym: "Apri scheda"

Continua la Home di opad.me. Differenzia il comportamento del bottone CTA per le aree Gym.

**Condizione:**
- L'area è di `type = health` con nome `gym` o `palestra` (case insensitive) **e** ha una scheda configurata (`gym_programs` esiste per quest'area)

**Quando la condizione è vera:**
- Il CTA dell'area Gym mostra `"Apri scheda"` (IT) / `"Open session"` (EN) invece di `"Fatto"`
- Tap → naviga a `/activities/:gymAreaId` (Area Detail gym — Epic 11)
- Il check-in per quel giorno viene registrato automaticamente alla navigazione (non serve un tap separato su "Fatto")

**Quando la scheda non è configurata:**
- Il CTA rimane `"Fatto"` / `"Done"` — comportamento standard

**Stato "Osservato":**
- Se il check-in gym è già registrato per il giorno selezionato, il bottone mostra `"Osservato ✓"` come le altre aree

---

## story-02-06 — Indicatore gym day nella Home

Continua la Home di opad.me. Aggiungi l'indicatore del giorno scheda palestra nella card dell'area Gym.

**Condizione di visibilità:**
- L'utente ha un'area `type = health` con nome `gym` o `palestra` (case insensitive)
- L'area ha una scheda configurata
- Il check-in gym per il giorno selezionato non è ancora completato

**Quando visibile — dentro la card dell'area Gym:**
- Sotto il nome area, riga aggiuntiva: `"Giorno 2 — Gambe →"` (IT) / `"Day 2 — Legs →"` (EN)
- Stile: `text-sm text-[#B9C0C1]`
- Tap sulla riga → naviga a `/activities/:gymAreaId`

**Logica "giorno successivo":**
- Stesso algoritmo dell'Epic 11: giorno successivo all'ultima sessione completata, rotazione ciclica
- Se nessuna sessione precedente → Giorno 1

---

## story-02-07 — Empty state Home

Continua la Home di opad.me. Implementa l'empty state per utenti senza aree attive.

**Quando appare:** nessuna area nella tabella `areas` per l'utente (o tutte archiviate).

**Cosa mostra:**
- Icona `Eye` (Lucide), 48px, `#7DA3A0`
- IT: `"Cosa vuoi osservare?"` / EN: `"What do you want to observe?"`
- IT: `"Aggiungi un'area in Attività per iniziare."` / EN: `"Add an area in Activities to start observing."`
- CTA IT: `"Vai ad Attività"` / EN: `"Go to Activities"` → naviga a `/activities`

---

## story-02-08 — QuantityCounter in Home per aree quantity_reduce ⏳

Continua la Home di opad.me. Integra il `<QuantityCounter>` nella lista attività per le aree con `tracking_mode = quantity_reduce` e `show_quick_add_home = true`.

**Condizione di attivazione:**
- L'area ha `tracking_mode = 'quantity_reduce'` AND `show_quick_add_home = true`

**Cosa mostra nella card (al posto del bottone "Fatto"):**
- Riga superiore: nome area + `"today: N"` in `text-[#B9C0C1]` sul lato destro
- Riga inferiore: bottone **–** · numero N (tappabile per modifica manuale) · bottone **+1**

**Behavior:**
- Tap **+1** → totale +1 ottimistico, salva su Supabase (`habit_quantity_daily`)
- Tap **–** → totale –1 (minimo 0)
- Tap sul numero → input numerico inline, salva su blur o "Done"
- Il totale viene caricato al cambio giorno dal selettore settimanale
- Giorni passati: completamente interattivi
- Giorno futuro: counter non interattivo (ma visibile in read-only se il giorno è mostrato)

**Quando `show_quick_add_home = false`:**
- La card non mostra il QuantityCounter
- Tap sulla card → naviga ad Area Detail dell'area

**Stile:**
- La card mantiene lo stesso stile delle altre card (`bg-[#1F4A50] rounded-xl p-4`)
- I bottoni + e – hanno `min-height: 44px`
- Il tono dei bottoni è warm (`#BFA37A`) — coerente con il colore Reduce nel brand system
- La card è visivamente distinta dalle card binarie ma non più prominente

**Cosa NON fare:**
- Non aggiungere icona note a queste card nella Home
- Non mostrare indicatori di tendenza (frecce su/giù, colori di performance)
- Non aggiungere messaggi di feedback valutativi
- Non dominare visivamente la Home — la card deve rimanere discreta

---

## story-02-09 — WeekBanner: dialogo calendario mensile ✅

Continua la Home di opad.me. Aggiungi un banner interattivo sopra il WeekSelector che, al tap, apre un dialogo calendario mensile per navigazione rapida tra le settimane.

**Cosa mostra il banner:**
- Testo che indica l'offset della settimana rispetto a oggi (es. "Questa settimana", "1 settimana fa", "2 settimane fa")
- Tap → apre un dialogo con calendario mensile

**Dialogo calendario:**
- Header con mese/anno + frecce prev/next mese
- Griglia calendario con evidenziazione della settimana selezionata
- Calcolo settimane ISO
- CTA "Oggi / Today" → torna alla settimana corrente
- CTA "Vai / Go" → chiude e naviga alla settimana selezionata
- Supporto locale IT/EN per nomi mesi e giorni

---

## story-02-10 — Google Tasks auto-sync al caricamento Home ✅

Continua la Home di opad.me. Integra la sincronizzazione automatica con Google Tasks.

> **Riferimento completo:** vedi `epics/google-tasks/epic-13-google-tasks.md` (Story 13-08).

**Al mount della Home:**
- Se l'utente ha Google collegato → fire-and-forget call a Edge Function `pull-google-tasks`
- Nessun loading state visibile, nessun errore mostrato

**Al check-in su area con sync attiva:**
- Fire-and-forget call a Edge Function `sync-google-tasks` con `{ area_id, date, completed }`
- Nessun impatto sull'UX del check-in locale
