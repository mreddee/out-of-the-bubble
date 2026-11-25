# CLAUDE.md — Out of the Bubble

> Contesto di progetto per Claude Code. Leggilo prima di ogni sessione.

## Come lavorare con me

- **Lingua:** parla con me in **italiano**. Codice, commenti, docstring, commit, README e testi dell'interfaccia vanno in **inglese** (il progetto è pensato per un pubblico internazionale e per il mio portfolio).
- **Obiettivo doppio:** costruire un prodotto davvero utile **e** imparare. Sto sviluppando competenze su Databricks, PySpark, dbt, BigQuery e Python "di produzione". Quando introduci un concetto nuovo (Delta Lake, ALS, medallion, test dbt…), spiegalo in 2–3 righe e motiva la scelta.
- **Passi piccoli e verificabili:** una fase alla volta. Proponi il piano prima di modifiche ampie, poi implementa, testa e riassumi cosa è cambiato.
- **Solo strumenti gratuiti:** non introdurre servizi a pagamento né dipendenze che richiedano carta di credito senza chiedermelo.
- **Niente segreti nel repo:** chiavi API e token solo in `.env` (in `.gitignore`) o nei GitHub Secrets. Fornisci sempre un `.env.example`.
- Se qualcosa non è verificato (limiti di un servizio, formato di un dump, endpoint di un'API), **controlla la documentazione ufficiale** o dimmelo, invece di indovinare.

---

## Il prodotto

**Out of the Bubble** aiuta chi ascolta molta musica a scoprire artisti poco conosciuti, uscendo dalla propria bolla in modo controllato e spiegato.

### Il problema
Gli algoritmi di raccomandazione tendono a favorire gli artisti popolari (bias di popolarità) e non spiegano perché suggeriscono qualcosa. Chi vuole davvero esplorare non ha strumenti con manopole e trasparenza.

### Utente target
Ascoltatori appassionati che sentono di ascoltare "sempre le stesse cose" e vogliono scoprire artisti di nicchia legati ai propri gusti. Indipendenti da qualunque servizio di streaming.

### Cosa lo rende unico (la combinazione, non i singoli pezzi)
1. **Aperto a chiunque in 30 secondi:** nessun account, poche domande precise.
2. **Manopola della notorietà:** l'utente sceglie quanto sconosciuti devono essere gli artisti.
3. **Il ponte spiegato:** ogni suggerimento mostra il percorso dai suoi artisti a quello nuovo.
4. **Misura se la scoperta dura:** per chi collega la cronologia, verifichiamo dopo 4 settimane se l'artista è entrato davvero nei suoi ascolti.

### Concorrenti noti (per non reinventare la ruota)
ListenBrainz Weekly Exploration / LB Radio (motore Troi), Discover Weekly, Gnoosic, Music-Map, Every Noise at Once (fermo dal 2023), Localify, Obscurify. Nessuno combina i quattro punti sopra.

---

## Flusso utente (MVP)

**Schermata 1 — Domande**
1. *Scegli 3–5 artisti che ami* — autocompletamento su un indice locale di artisti (MBID MusicBrainz). Obbligatoria.
2. *Quanto vuoi allontanarti?* — `near` (stesso mondo sonoro) / `sideways` (generi confinanti) / `far` (sorprendimi, con un filo logico). Obbligatoria.
3. *Quanto devono essere sconosciuti?* — `less_known` / `niche` / `hidden_gems`. Obbligatoria.
4. *Facoltativa:* username **Last.fm** o **ListenBrainz** → abilita l'esclusione degli artisti già noti e il riepilogo dopo 4 settimane.

**Schermata 2 — Risultati:** 8 artisti. Per ciascuno: genere e numero indicativo di ascoltatori, **ponte** (seed → artista intermedio → suggerito) con spiegazione di una riga, link di ascolto (URL di ricerca Spotify / YouTube / Bandcamp, **nessuna API**), feedback (👍 *Like* / 👎 *Not for me* / 🙃 *Already knew it*), pulsante *More like this*.

**Schermata 3 — Riepilogo dopo 4 settimane** (solo per chi ha dato lo username): imbuto suggeriti → provati → adottati → ancora ascoltati, con lo stato di ogni artista.

**Rimandati a una fase successiva (NON nell'MVP):** email mensile con nuove raccomandazioni, pagina "Gestisci preferenze", pagina pubblica dei KPI, caricamento dell'export GDPR di Spotify.

---

## Vincoli importanti

- **Niente API Spotify.** Dal 27/11/2024 le nuove app non possono più usare recommendations, audio features, audio analysis e related artists. Dal 2025–2026 la modalità sviluppatore è limitata a pochissimi utenti e l'accesso esteso è riservato alle aziende con un numero molto alto di utenti attivi. Per Spotify usiamo **solo link di ricerca**.
- **Databricks Free Edition:** solo compute serverless, accesso a internet in uscita limitato, un solo SQL warehouse 2X-Small, massimo 5 task di job in parallelo, **solo uso non commerciale**. Lavoriamo su 12–24 mesi di dati, non sull'intera storia.
- **Last.fm API:** gratuita per uso non commerciale (chiave API necessaria); per l'uso commerciale serve un accordo.
- **MusicBrainz API:** rate limit basso (circa 1 richiesta/secondo, con User-Agent identificativo). Per l'autocompletamento usiamo un **indice locale precalcolato**.
- **Privacy (GDPR):** salva il minimo indispensabile. Sessioni ed eventi legati a un id anonimo. Lo username (facoltativo) è l'unico dato personale nell'MVP.

---

## Architettura

Flusso: **ListenBrainz/MusicBrainz → BigQuery (esplorazione + aggregazione) → Databricks (bronze/silver/gold, modello, valutazione) → dbt (tabelle finali testate) → Parquet su Hugging Face → app Streamlit con DuckDB**. Aggiornamento ogni 2 settimane (Databricks Jobs + GitHub Actions).

| # | Livello | Strumento | Ruolo |
|---|---|---|---|
| 1 | Sorgenti | Dump ListenBrainz (formato Spark, ultimi 12–24 mesi), dati MusicBrainz | Ascolti (CC0) e anagrafica artisti (MBID, tag, relazioni). Chiave di join: `artist_mbid`. Si scartano gli ascolti non associati a un MBID. |
| 2 | Esplorazione | BigQuery (sandbox gratuita) + `google-cloud-bigquery` | EDA in SQL; aggrega in `user_id, artist_mbid, plays, first_listen, last_listen` ed esporta in Parquet. |
| 3 | Elaborazione | Databricks Free Edition: PySpark, Delta Lake, Unity Catalog Volumes | Medallion: **bronze** (grezzo), **silver** (pulizia: bot, artisti con pochi ascoltatori, join MusicBrainz), **gold** (`artist_popularity`, `user_artist_matrix`). |
| 4 | Modello | PySpark MLlib **ALS** (feedback implicito) + **MLflow** | Baseline di co-ascolto (PMI) vs ALS. Pesi `log(1 + plays)`. Vicini top 50–100 per artista (coseno). Ponti: artista intermedio B che massimizza sim(S,B)·sim(B,C). Backtest temporale. |
| 5 | Trasformazioni | **dbt Core** + `dbt-databricks` | Modelli SQL silver → gold, test (`unique`, `not_null`, `accepted_values`, test personalizzati), `dbt docs`. Il ML resta in Python. |
| 6 | Orchestrazione | **Databricks Jobs** + **GitHub Actions** | GHA: estrazione da BigQuery → upload nel Volume → avvio del job via API; CI con pytest. Job: bronze → silver → `dbt build` → modello → valutazione → export. |
| 7 | Serving | Parquet su un **dataset Hugging Face** + **DuckDB** | `artist_neighbors`, `artist_popularity`, `artist_index`. L'app non chiama mai Databricks. |
| 8 | App | **Streamlit Community Cloud** | UI del flusso; `recommend()` in Python puro e testato; feedback salvato in **Neon Postgres** (piano gratuito). |
| 9 | API live | API ListenBrainz (senza autenticazione), API Last.fm (chiave) | Esclude gli artisti già ascoltati; ciclo chiuso dopo 4 settimane (script programmato in GHA). |
| 10 | Qualità | GitHub, pytest, ruff, GitHub Actions | Repo pubblico, test, CI, README orientato al prodotto. |

### Logica di raccomandazione (`recommend()`)
1. Candidati = vicini degli artisti di partenza secondo la distanza scelta: `near` = vicini diretti; `sideways` = un passaggio intermedio; `far` = due passaggi, purché la catena resti solida (soglia minima di similarità).
2. Filtro per fascia di popolarità (soglie decise dall'EDA, **non** inventate).
3. Esclusione degli artisti di partenza e, se c'è lo username, di quelli già ascoltati.
4. Punteggio = aggregazione delle similarità verso i vari artisti di partenza.
5. Diversificazione (es. MMR) per evitare 8 artisti dello stesso micro-genere.
6. Spiegazione del ponte generata dai dati (artista di partenza, artista intermedio, similarità).
7. Funzione **deterministica** a parità di input (seed fisso), facile da testare.

---

## Metriche

**Offline (backtest):** recall@k e precision@k sugli artisti *nuovi per l'utente* e sotto la soglia di popolarità; novità; copertura del catalogo; popolarità media dei suggerimenti (bias di popolarità). Sempre ALS **contro** la baseline.

**Online (prodotto):**
- **Attivazione:** % di sessioni con domande completate e almeno un clic su un artista
- **Novità:** % di "Already knew it" (più bassa = meglio)
- **Gradimento:** rapporto 👍/👎, diviso per distanza e notorietà
- **Tasso di scoperta riuscita** (*North Star*): % di artisti suggeriti ancora ascoltati dopo 4 settimane (solo utenti con cronologia collegata)
- **Uplift causale:** per ogni utente con cronologia, alcuni candidati validi **non vengono mostrati** (gruppo di controllo); si confronta il loro tasso di adozione spontanea con quello degli artisti mostrati

Imbuto per artista: **provato** (≥1 ascolto) → **adottato** (ascoltato in ≥2–3 giorni diversi) → **scoperta riuscita** (ancora ascoltato dopo 4 settimane).

---

## Struttura del repository

```
out-of-the-bubble/
├── CLAUDE.md
├── README.md                 # problema → utente → demo → architettura → risultati
├── .env.example
├── pyproject.toml            # dipendenze, ruff, pytest
├── exploration/              # query BigQuery + notebook EDA
├── pipelines/                # notebook/script Databricks (bronze, silver, model, export)
├── dbt/                      # progetto dbt (models/, tests/, macros/)
├── app/                      # Streamlit (UI) + core/ (recommend, bridges, data access)
├── jobs/                     # script programmati (estrazione, outcome a 4 settimane)
├── tests/                    # pytest
└── .github/workflows/        # CI + pipeline programmata
```

Convenzioni: Python 3.11+, type hints, ruff per lint e formattazione, funzioni pure nel `core/` dell'app (niente Streamlit dentro la logica), nomi tabelle in `snake_case`, modelli dbt con prefissi `stg_` / `int_` / tabelle finali senza prefisso.

---

## Roadmap (ordine di lavoro)

**Fase 0 — Setup**
Repo, `pyproject.toml`, ruff, pytest, CI di base, `.env.example`. Account: Google Cloud (sandbox BigQuery), Databricks Free Edition, Hugging Face, Neon, Last.fm API.
✅ *Fatto quando:* la CI passa su un test vuoto.

**Fase 1 — Esplorazione (BigQuery)**
Rispondi con numeri reali a:
- quanti artisti hanno ≥20–30 ascoltatori distinti?
- distribuzione della popolarità, cioè dove mettere le soglie `less_known` / `niche` / `hidden_gems`?
- quanti utenti hanno un comportamento da bot?
- che percentuale di ascolti ha un `artist_mbid`?

Produci la tabella aggregata in Parquet.
✅ *Fatto quando:* c'è un notebook EDA con grafici e le soglie proposte, motivate.

**Fase 2 — Pipeline (Databricks + dbt)**
Bronze → silver → gold, test dbt verdi, `dbt docs` generato.
✅ *Fatto quando:* le tabelle gold esistono e i test passano.

**Fase 3 — Modello e valutazione**
Baseline PMI, ALS, vicini, ponti, backtest con le metriche offline su MLflow.
✅ *Fatto quando:* c'è un confronto documentato ALS vs baseline e le tabelle `artist_neighbors` esportate.

**Fase 4 — Serving + app MVP**
Parquet su Hugging Face, `recommend()` testata, app Streamlit con le schermate 1–2, feedback su Neon, deploy.
✅ *Fatto quando:* l'app è online e utilizzabile da chiunque.

**Fase 5 — Ciclo chiuso**
Lettura della cronologia ListenBrainz/Last.fm, esclusione degli artisti noti, gruppo di controllo, script degli esiti a 4 settimane, schermata 3.
✅ *Fatto quando:* il tasso di scoperta riuscita è calcolabile.

**Fase 6 — Automazione**
Databricks Jobs + GitHub Actions ogni 2 settimane.
✅ *Fatto quando:* un nuovo dump aggiorna l'app senza interventi manuali.

**Dopo l'MVP:** email mensile (Resend, piano gratuito), gestione preferenze, pagina pubblica dei KPI, upload dell'export Spotify, eventuale frontend dedicato (FastAPI + frontend).

---

## Design

Esiste un mockup delle 3 schermate su mobile: tema scuro, un solo accento ambra (`#F2B04A`), titoli in Space Grotesk, testo in Inter. In Streamlit riproducilo in modo semplificato: la priorità è chiarezza e velocità, non la fedeltà al pixel.
