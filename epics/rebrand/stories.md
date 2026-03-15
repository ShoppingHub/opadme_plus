# Stories — Rebrand: opad.me → opad.me

## Sequenza di implementazione

```
story-rebrand-01 → Rename app + aggiorna wordmark con nuovi colori logo   ⏳ da fare
```

---

## story-rebrand-01 — Rename app e wordmark opad.me

L'app cambia nome da **opad.me** a **opad.me**. Aggiorna tutto il codebase con il nuovo nome e il nuovo stile del wordmark.

---

### 1. Rename globale

Sostituisci tutte le occorrenze di `"opad.me"` con `"opad.me"` in:
- `<title>` del documento HTML
- Meta tags (`og:title`, `og:site_name`, ecc.)
- Header della Dashboard e di ogni pagina che mostra il nome app
- Qualsiasi stringa hardcoded nel codice

---

### 2. Wordmark nella sidebar desktop e nell'header mobile

Il wordmark `"opad.me"` è composto da due parti cromatiche distinte:

```
opad  .me
────  ───
  ↑    ↑
  │    └── colore #B5453A (terracotta)
  └─────── colore #FFFFFF (bianco)
```

**Implementazione:**
- Renderizza il wordmark come due `<span>` inline:
  - `<span style="color: #FFFFFF">opad</span><span style="color: #B5453A">.me</span>`
- Font: Inter, font-weight 600
- Non usare `#B5453A` in nessun altro elemento dell'UI — è esclusivo del wordmark

**Posizioni dove appare il wordmark:**
- Sidebar desktop: in cima, dove ora c'è `"opad.me"` (testo)
- Header mobile della Home: a sinistra, dove ora c'è `"opad.me"` (testo)

---

### 3. Icona / logo (da fare in futuro)

Il logo finale prevede un'icona SVG a sinistra del wordmark (figura Uomo Vitruviano in cerchio con ingranaggio alla base) che sostituisce visivamente la "o" di "opad". L'asset SVG non è ancora disponibile — **per ora usare solo il wordmark testuale `"opad.me"`** con i due colori sopra. Quando l'SVG sarà pronto verrà gestito in una story separata.

---

### 4. Favicon e app icon

Aggiorna il `favicon.ico` e l'`apple-touch-icon` con il nuovo nome/identità se gli asset sono disponibili. Se non lo sono, lascia invariato per ora — non bloccare questa story per la mancanza di asset grafici.

---

### Note

- Il colore `#B5453A` non è nel sistema dei colori dell'app (non si usa su pulsanti, grafici, stati UI) — è **solo per il wordmark**
- Il tono visivo dell'app rimane invariato — stessi background, stessi grafici, stessa tipografia
