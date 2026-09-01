# Piano di lavoro — Piano Fueling + UI Refresh

> Documento operativo **per l'agente di sviluppo** (Claude Code). Fonte di verità: `SPECpianofueling.md`.
> Regola d'oro: si implementa **solo quando l'utente lo chiede esplicitamente**. Fino ad allora questo file è la mappa; si spuntano le caselle man mano.

- **Branch**: `feature/fueling-ui-refresh` (mai su `main`; merge su `main` solo a criteri di successo verificati).
- **File unico**: `index.html` (HTML + CSS + JS vanilla, nessuna build).
- **Deploy**: push → GitHub Pages (workflow esistente, nessuna modifica).

---

## 0. Registro decisioni (validate con l'utente — non ri-discutere)

| # | Tema | Decisione |
|---|---|---|
| D1 | **Export PDF** | Si usa una **libreria via CDN** (es. `html2pdf.js` o `jsPDF`). Deroga consapevole al principio "zero dipendenze runtime": serve rete al primo load. Fallback `window.print()` non richiesto. |
| D2 | **Durata non multipla della cadenza** | La **durata di frazione usata dal pianificatore viene arrotondata ai 10 minuti più vicini** (round-half-up, soglia a 5 min). Esempi dell'utente: `5h03 → 5h00`, `4h07 → 4h10` (3→0, 7→10: regola dei 10 min, non dei 5 — corretto in Step 4). Su questa durata arrotondata si generano gli slot e si calcolano i totali orari; niente coda residua da gestire a parte. |
| D3 | **Ambito sessione corrente** | Questa sessione: (a) crea il branch, (b) redige questo `piano.md`. L'implementazione parte **solo su richiesta esplicita dell'utente**, step per step. |
| D4 | **Estetica** | Rivisitazione **leggera**, identità invariata: dark sport-editorial, Bebas Neue + JetBrains Mono, palette esistente (`--bg`, `--accent #d4ff4a`, `--bike #f4a261`, `--run #e63946`, ecc.). Niente redesign. |

**Nota (D2) — risolta**: arrotondamento ai 10 min con `Math.round(min/10)*10` → tie a metà (5 min) va per eccesso (half-up). Es.: 75→80, 25→30, 74→70. Confermato coi due esempi dell'utente.

---

## 1. Vincoli invarianti (checklist di guardia — valgono per OGNI step)

- [ ] Single file `index.html`; nessuna build; nessun backend/account/tracking/chiamata di rete (eccetto CDN libreria PDF e i font già presenti).
- [ ] **Non toccare** markup né logica della sezione calcolo tempi esistente, se non per **leggere** le durate bici/corsa. Qualunque modifica necessaria a quella sezione → **chiedere prima**.
- [ ] Persistenza **solo** tramite `StorageAdapter` (`get/set/list/delete`), dati JSON serializzabili, `schemaVersion` nel modello.
- [ ] I tipi di vincolo di posizionamento restano **esattamente tre** (momento preciso / solo prima metà / solo seconda metà). Non estendere.
- [ ] Nuoto escluso; T1/T2 ignorate.
- [ ] Validare tutti gli input numerici.
- [ ] Estetica coerente (D4).

---

## 2. Modello dati (riferimento per gli step)

```
schemaVersion: <int>
Prodotto {
  id, nome,
  formato: 'gel' | 'barretta' | 'borraccia' | 'capsula',
  carbo_g, sodio_mg, caffeina_mg, volume_ml, quantita,
  componenti?: [ {nome, carbo_g, sodio_mg, caffeina_mg, volume_ml} ]   // SOLO borraccia
}
InventarioBici: Prodotto[]      // liste separate
InventarioCorsa: Prodotto[]
VincoloProdotto (per frazione, opzionale) {
  tipo: 'momento' | 'prima_meta' | 'seconda_meta',
  minuto?: <int>                // solo se tipo='momento'
}
FrazioneConfig { durataMin (editabile, arrotondata D2), cadenzaMin, targetCarbo_gh, targetSodio_mgh, targetLiquidi_mlh, offsetPrimoSlotMin? }
RigheManuali { colazione, preGara, minuto0 }  // fuori allocatore, fuori rate orari
Piano { ...snapshot risultato per diff su ricalcolo }
```

