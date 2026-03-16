# Stories — Epic 12 — Progress

## Sequenza di implementazione

```
story-12-01 → Layout Progress: grafico aggregato + TimeRangeSelector          ✅ completata
story-12-02 → MacroAreaSelector: filtro per macro-area                        ✅ completata
story-12-03 → Empty state e loading state                                     ✅ completata
```

> **Dipendenze:** Richiede story-09-04 (route `/progress` nella nav). Il codice del grafico aggregato esiste già in `src/pages/Index.tsx` e può essere estratto.

---

## story-12-01 — Layout Progress con grafico aggregato

opad.me è un'app mobile-first di osservazione del benessere. Crea la schermata Progress (route `/progress`) che mostra l'andamento storico globale dell'utente.

**Estrai il grafico aggregato** da `src/pages/Index.tsx` e spostalo in un nuovo file `src/pages/Progress.tsx`.

**Cosa mostra:**
- Header: `"Progress"` (stesso in IT e EN)
- Selettore time range: pill `"15d"` · `"1m"` · `"3m"` · `"6m"` · `"1y"` · `"All"` — default `"1m"`
  - IT: `"15g"` · `"1m"` · `"3m"` · `"6m"` · `"1a"` · `"Tutto"`
  - EN: `"15d"` · `"1m"` · `"3m"` · `"6m"` · `"1y"` · `"All"`
  - Mappatura giorni: 15, 30, 90, 180, 365, 3650
- Grafico Recharts `<LineChart>` di altezza 60vh:
  - Dati: media mobile dei `cumulative_score` di tutte le aree attive dell'utente
  - Colore linea calcolato da slope ultimi 7 giorni: `#7DA3A0` (salita) / `#8C9496` (neutro) / `#BFA37A` (discesa)
  - Griglia solo orizzontale, `opacity-10`
  - Asse Y nascosto, asse X con tick date, 12px, `#B9C0C1`
  - Background card: `bg-[#1F4A50] rounded-xl p-4`
- Sotto il grafico: `<MacroAreaSelector>` (story-12-02)

**Questa schermata non contiene azioni.** Nessun bottone di check-in, nessun link ad aree.

---

## story-12-02 — MacroAreaSelector in Progress

Continua Epic 12 di opad.me. Aggiungi il selettore macro-area sotto il grafico in Progress.

**Componente `<MacroAreaSelector>`** (può riusare quello esistente da Index.tsx):
- Row di pill scorrevole orizzontalmente
- 5 opzioni nella lingua dell'utente:
  - IT: `Tutto · Salute · Studio · Riduci · Finanze`
  - EN: `All · Health · Study · Reduce · Finance`
- Default: `Tutto / All` attivo
- Pill attiva: background `#7DA3A0`, testo scuro
- Pill inattiva: background trasparente, bordo sottile, testo `#B9C0C1`

**Behavior:**
- Tap macro-area → grafico si aggiorna con la media dei `cumulative_score` delle sole aree di quel tipo
- Tap `Tutto / All` → torna alla media di tutte le aree
- Se nessun dato per la macro-area selezionata: messaggio IT `"Nessun dato per questa categoria"` / EN `"No data for this category yet"` — nessuna linea piatta, nessun errore
- La selezione rimane quando si cambia il time range

---

## story-12-03 — Empty state e loading state

Continua Epic 12 di opad.me. Aggiungi gli stati speciali della schermata Progress.

**Empty state** (utente senza aree attive):
- Icona `Eye` (Lucide), 48px, `#7DA3A0`
- IT: `"Nessun dato ancora."` / EN: `"No data yet."`
- IT: `"Aggiungi un'area in Attività e inizia a registrare."` / EN: `"Add an area in Activities and start logging."`
- CTA IT: `"Vai ad Attività"` / EN: `"Go to Activities"` → naviga a `/activities`

**Loading state** (query Supabase in corso):
- Skeleton `animate-pulse bg-[#1F4A50] rounded-xl h-[60vh]` al posto del grafico
- Skeleton di 5 pill muted al posto del MacroAreaSelector
- Il TimeRangeSelector rimane visibile ma non interattivo
