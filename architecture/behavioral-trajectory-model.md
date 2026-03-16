# Modello di Traiettoria Comportamentale

**opad.me — Documentazione architetturale**
Versione 1.0 · Marzo 2026

---

## Indice

1. [Filosofia e contesto](#1-filosofia-e-contesto)
2. [Perché non usare l'accumulo cumulativo](#2-perché-non-usare-laccumulo-cumulativo)
3. [Generazione del segnale giornaliero](#3-generazione-del-segnale-giornaliero)
4. [Modello di stato latente — EWMA](#4-modello-di-stato-latente--ewma)
5. [Coefficiente di adattamento α](#5-coefficiente-di-adattamento-α)
6. [Interpretazione del grafico](#6-interpretazione-del-grafico)
7. [Note di implementazione per sviluppatori](#7-note-di-implementazione-per-sviluppatori)
8. [Caso speciale: area Finance](#8-caso-speciale-area-finance)
9. [Valori di riferimento e boundary conditions](#9-valori-di-riferimento-e-boundary-conditions)

---

## 1. Filosofia e contesto

opad.me non misura la performance. Non assegna punti, non registra streak, non premia i risultati. Il prodotto è uno **specchio neutro della direzione comportamentale nel tempo**.

Il grafico è l'interfaccia primaria. Il numero è secondario e nascosto di default.

L'obiettivo del modello matematico è rispondere a una sola domanda:

> *"In che direzione sto andando, considerando il mio comportamento recente?"*

Non: "quante cose ho fatto", "quante ne ho mancate", "qual è il mio punteggio totale".

Questa distinzione cambia completamente il modello da adottare.

---

## 2. Perché non usare l'accumulo cumulativo

I tracker tradizionali usano uno **score cumulativo**:

```
score_totale = Σ(daily_score)
```

Questo approccio ha difetti strutturali che lo rendono inadatto per opad.me:

| Problema | Effetto |
|---|---|
| **Crescita esponenziale** | Dopo mesi di uso, lo score diventa così alto che le variazioni recenti diventano invisibili nel grafico |
| **Asimmetria della punizione** | Mancanze nelle prime settimane restano permanentemente nel saldo, senza possibilità di recupero reale |
| **Pressione delle streak** | L'utente sente il peso delle serie consecutive — esattamente la pressione che il prodotto vuole evitare |
| **Indistinguibilità** | Un utente con 1 anno di uso con score 200 e uno con 6 mesi con score 200 sembrano identici, ma hanno traiettorie completamente diverse |
| **Distorsione del grafico** | La forma della curva dipende dalla lunghezza della storia, non dalla qualità recente del comportamento |

Il modello di traiettoria risolve tutti questi problemi rappresentando lo **stato corrente** del comportamento, non la sua storia accumulata.

---

## 3. Generazione del segnale giornaliero

Ogni giorno, il comportamento dell'utente produce un **segnale di input** (`input_t`) secondo questa regola:

```
Azione completata          →  input_t = +1.0
Prima mancanza             →  input_t =  0.0
Seconda mancanza consecutiva →  input_t = -0.5
Terza mancanza consecutiva o più →  input_t = -1.0
```

### Razionale dei valori

Il segnale non è simmetrico per design:

- **+1.0**: il comportamento è avvenuto — segnale pieno positivo
- **0.0**: prima mancanza — tolleranza neutra, nessuna penalità immediata (la vita reale include interruzioni)
- **-0.5**: seconda mancanza — il pattern inizia a emergere, segnale lieve negativo
- **-1.0**: terza+ mancanza — pattern consolidato, segnale negativo pieno

Questa gradazione evita la punizione istantanea per un singolo giorno saltato, ma registra in modo progressivo i pattern di abbandono.

> **Nota:** Questo segnale è calcolato internamente e viene usato solo come input nel modello. Non viene mai visualizzato come numero in UI (salvo override esplicito dell'utente nelle impostazioni).

### Mancanze non registrate

Se l'utente non apre l'app, il sistema non genera automaticamente un segnale negativo. Il segnale `-0.5` e `-1.0` si attivano solo quando il sistema può determinare l'assenza rispetto alla frequenza configurata per l'area.

---

## 4. Modello di stato latente — EWMA

Il sistema stima uno **stato comportamentale latente** `state_t` che rappresenta la direzione corrente del comportamento dell'utente. Questo stato non è osservato direttamente — è inferito dal segnale giornaliero.

### Formula di aggiornamento

```
state_t = state_(t-1) + α × (input_t − state_(t-1))
```

Equivalentemente:

```
state_t = (1 − α) × state_(t-1) + α × input_t
```

Dove:

| Simbolo | Significato |
|---|---|
| `state_t` | Stato della traiettoria al giorno t (valore visualizzato nel grafico) |
| `state_(t-1)` | Stato del giorno precedente |
| `input_t` | Segnale giornaliero generato dal comportamento (+1, 0, -0.5, -1) |
| `α` | Coefficiente di adattamento (vedi sezione 5) |

### Comportamento del modello

Il modello è una **media mobile esponenziale** (EWMA — Exponentially Weighted Moving Average). Ogni nuovo segnale aggiorna lo stato in modo proporzionale alla distanza dal valore corrente, pesando il passato con un fattore di decadimento.

Proprietà chiave:

- **Memoria graduata**: i giorni recenti pesano più di quelli lontani
- **Reversibilità**: dopo una serie di mancanze, un ritorno al comportamento fa risalire lo stato gradualmente — non istantaneamente, ma in modo credibile
- **Stabilità**: una singola giornata eccezionale non genera picchi artificiali
- **Assenza di drift esponenziale**: lo stato rimane sempre nello stesso range, indipendentemente dalla lunghezza della storia

### Inizializzazione

```
state_0 = 0.0
```

Lo stato inizia a zero e converge verso il comportamento reale dell'utente nelle prime settimane.

---

## 5. Coefficiente di adattamento α

### Valore MVP

```
α = 0.08
```

### Interpretazione

Il coefficiente `α` determina quanto velocemente il modello risponde ai cambiamenti comportamentali.

Con `α = 0.08`:

- La **memoria effettiva** è di circa 25–30 giorni
- Un singolo giorno ha peso ≈ 8% nella determinazione dello stato corrente
- Dopo 30 giorni consecutivi di comportamento positivo (+1), lo stato supera +0.85
- Dopo 30 giorni consecutivi di assenza (-1), lo stato scende sotto -0.85
- Il modello non raggiunge mai esattamente ±1 — converge asintoticamente

### Formula della memoria effettiva

```
giorni_effettivi ≈ 1 / α = 1 / 0.08 ≈ 12.5 giorni (media esponenziale)
```

In pratica, l'EWMA con `α = 0.08` ha un half-life di circa 8–9 giorni: il segnale di un giorno specifico perde metà del suo peso in 8-9 giorni.

### Perché 0.08 è il valore giusto per il MVP

| α | Comportamento | Problema |
|---|---|---|
| `0.30` | Risposta rapida, alta reattività | Grafico rumoroso, falsi segnali |
| `0.15` | Risposta media | Accettabile, ma troppo volatile per traiettoria |
| `0.08` | Risposta graduale, memoria ~30 giorni | **Bilanciamento ottimale** |
| `0.03` | Risposta molto lenta | Cambiamenti reali non si vedono per mesi |

### Evoluzione futura

Per il MVP viene usato un **singolo coefficiente globale**. Versioni future potranno introdurre coefficienti differenziati per categoria:

- Abitudini fisiologiche (movimento, alimentazione): risposta più rapida, α ≈ 0.12
- Abitudini cognitive (studio, lettura): risposta media, α ≈ 0.08
- Abitudini da ridurre: risposta simmetrica al peggioramento, α ≈ 0.10

Questa differenziazione è architetturalmente semplice da introdurre: il coefficiente viene letto dalla configurazione dell'area, non hardcodato nel motore.

---

## 6. Interpretazione del grafico

### Cosa visualizza il grafico

Il grafico visualizza `state_t` nel tempo — la traiettoria dello stato comportamentale latente.

**Non rappresenta:**
- un punteggio totale accumulato
- una somma di azioni completate
- un ranking o un confronto con obiettivi

**Rappresenta:**
- la direzione corrente del comportamento
- quanto il comportamento recente è allineato con la frequenza configurata

### Lettura della curva

| Forma della curva | Significato |
|---|---|
| In salita | Il comportamento recente è prevalentemente positivo |
| Piatta | Il comportamento è stabile (mix equilibrato di completamenti e mancanze) |
| In discesa | Il comportamento recente è prevalentemente assente |

### Colore della linea (slope-based)

Il colore della linea non è fisso — si adatta alla direzione degli ultimi 7 giorni:

```typescript
function getLineColor(slope7d: number): string {
  if (slope7d > 0.1)  return '#7DA3A0'; // positivo — verde acqua
  if (slope7d < -0.1) return '#BFA37A'; // in declino — oro caldo (MAI rosso)
  return '#8C9496';                     // neutro — grigio
}
```

Il rosso non viene mai usato per segnalare declino. Il tono non è mai punitivo.

### Granularità adattiva del grafico

Il grafico adatta la risoluzione temporale al range selezionato, seguendo il pattern delle app di trading (es. dati giornalieri per 1 settimana, settimanali per 6+ mesi).

La granularità è determinata dallo **span effettivo dei dati**, non dal nome del range selezionato:

| Span dei dati | Granularità | Punto nel grafico |
|---|---|---|
| ≤ 90 giorni | Giornaliera | `trajectory_state` del singolo giorno |
| 91 giorni – 2 anni | Settimanale | `trajectory_state` dell'ultimo giorno della settimana |
| > 2 anni | Mensile | `trajectory_state` dell'ultimo giorno del mese |

L'obiettivo è mantenere **30–80 punti visibili** nel grafico, indipendentemente dalla lunghezza della storia dell'utente. Questo gestisce correttamente anche il range "All" che cresce nel tempo.

**Perché usare l'ultimo giorno del periodo e non la media?**

L'EWMA è già una media pesata esponenziale: il valore dell'ultimo giorno del periodo (settimana o mese) incorpora l'intera storia con il peso corretto. Usarlo come punto rappresentativo è matematicamente coerente — non è necessario ricalcolare una media di valori già mediati.

**Calcolo della slope adattivo:**

La finestra della slope si adatta alla granularità per restare proporzionata:
- Giornaliera → slope sugli ultimi 7 punti (7 giorni)
- Settimanale → slope sulle ultime 4 settimane
- Mensile → slope sugli ultimi 3 mesi

### Range dei valori visualizzati

Con `α = 0.08` e segnali compresi in `[-1.0, +1.0]`, lo stato tende a rimanere in:

```
state_t ∈ (-1.0, +1.0)
```

I valori estremi si raggiungono solo dopo settimane di comportamento costante nello stesso segno. Il grafico si adatta automaticamente all'asse Y — non ha target fissi.

---

## 7. Note di implementazione per sviluppatori

### Schema database (Supabase)

La tabella `score_daily` mantiene sia il segnale giornaliero che lo stato EWMA:

```sql
score_daily (
  id              UUID PRIMARY KEY,
  area_id         UUID REFERENCES areas(id),
  user_id         UUID REFERENCES users(id),
  date            DATE,
  daily_score     FLOAT,   -- il segnale input_t (+1, 0, -0.5, -1)
  trajectory_state FLOAT,  -- il nuovo state_t calcolato con EWMA
  consecutive_missed INT,  -- contatore mancanze consecutive
  UNIQUE(area_id, date)
)
```

> **Nota di migrazione:** la colonna `cumulative_score` esistente rappresenta l'accumulazione semplice (vecchio modello). La nuova colonna `trajectory_state` contiene il valore EWMA. Il grafico deve leggere `trajectory_state`, non `cumulative_score`. Entrambe le colonne possono coesistere durante la transizione.

### Calcolo al momento del check-in

```typescript
// Pseudocodice — logica da implementare in Edge Function o backend
function computeTrajectoryState(
  previousState: number,
  inputSignal: number,
  alpha: number = 0.08
): number {
  return previousState + alpha * (inputSignal - previousState);
}

function computeInputSignal(consecutiveMissed: number, completed: boolean): number {
  if (completed) return 1.0;
  if (consecutiveMissed === 0) return 0.0;   // prima mancanza
  if (consecutiveMissed === 1) return -0.5;  // seconda consecutiva
  return -1.0;                               // terza+ consecutiva
}
```

### Ordine di calcolo

1. Leggere `state_(t-1)` dall'ultimo record `score_daily` per l'area
2. Determinare `input_t` dalla regola segnale (completato / mancanza consecutiva)
3. Calcolare `state_t = state_(t-1) + α × (input_t − state_(t-1))`
4. Salvare `{ date, daily_score: input_t, trajectory_state: state_t, consecutive_missed }`

### Gestione del primo giorno

Se non esiste un record precedente per l'area:

```
state_(t-1) = 0.0  (inizializzazione)
```

### Retrocompatibilità

Il calcolo EWMA può essere eseguito retroattivamente su tutta la storia esistente. Se un'area ha già `score_daily` con `daily_score` popolato, è sufficiente ricalcolare `trajectory_state` in ordine cronologico applicando la formula ricorsiva a partire dal primo record.

### Dove usare trajectory_state

| Schermata | Dato da usare |
|---|---|
| Area Detail — grafico traiettoria | `trajectory_state` |
| Progress — grafico aggregato | media di `trajectory_state` di tutte le aree attive |
| Dashboard — mini grafico card | `trajectory_state` |
| Settings — score visibile | `trajectory_state` (opzionale, se l'utente abilita) |

---

## 8. Caso speciale: area Finance

L'area Finance introduce due dimensioni concettuali distinte che richiedono trattamento separato.

### Dimensione 1 — Abitudine di risparmio (comportamentale)

Il check-in finanziario quotidiano (hai fatto una azione di risparmio oggi?) segue **esattamente lo stesso modello EWMA** descritto in questo documento.

```
state_t(finance_habit) = state_(t-1) + α × (input_t − state_(t-1))
```

Il grafico dell'abitudine di risparmio è identico agli altri grafici comportamentali.

### Dimensione 2 — Crescita del capitale (economica)

La proiezione finanziaria non è un segnale comportamentale — è una proiezione di una grandezza economica (il saldo o il risparmio accumulato).

Questa dimensione segue un **modello di proiezione economica separato**, non EWMA:

**MVP: regressione lineare semplice**

```
// Ultimi 30 check-in → retta y = mx + b → proiezione 30 giorni futuri
```

**Futuri miglioramenti possibili** (sostituibili in modo modulare):
- Interesse composto su saldo
- Proiezione con tasso di risparmio variabile
- Monte Carlo su scenari

### Separazione architetturale

Il layer di calcolo per la proiezione finanziaria è **modulare e sostituibile** indipendentemente dal motore EWMA:

```
┌─────────────────────────────────┐
│  EWMA Engine (globale)          │  ← calcola trajectory_state per TUTTE le aree
└────────────────┬────────────────┘
                 │
     ┌───────────┴───────────┐
     │                       │
┌────▼────┐          ┌───────▼──────────┐
│ Aree    │          │ Finance          │
│ standard│          │                  │
│ (habit, │          │  habit → EWMA ✓  │
│ health, │          │  capital → modulo│
│ study)  │          │  proiezione sep. │
└─────────┘          └──────────────────┘
```

Il modulo di proiezione finanziaria può essere aggiornato senza toccare il motore EWMA.

---

## 9. Valori di riferimento e boundary conditions

### Simulazione del modello con α = 0.08

| Scenario | state dopo 7gg | state dopo 30gg | state dopo 90gg |
|---|---|---|---|
| Comportamento perfetto (tutti +1) | ≈ +0.43 | ≈ +0.92 | ≈ +0.99 |
| Comportamento assente (tutti -1) | ≈ -0.43 | ≈ -0.92 | ≈ -0.99 |
| Comportamento misto (alterna +1/-1) | ≈ 0.0 | ≈ 0.0 | ≈ 0.0 |
| Recovery: 30gg assenza → 30gg presente | → -0.92 → ≈ -0.50 dopo altri 30gg |

### Convergenza

- Il modello non esplode: `state_t` è sempre limitato in `(-1, +1)` con segnali in `[-1, +1]`
- Il modello è stazionario: indipendente dalla lunghezza della storia
- Il modello è deterministico: dati gli stessi input, produce sempre lo stesso output

### Differenza rispetto al cumulative_score

| Caratteristica | cumulative_score | trajectory_state (EWMA) |
|---|---|---|
| Range | Illimitato (cresce nel tempo) | Sempre in (-1, +1) |
| Memoria | Infinita (tutto pesa uguale) | Esponenzialmente decrescente |
| Sensibilità al passato remoto | Alta | Bassa (quasi nulla oltre 60 giorni) |
| Reversibilità | Difficile (le mancanze restano) | Naturale (il segnale si attenua) |
| Confrontabilità tra utenti | Impossibile (dipende dalla durata) | Possibile (stesso range per tutti) |
| Crescita nel tempo | Sì (artefatto matematico) | No (stazionario) |

---

*Documento mantenuto in `architecture/behavioral-trajectory-model.md`*
*Aggiornato: marzo 2026*