- **Frazionabilità**: gel = intero; barretta = intera o metà; capsula = intera (o sciolta in borraccia come componente); borraccia = a sorsi (l'app calcola la suddivisione in sorsi sugli slot).
- **Borraccia**: mini-form "aggiungi componente"; i totali borraccia = somma componenti.

---

## 3. Step di implementazione (ordinati, ognuno con accettazione + verifica)

> Ogni step è piccolo e chiudibile da solo. Committare a fine step. Aprire il file nel browser per verificare (nessuna build).

### Step 1 — StorageAdapter + versioning ✅
- [x] Implementare `StorageAdapter` con `get/set/list/delete` su `localStorage`, JSON serializzabile, chiavi namespaced (`fueling:*`).
- [x] Aggiungere `schemaVersion` e uno stub di migrazione (`migrateFueling(data)` no-op iniziale) + `initFuelingStore()` che crea/aggiorna `fueling:__meta`.
- **Accettazione**: salvando e ricaricando la pagina i dati persistono; l'interfaccia non conosce `localStorage` direttamente (solo via adapter). ✅ + fallback in-memory se localStorage non disponibile.
- **Verifica**: round-trip `get/set/list/delete` testato con Node (stub); `fueling:__meta` = `{"schemaVersion":1}`. In browser: DevTools → Application → localStorage mostra `fueling:__meta`; reload mantiene lo stato.
- **Note**: adapter esposto su `window.Fueling` per verifica manuale (nessuna UI in questo step, come da piano).

### Step 2 — Sezione Fueling: scaffold + lettura durate ✅
- [x] Nuova sezione "Piano Fueling" sotto il calcolatore esistente (esistente **non alterato**: durate lette ricalcolando dagli stessi input, listener additivi — nessuna modifica a `calc()`).
- [x] Campi durata bici/corsa (in minuti) **pre-popolati** dal calcolatore, **editabili**; override manuale con bottone «sync da calcolo» per ripristinare; stato salvato via `StorageAdapter` (`fueling:durations`).
- **Accettazione**: cambiando i tempi nel calcolatore le durate fueling si aggiornano finché l'utente non le sovrascrive. ✅
- **Verifica**: simulazione Node (DOM stub) — init bici 38/corsa 25; bici 40 km → 75; override corsa 60 tiene al cambio distanza; «sync» ripristina 50; persistenza OK. Screenshot: sezione integrata nell'estetica esistente. (spec §1)

### Step 3 — CRUD prodotti + inventari separati ✅
- [x] Form prodotto (campi modello §2) con create/edit/delete; lista prodotti; inventari **bici** (`fueling:inv_bike`) e **corsa** (`fueling:inv_run`) distinti, toggle in UI.
- [x] Formato "borraccia": mini-form componenti dinamici; totali borraccia = somma componenti (denormalizzati nel prodotto salvato). Gel/barretta/capsula: campi diretti. Validazione input (nome obbligatorio, numerici ≥0, ≥1 componente per borraccia).
- **Accettazione**: borraccia con 3 componenti → totali = somma. ✅ (test: carbo 90, sodio 600, caff 75, volume 500)
- **Verifica**: test jsdom 17/17 — add di ogni formato, borraccia=somma, inventari separati, validazione nome vuoto, delete, persistenza dopo reload. Screenshot: form integrato nell'estetica.

### Step 4 — Config frazione: cadenza, target, offset corsa, arrotondamento D2 ✅
- [x] Cadenza indipendente bici/corsa; target g/h, mg/h, ml/h indipendenti per frazione; nessun target caffeina (nota in UI).
- [x] Bici: 1° slot al primo tick di cadenza. Corsa: 1° slot al **minuto personalizzato** (offset), poi cadenza. Nessun minuto 0 corsa.
- [x] Arrotondamento durata D2 = **ai 10 min più vicini** (half-up), applicato a slot e totali. Config persistita (`fueling:config`). Preview live per frazione (durata arrotondata, slot, target).
- **Accettazione**: slot bici dal primo tick; slot corsa dall'offset; arrotondamento corretto (5h03→5h00, 4h07→4h10). ✅
- **Verifica**: test jsdom 16/16 — roundDuration(303)=300, (247)=250, (75)=80, (74)=70; bici 62→60 slot 20,40,60; corsa 52→50 offset10/cad15 slot 10,25,40; target scalano con la durata (90g@60min, 135g@90min); cadenze indipendenti; persistenza offset dopo reload. Screenshot: card a due colonne. (spec §2)
- **API esposte** (per Step 6/7): `Fueling.roundDuration`, `Fueling.buildSlots(kind)`, `Fueling.fractionTargets(kind)`, `Fueling.getConfig()`.

### Step 5 — Righe manuali (colazione, pre-gara, minuto 0) ✅
- [x] Tre righe (orari default -2:30 / -0:15 / 0) con prodotto e orario a scelta + carbo/sodio/caff/volume; **fuori** dall'allocatore e **fuori** dai rate orari. Riga "attiva" se ha prodotto o un valore >0. Persistite (`fueling:manual`), con totali live.
- **Accettazione**: compaiono nei totali (e in tabella/lista prodotti agli step 9) ma non spostano i g/h calcolati. ✅
- **Verifica**: test jsdom 13/13 — `fractionTargets` e `buildSlots` **identici** prima/dopo l'inserimento di righe manuali (byte-per-byte); filtro righe attive (0→2→3); totali (carbo 105, sodio 200, caff 100); persistenza dopo reload. Screenshot: card tre righe integrata. (spec §3)
- **API esposte**: `Fueling.manualRows()`, `Fueling.getManual()`, `Fueling.manualTotals()`.

### Step 6 — Vincoli di posizionamento (3 tipi) ✅
- [x] Sul prodotto (l'inventario è già per-frazione): **momento preciso** (aggancio metà / fine / minuto X), **solo prima metà**, **solo seconda metà**. Esattamente 3 tipi. UI nel form con sub-campi condizionali; badge in lista; persistito col prodotto.
- [x] Resolver `Fueling.resolveConstraint(vincolo, kind)` → per momento: slot più vicino al minuto target (tie → slot precedente); per prima/seconda metà: sottoinsieme di slot ammessi (`≤half` / `>half`); nessuno → tutti.
- **Accettazione**: vincolo "metà bici" → slot più vicino alla metà (dur 80/cad 20 → slot 40 esatto). ✅
- **Verifica**: test jsdom 15/15 — meta→40 (esatto) e →20 (tie, precedente), fine→60, min33→40, prima_meta→[20], seconda_meta→[40,60], nessuno→tutti; vincolo salvato/ripristinato sul prodotto (incl. edit dopo reload); badge in lista; visibilità sub-campi.
- **Bugfix**: aggiunto `[hidden]{display:none!important}` — una regola d'autore (`.row{display:grid}`) batteva lo stile UA di `[hidden]`, lasciando visibile il campo "Minuto"; jsdom non lo rilevava (non calcola CSS), lo screenshot sì.
- **API esposte**: `Fueling.resolveConstraint(vincolo, kind)`.

### Step 7 — Allocatore greedy (Opzione A) ✅
- [x] Ordine implementato: (1) momenti precisi (tutte le unità nello slot più vicino) → (2) borracce a sorsi distribuite uniformemente sugli slot ammessi (vincoli prima/seconda metà) → (3) gap carbo con gel + mezze barrette → (4) gap sodio con capsule.
- [x] Rispetta quantità (residuo per prodotto, mezza barretta = 0.5) e vincoli; **mai inventa** (usage ≤ disponibile).
- [x] Totale di frazione come àncora; per slot sceglie lo slot col deficit maggiore (minimizza scarto); delta per slot calcolato; greedy si ferma entro ±1 unità minima (piazza l'overshoot solo se avvicina: `val < 2·gap`).
- [x] Deficit riportato e **evidenziato a colore** (rosso) con "Nessun prodotto inventato"; pannello "Piano calcolato" + bottone; snapshot `fueling:lastplan`.
- **Accettazione**: inventario sufficiente → carbo/sodio/liquidi = target esatti; insufficiente → deficit colorato, nessuna invenzione. ✅
- **Verifica**: test jsdom 20/20 — sufficiente (borraccia 30/300/500 + gel×3 + caps×2, target 90/500/500): carbo=90, sodio=500, liquidi=500 esatti, usa 2/3 gel, 1/2 caps, 1 borraccia, liquido = 500/3 per slot; insufficiente (1 gel): deficit carbo=30, usage≤disponibile, ok=false; momento "fine" → tutte le unità all'ultimo slot; prima_meta → solo slot ammessi ([20]); inventario vuoto → deficit pieno, nessun uso, nessun crash; mezza barretta. Rendering verificato (deficit colorato, lastplan persistito) + screenshot. (spec §5, §6)
- **API esposte**: `Fueling.allocateFraction(kind)`, `Fueling.allocatePlan()`.

### Step 8 — Ricalcolo durata (diff) ✅
- [x] Rate orari e mix invariati (sono config): cambiando la durata l'allocatore (Step 7) riscala i target e aggiunge/toglie unità. Slot ricalcolati da `buildSlots` (tabella si allunga/accorcia).
- [x] Snapshot "piano base" (`fueling:baseline`) + confronto con il piano corrente: lista prodotti usati con badge **aggiunto/aumentato** (verde) e **rimosso/diminuito** (rosso); primo calcolo = base automatica; bottone «fissa piano base» per aggiornarla. Deficit invariato dallo Step 7.
- **Accettazione**: +durata estende la tabella e colora le unità aggiunte/rimosse. ✅
- **Verifica**: test jsdom 11/11 — base 3 gel @60′; +60′→6 gel (+3, increased); −a 40′→2 gel (−1, decreased); prodotto reso indisponibile→removed vs base; prodotto nuovo usato→added; rendering con classi colore; persistenza baseline dopo reload. Screenshot: Gel ×6 (+3) e Salt caps (+2) in verde.
- **API esposte**: `Fueling.planDiff()`, `Fueling.setPlanBaseline()`, hook `Fueling.afterPlan`.

### Step 9 — Output: tabelle e riepilogo ✅
- [x] Card "Report piano": **tabella per slot** per frazione (righe = tick cadenza; le 3 righe manuali in testa alla tabella bici, taggate "manuale", con Δ = —), prodotti per slot e Δ carbo/liquidi.
- [x] **Riepilogo orario** per frazione: bucket da 60′ (il tick a fine ora appartiene all'ora che chiude), carbo/sodio/liquidi actual/target con sotto-target in rosso, caffeina come totale (nessun target).
- [x] **Lista prodotti finale**: uso allocatore per frazione (con diff colorato vs piano base) + righe manuali + **totale complessivo incl. manuali**.
- **Accettazione**: le tre viste rendono coerentemente; caffeina come totale mg. ✅
- **Verifica**: test jsdom 14/14 — 3 sezioni presenti; righe manuali in tabella; 2 bucket orari a 120′ con target /90; totale complessivo carbo 285 (180 piano + 105 manuali); coerenza totale di frazione 180; placeholder pre-calcolo; hook Step 8 ancora attivo (catena non sostituita). Regressione completa step 3–9: 106/106. Screenshot report.
- **Miglioria allocatore (Step 7)**: tie-break sugli slot ora sceglie il più lontano dagli slot già potenziati (`boosted`) → distribuzione più uniforme nel tempo (spec §"distribuzione uniforme"), meno clustering iniziale. La parità perfetta per-ora non è garantita con unità discrete (non richiesta): conta il totale di frazione + delta per slot. test7 invariato 20/20.
- **API esposte**: `Fueling.renderReport(plan)`.

### Step 10 — Export PDF (D1) ✅
- [x] Libreria confermata: **jsPDF 2.5.2 + AutoTable 3.8.2** via cdnjs (UMD, versioni pinnate). PDF chiaro da stampa, testo vettoriale/selezionabile.
- [x] `buildExportModel()` (dati puri) + `exportPDF()` (rendering): per frazione tabella per slot (incl. righe manuali) + riepilogo orario actual/target + deficit; poi lista prodotti finale (con diff) e totale complessivo incl. manuali. Bottone «Esporta PDF» con guardia "libreria non caricata".
- **Accettazione**: export produce un PDF leggibile con le tre parti. ✅
- **Verifica**: test jsdom+Node 15/15 — modello corretto (durate, slot incl. 2 manuali, 6 tick, hourly, totale 285, caff 100); **PDF reale generato con jsPDF+AutoTable** (datauri application/pdf, 2 pagine, AutoTable finalY>0, guardia no-lib). Estrazione testo (pdf-parse) verificata. URL cdnjs confermati coi nomi file canonici (`jspdf.umd.min.js`, `jspdf.plugin.autotable.min.js`). Regressione step 3–10: 121/121.
- **Fix**: caratteri fuori WinAnsi nei font standard jsPDF (Δ → "var"/"Diff"; U+2212 → "-" via `pdfSafe`), così il PDF non mostra glifi errati; rimosso un byte NUL introdotto per sbaglio in una regex.
- **API esposte**: `Fueling.buildExportModel()`, `Fueling.exportPDF()`.
- **Nota offline**: la libreria è CDN → serve rete al primo caricamento (deroga D1 accettata).

### Step 11 — UI refresh leggero ✅
- [x] Audit con `baseline-ui` + `fixing-accessibility` (skill caricate). Fix mirati, nessun redesign, identità invariata (D4).
- [x] **Focus visibile da tastiera**: gli input azzeravano l'outline su `:focus` → aggiunti anelli `:focus-visible` con accent su input/select/bottoni/link (mouse invariato).
- [x] **Annunci screen reader**: `aria-live="polite"` su `#planResult` e `#reportBody`, `role="status"` su `#pdfNote`; preview live-typing lasciate mute per non spammare.
- [x] **Form**: `aria-describedby="prodError"` sul nome prodotto; empty-state inventario con azione successiva.
- [x] **Motion**: `@media (prefers-reduced-motion: reduce)` azzera transizioni/animazioni.
- [x] **Tipografia/dati**: `text-wrap: balance` su titoli, `pretty` su paragrafi, `tabular-nums` sulle tabelle dati; leggero bump contrasto `.stat-label`.
- **Accettazione**: facciata più pulita, nessuna regressione d'identità, accessibilità migliorata. ✅
- **Verifica**: regressione completa step 3–10 = 121/121 (CSS/markup non toccano la logica); screenshot: header/calcolatore identici (Bebas Neue + JetBrains Mono, accent lime, colori disciplina). Nessuna modifica alla logica calcolo tempi.

### Step 12 — Verifica finale criteri di successo
- [ ] Ripercorrere tutti i checkbox "Criteri di successo (testabili)" della spec e spuntarli.
- [ ] Test persistenza: reload mantiene prodotti, inventari, ultimo piano.
- **Accettazione**: tutti i criteri spec verdi.
- **Verifica**: solo allora proporre merge su `main`.

---

## 4. Mappa criteri di successo spec → step

| Criterio spec | Step |
|---|---|
| Durate popolate ed editabili | 2 |
| Cadenze/target indipendenti; primo slot bici/corsa | 4 |
| Righe manuali fuori dai rate | 5 |
| Borraccia = somma componenti | 3 |
| Totale frazione = target ±1; delta per slot | 7 |
| Deficit a colore, nessun prodotto inventato | 7 |
| Vincolo "metà bici" → slot più vicino | 6 |
| +30 min estende tabella + diff colorato | 8 |
| PDF leggibile con 3 parti | 10 |
| Reload mantiene stato | 1, 12 |
