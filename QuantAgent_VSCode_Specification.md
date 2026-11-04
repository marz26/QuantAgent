# QuantAgent --- Specifica tecnica e prompt di implementazione

> **Scopo:** fornire a un coding agent in VS Code (GitHub Copilot,
> Claude Code o altro) una specifica completa per generare il codice di
> un agente quantitativo multi-asset che esegue un ciclo di analisi una
> volta al giorno e opera esclusivamente su **Alpaca Paper Trading**.
>
> **Vincolo fondamentale:** il progetto deve poter funzionare con costi
> ricorrenti pari a zero, usando servizi e piani gratuiti entro i
> rispettivi limiti. Non introdurre API a pagamento come requisito. Non
> effettuare mai ordini reali.

------------------------------------------------------------------------

## 0. Istruzioni per il coding agent

Leggi tutta questa specifica prima di modificare il repository.

1.  Prima ispeziona il repository esistente. Non sovrascrivere codice,
    configurazioni o documentazione senza averli esaminati.
2.  Se il repository è vuoto, crea la struttura descritta sotto.
3.  Implementa il progetto in modo incrementale, mantenendo il codice
    semplice, modulare, tipizzato e testabile.
4.  Non limitarti a generare scheletri o TODO: implementa le
    funzionalità richieste, i test e la documentazione.
5.  Non inventare endpoint, parametri o capacità delle API. Per Alpaca,
    consulta la documentazione ufficiale corrente e usa l'SDK/API
    supportata al momento dell'implementazione.
6.  Quando una funzionalità dipende da un piano, una quota gratuita, una
    regione o la disponibilità di uno strumento, documenta il limite e
    rendila configurabile. Non assumere che ogni asset sia negoziabile.
7.  Se mancano credenziali, il programma deve comunque poter essere
    eseguito in modalità `dry-run`/test senza inviare ordini.
8.  Non chiedere né scrivere chiavi API nel codice, nei log, nei test,
    nei commit o nei file di esempio.
9.  Non usare scraping non autorizzato, automazioni del browser per
    simulare chiamate API o metodi per aggirare paywall/rate limit.
10. Prima di concludere, esegui i test disponibili, correggi gli errori
    e riepiloga file creati, decisioni, limiti e istruzioni per l'avvio.

Se devi fare assunzioni, usa quelle indicate in questa specifica e
documentale.

------------------------------------------------------------------------

## 1. Visione del prodotto

Realizzare **QuantAgent**, un agente di paper trading multi-asset con
ciclo giornaliero:

1.  Recupera dati di mercato aggiornati e storici.
2.  Controlla qualità, completezza e timestamp dei dati.
3.  Calcola indicatori e segnali quantitativi in Python.
4.  (Opzionale) invia una sola richiesta giornaliera a un provider LLM
    con piano gratuito per ottenere un commento testuale sul contesto e
    sui segnali.
5.  Calcola una proposta di allocazione del portafoglio.
6.  Applica controlli deterministici di rischio.
7.  Se tutte le condizioni sono soddisfatte, invia ordini **solo**
    all'account Alpaca Paper Trading.
8.  Verifica lo stato degli ordini e aggiorna il report.
9.  Invia un riepilogo Telegram.
10. Registra decisioni, prezzi, ordini, errori e metriche per consentire
    audit e valutazione delle performance.

Il sistema è un progetto educativo e sperimentale. Non promettere
rendimenti, non presentare i segnali come consulenza finanziaria e non
collegare mai un account live.

### Obiettivi non funzionali

-   Python 3.11 o versione successiva compatibile con le dipendenze.
-   Esecuzione locale su macOS e su GitHub Actions Linux.
-   Un solo ciclo pianificato al giorno, dal lunedì al venerdì, con
    avvio manuale possibile.
-   Costo ricorrente zero come obiettivo: servizi esterni opzionali
    devono avere un percorso alternativo gratuito/offline.
-   Architettura modulare, test automatici, logging chiaro e
    riproducibilità.
-   Gestione corretta di fusi orari, giorni festivi, mercati chiusi e
    dati non aggiornati.

------------------------------------------------------------------------

## 2. Assunzioni iniziali e universo investibile

Usare queste impostazioni iniziali, tutte modificabili in `config.yaml`:

-   Capitale iniziale di riferimento: **10.000 USD** (il saldo effettivo
    dell'account Alpaca Paper è la fonte di verità).
-   Mercati iniziali:
    -   azioni quotate negli Stati Uniti;
    -   ETF quotati negli Stati Uniti;
    -   un piccolo elenco configurabile di criptovalute, solo se
        effettivamente supportate dall'account e dall'API Alpaca
        utilizzati.
-   Non assumere che azioni europee, tutti gli ETF o tutte le
    criptovalute siano disponibili.
-   Nessuna leva, nessuna vendita allo scoperto, nessuna opzione,
    futures o strumenti derivati.
-   Niente trading intraday: il sistema valuta il portafoglio una volta
    al giorno.
-   Per la prima versione usare un universo ristretto e configurabile,
    ad esempio:
    -   azioni: `AAPL`, `MSFT`, `NVDA`, `AMZN`, `GOOGL`;
    -   ETF: `SPY`, `QQQ`, `GLD`, `TLT`;
    -   crypto: simboli/coppie da confermare tramite gli asset
        effettivamente negoziabili nell'account.
-   Non inviare ordini per simboli non verificati come tradable.
-   Le liste di simboli sono esempi iniziali, non raccomandazioni di
    investimento.

**Importante:** prima di usare un simbolo, interrogare l'API Alpaca per
verificare che l'asset sia attivo, negoziabile, supportato dal paper
account e coerente con la classe di asset. Non hardcodare disponibilità
non verificate.

### Valuta

Il conto paper può essere denominato in USD. Conservare prezzi e valori
nella valuta nativa e gestire esplicitamente la conversione per
eventuali asset crypto quotati in USD. Se non è disponibile un tasso FX
affidabile e gratuito, non sommare valori in valute diverse: bloccare
l'allocazione cross-currency e riportare chiaramente il problema. Non
usare un cambio inventato o obsoleto.

------------------------------------------------------------------------

## 3. Architettura

Separare rigorosamente:

-   **Data layer:** acquisizione e validazione dei dati.
-   **Quant layer:** indicatori e segnali deterministici.
-   **AI layer:** commento testuale opzionale, mai autorità finale sugli
    ordini.
-   **Portfolio layer:** target allocation e quantità.
-   **Risk layer:** controlli hard-coded che l'LLM non può modificare.
-   **Execution layer:** interfaccia broker con Alpaca paper-only.
-   **Persistence layer:** database e storico.
-   **Reporting layer:** Telegram e report Markdown/JSON.

Flusso:

``` text
GitHub Actions / CLI
        |
        v
Market data collection
        |
        v
Data validation and freshness checks
        |
        v
Indicators + quantitative signals
        |
        +------> Optional LLM market commentary
        |
        v
Portfolio target weights
        |
        v
Risk checks (deterministic, mandatory)
        |
        v
Alpaca Paper Trading API
        |
        v
Order status verification
        |
        v
Database + daily report + Telegram
```

L'LLM non deve poter chiamare direttamente tool di trading, accedere
alle chiavi, modificare il database o inviare ordini. L'LLM restituisce
solo testo/JSON interpretativo. Le decisioni finali sono calcolate e
validate dal codice.

------------------------------------------------------------------------

## 4. Stack tecnologico

Preferire dipendenze mantenute e documentate:

-   Python 3.11+
-   `pandas`, `numpy`
-   `pydantic` per configurazione e validazione dei dati
-   `PyYAML` per `config.yaml`
-   `python-dotenv` solo per sviluppo locale
-   SDK ufficiale Alpaca (`alpaca-py`, se ancora raccomandato dalla
    documentazione corrente)
-   `requests` o `httpx` per Telegram e provider LLM
-   `tenacity` per retry limitati sulle richieste idempotenti
-   `pytest` e `pytest-cov`
-   `ruff` per lint/format
-   SQLite tramite `sqlite3` standard library
-   `matplotlib` opzionale per grafici statici

Non aggiungere un framework web, Docker, Redis, Celery o un database
remoto nella prima versione.

Per indicatori, preferire implementazioni trasparenti con Pandas/NumPy.
Evitare dipendenze pesanti se non necessarie.

------------------------------------------------------------------------

## 5. Struttura del repository

Crea una struttura simile alla seguente:

``` text
quantagent/
├── config/
│   └── config.yaml
├── src/
│   └── quantagent/
│       ├── __init__.py
│       ├── main.py
│       ├── cli.py
│       ├── config.py
│       ├── logging_config.py
│       ├── data/
│       │   ├── __init__.py
│       │   ├── alpaca_data.py
│       │   ├── crypto_data.py
│       │   └── validation.py
│       ├── quant/
│       │   ├── __init__.py
│       │   ├── indicators.py
│       │   ├── signals.py
│       │   └── scoring.py
│       ├── ai/
│       │   ├── __init__.py
│       │   ├── base.py
│       │   ├── gemini_provider.py
│       │   ├── prompts.py
│       │   └── schemas.py
│       ├── portfolio/
│       │   ├── __init__.py
│       │   ├── allocator.py
│       │   ├── sizing.py
│       │   └── rebalancer.py
│       ├── risk/
│       │   ├── __init__.py
│       │   └── controls.py
│       ├── execution/
│       │   ├── __init__.py
│       │   ├── broker_base.py
│       │   ├── alpaca_paper.py
│       │   └── order_manager.py
│       ├── storage/
│       │   ├── __init__.py
│       │   ├── database.py
│       │   └── repositories.py
│       ├── reporting/
│       │   ├── __init__.py
│       │   ├── report.py
│       │   └── telegram.py
│       └── backtesting/
│           ├── __init__.py
│           ├── engine.py
│           └── metrics.py
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── reports/
│   └── .gitkeep
├── .github/
│   └── workflows/
│       └── daily_agent.yml
├── .env.example
├── .gitignore
├── pyproject.toml
├── README.md
└── requirements.txt
```

Adatta i nomi se necessario, ma mantieni le separazioni architetturali.

------------------------------------------------------------------------

## 6. Configurazione e segreti

Creare `.env.example` con nomi delle variabili, mai valori reali:

``` dotenv
ALPACA_API_KEY=
ALPACA_SECRET_KEY=
ALPACA_PAPER=true

GEMINI_API_KEY=
GEMINI_MODEL=

TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=

QUANTAGENT_CONFIG=config/config.yaml
QUANTAGENT_DB_PATH=data/quantagent.sqlite
```

-   `.env` deve essere nel `.gitignore`.
-   Non committare database contenenti dati personali o segreti.
-   Su GitHub Actions usare Repository Secrets.
-   Verificare all'avvio che `ALPACA_PAPER=true`.
-   Se la configurazione indica un endpoint live o un account live,
    terminare immediatamente con errore.
-   Non implementare un'opzione per il trading live.
-   Non stampare mai API key, secret, token o header di autenticazione
    nei log.
-   Fornire istruzioni per creare un account Alpaca Paper Only e
    generare le credenziali.

### `config.yaml`

Creare una configurazione leggibile e validata, con almeno:

-   elenco simboli per classe di asset;
-   timeframe e periodo storico;
-   parametri degli indicatori;
-   pesi del modello quantitativo;
-   target allocation e cash reserve;
-   limiti di rischio;
-   costi simulati/slippage;
-   provider AI abilitato/disabilitato;
-   modalità `dry_run`;
-   soglia minima di liquidità;
-   orari e timezone;
-   impostazioni Telegram;
-   percorso database.

I limiti di rischio devono avere valori predefiniti prudenti e non poter
essere modificati dalla risposta LLM.

------------------------------------------------------------------------

## 7. Raccolta dati

### Fonte principale

Usare Alpaca Market Data API per gli asset supportati, tramite
l'interfaccia documentata e i permessi effettivi dell'account. Prima di
scegliere feed o timeframe, verificare le condizioni del piano gratuito
corrente.

### Requisiti

-   Scaricare almeno 6 mesi di barre giornaliere per gli indicatori
    iniziali.
-   Usare dati adjusted quando appropriato e documentare la scelta.
-   Gestire simboli mancanti, timestamp duplicati, buchi, NaN e dati
    fuori ordine.
-   Controllare che l'ultima barra sia sufficientemente recente.
-   Non interpretare una barra non ancora conclusa come close
    definitivo.
-   Gestire calendario e festività del mercato.
-   Salvare timestamp in UTC e mostrare gli orari in `America/New_York`
    o `Europe/Rome` quando appropriato.
-   Non effettuare richieste una per asset se è possibile usare
    richieste batch.
-   Applicare rate limit e retry con backoff.
-   Se i dati sono incompleti o stantii, non generare ordini per l'asset
    interessato.
-   Registrare provider, timestamp di acquisizione, timeframe e qualità
    dei dati.

### Dati crypto

Verificare quali coppie crypto sono disponibili tramite Alpaca e
utilizzare i relativi dati Alpaca per coerenza con l'esecuzione. Se si
usa un provider esterno gratuito, mantenere separata la fonte e
controllare che simboli, valuta e timestamp siano coerenti. Non
presumere che un dato ottenuto da un provider sia direttamente
eseguibile su Alpaca.

### Data validation

Implementare una funzione che restituisca un esito strutturato per ogni
simbolo:

-   `valid`
-   `stale`
-   `missing`
-   `insufficient_history`
-   `invalid_prices`

Bloccare l'asset se non è `valid`.

------------------------------------------------------------------------

## 8. Quantitative engine

Implementare indicatori giornalieri trasparenti e testati:

-   Simple Moving Average: 20 e 50 sessioni.
-   Exponential Moving Average: 20 sessioni.
-   RSI: 14 sessioni.
-   MACD: 12, 26, 9.
-   ATR: 14 sessioni.
-   Momentum: 1, 3 e 6 mesi (approssimazione documentata in sessioni di
    mercato).
-   Volatilità realizzata: 20 e 60 sessioni.
-   Rendimenti giornalieri.
-   Correlazione rolling tra asset.

Evitare look-ahead bias: ogni decisione deve usare esclusivamente dati
disponibili al momento della decisione. Gli indicatori devono essere
calcolati con finestre che non includono dati futuri.

### Segnali

Creare un sistema di scoring deterministico e configurabile. Ad esempio:

-   trend positivo quando prezzo/medie mobili e medie mobili confermano
    un trend;
-   momentum positivo/negativo;
-   RSI come indicatore di contesto, non come regola isolata;
-   penalizzazione per volatilità eccessiva;
-   nessun segnale quando i dati non superano i controlli.

Output per asset:

``` json
{
  "symbol": "AAPL",
  "asset_class": "equity",
  "as_of": "2026-01-01T00:00:00Z",
  "score": 0.62,
  "signal": "BUY",
  "indicators": {
    "rsi_14": 55.2,
    "sma_20": 190.1,
    "sma_50": 185.4,
    "momentum_1m": 0.03,
    "volatility_20d": 0.18
  },
  "data_quality": "valid"
}
```

I valori sono un esempio di schema, non dati reali.

Prevedere `BUY`, `HOLD`, `SELL` e `NO_TRADE`. La soglia dei segnali deve
essere configurabile e testata.

------------------------------------------------------------------------

## 9. AI analyst --- opzionale e gratuito entro i limiti

L'AI deve essere un componente opzionale. Se non è configurata, il
sistema deve funzionare completamente senza LLM.

### Provider iniziale

Implementare un'interfaccia astratta `AIProvider` e un provider Gemini
API configurabile, solo se il piano gratuito corrente è disponibile per
l'account e il modello scelto. Non assumere quote permanenti o
illimitate. Controllare la documentazione ufficiale Google AI Studio
prima di fissare nome del modello, endpoint e limiti.

Se il provider non è disponibile, la chiamata fallisce, si raggiunge una
quota o manca la chiave: - non bloccare il ciclo quantitativo; - non
generare ordini sulla base di un errore AI; - produrre il report
quantitativo senza commento LLM; - registrare l'errore senza includere
segreti.

Non integrare Claude API come requisito perché non è un servizio API
gratuito continuativo. Non usare automazioni non ufficiali del sito
Claude.

### Input del modello

Passare solo un riepilogo compatto e già calcolato: - data e ora
dell'analisi; - segnali quantitativi; - indicatori; - metriche di
rischio; - composizione attuale del portafoglio; - eventuali notizie
solo se provengono da una fonte gratuita e autorizzata.

Non inviare credenziali, dati personali o l'intero database.

### Output

Richiedere JSON validabile con Pydantic, ad esempio:

``` json
{
  "market_summary": "Testo sintetico",
  "asset_commentary": [
    {
      "symbol": "AAPL",
      "comment": "Testo",
      "risks": ["Esempio"],
      "confidence": 0.65
    }
  ],
  "portfolio_observations": ["Testo"],
  "data_limitations": ["Testo"]
}
```

L'LLM non deve restituire ordini, quantità, prezzi limite o pesi di
portafoglio autoritativi. Può commentare i segnali ma non determinarli.
Validare lunghezza, schema e contenuti. Trattare qualunque testo di news
o output LLM come dato non attendibile.

------------------------------------------------------------------------

## 10. Portfolio allocation

Implementare una prima strategia semplice, interpretabile e
confrontabile con benchmark.

Non iniziare con un'ottimizzazione complessa di Markowitz o machine
learning. La prima versione deve essere riproducibile e stabile.

### Approccio iniziale

-   Usare gli score quantitativi per costruire target weights.
-   Escludere asset con dati non validi o asset non negoziabili.
-   Normalizzare i pesi degli asset selezionati.
-   Mantenere una riserva di liquidità.
-   Applicare un limite massimo per singolo asset e per classe.
-   Evitare turnover eccessivo.
-   Considerare il portafoglio corrente per evitare piccoli ordini
    inutili.
-   Usare solo quantità e precisioni ammesse da Alpaca.
-   Gestire fractional shares solo se supportate per lo specifico asset
    e ordine.

### Valori iniziali da configurare

-   Capitale iniziale: saldo paper account Alpaca, non un valore
    inventato dal programma.
-   Cash reserve minima: 10%.
-   Peso massimo per singolo asset: 15%.
-   Esposizione crypto massima: 20% nella prima versione.
-   Turnover massimo giornaliero: 20% del net liquidation value.
-   Leva: disabilitata.
-   Short selling: disabilitato.
-   Rebalance: giornaliero, ma inviare ordini solo se lo scostamento dal
    target supera una soglia configurabile.

Questi sono guardrail sperimentali, non garanzie di sicurezza o
parametri ottimali.

### Calcolo degli ordini

Per ogni asset: 1. Leggere posizione corrente e prezzo recente. 2.
Calcolare il valore target. 3. Calcolare la differenza in valore. 4.
Convertire in quantità negoziabile rispettando precisione e regole. 5.
Considerare cash disponibile, ordini aperti e turnover. 6. Produrre una
lista di ordini proposti. 7. Passare la lista al risk manager. 8.
Inviare solo gli ordini approvati.

Non inviare ordini duplicati se un ciclo viene rilanciato. Usare
`client_order_id` deterministici o un meccanismo idempotente
documentato, e controllare gli ordini aperti prima di creare nuovi
ordini.

------------------------------------------------------------------------

## 11. Risk management --- controlli obbligatori

I controlli devono essere deterministici, indipendenti dall'LLM e non
disattivabili da una risposta AI.

Implementare almeno:

-   Paper account obbligatorio.
-   Asset attivo e tradable.
-   Dati validi e recenti.
-   Nessun ordine duplicato.
-   Nessun ordine se il mercato è chiuso, salvo asset e tipo di ordine
    esplicitamente compatibili con la sessione.
-   Nessuna leva.
-   Nessuna posizione short.
-   Cash reserve minima.
-   Limite per asset.
-   Limite per classe di asset.
-   Limite di turnover giornaliero.
-   Quantità positiva e valida.
-   Prezzo recente valido.
-   Nessun ordine se il valore dell'account o la posizione non sono
    leggibili.
-   Circuit breaker se il database o l'API broker restituiscono uno
    stato incoerente.
-   `dry_run=true` come default.

Prevedere una modalità `--no-orders` che calcola e mostra gli ordini ma
non li invia.

### Kill switch

Implementare un flag configurabile `TRADING_ENABLED=false` di default.
Per la prima versione, l'invio ordini deve richiedere: -
`ALPACA_PAPER=true`; - `TRADING_ENABLED=true`; - `DRY_RUN=false`; -
account verificato come paper; - tutti i risk checks superati.

Non implementare alcun modo per bypassare il controllo paper-only.

------------------------------------------------------------------------

## 12. Alpaca Paper execution

Usare esclusivamente l'ambiente Paper Trading e le API ufficiali
documentate.

Funzionalità richieste:

-   Verificare autenticazione e account.
-   Verificare che l'account sia paper.
-   Leggere saldo, buying power, equity e cash.
-   Leggere posizioni.
-   Leggere ordini aperti e recenti.
-   Verificare tradability e asset class dei simboli.
-   Inviare ordini paper approvati.
-   Preferire ordini limit o marketable limit con regole prudenti, dopo
    aver verificato i requisiti correnti dell'API. Non inviare ordini
    market su dati stantii o durante condizioni anomale.
-   Polling limitato dello stato ordine.
-   Gestire `new`, `partially_filled`, `filled`, `canceled`, `rejected`
    e altri stati documentati.
-   Gestire fill parziali e ordini non eseguiti.
-   Non assumere che un ordine sia stato eseguito solo perché la
    richiesta API è riuscita.
-   Non reinviare automaticamente ordini in stato incerto: prima
    interrogare il broker per evitare duplicazioni.

Implementare un `BrokerInterface` per isolare il codice Alpaca e rendere
possibili test con un broker finto.

------------------------------------------------------------------------

## 13. Persistenza e idempotenza

Usare SQLite per lo storico applicativo. Alpaca resta la fonte di verità
per cash, equity, posizioni e ordini effettivi.

Tabelle minime:

-   `runs`: un record per ogni esecuzione.
-   `market_snapshots`: dati/indicatori usati per la decisione.
-   `signals`: segnali quantitativi.
-   `allocations`: pesi target e correnti.
-   `orders`: ordini proposti e identificativi broker.
-   `fills`: esecuzioni effettive.
-   `ai_analyses`: output LLM e provider/modello.
-   `daily_metrics`: metriche del portafoglio.

Ogni run deve avere un ID univoco e una data di riferimento. Le
operazioni devono essere idempotenti: un rerun non deve creare ordini
duplicati.

### Persistenza su GitHub Actions

I runner GitHub Actions sono effimeri. Non fare affidamento sul
filesystem locale tra esecuzioni.

Per la prima versione: - salvare lo storico necessario in SQLite; -
utilizzare un GitHub Actions artifact o una cache solo per gli scopi per
cui è appropriato, ma non trattare una cache come storage durevole; -
preferire un commit controllato del database/report su un branch
dedicato `agent-state`, oppure usare un altro meccanismo persistente
gratuito e documentato; - non committare segreti; - usare
lock/concurrency per evitare due run contemporanee; - gestire conflitti
e fallimenti di push senza sovrascrivere stato più recente.

**Valutare attentamente:** un database SQLite versionato può causare
conflitti e crescita del repository. Implementare salvataggio atomico e
backup, e documentare i limiti. Se si sceglie un file JSON di stato,
spiegare quali dati sono salvati e come viene garantita l'idempotenza.
Non dichiarare persistente una soluzione basata soltanto sul filesystem
del runner.

Al riavvio, ricostruire sempre lo stato corrente da Alpaca; il database
locale serve per audit e storico, non per sostituire lo stato broker.

------------------------------------------------------------------------

## 14. Telegram reporting

Implementare notifiche tramite Telegram Bot API, senza librerie non
necessarie.

Inviare un messaggio al termine di ogni run contenente:

-   data e ora;
-   stato del ciclo (successo, parziale, errore);
-   equity e cash;
-   posizioni principali e relativi pesi;
-   variazione giornaliera e rendimento cumulato se calcolabili;
-   segnali principali;
-   ordini proposti, inviati, eseguiti, parziali o rifiutati;
-   eventuali problemi dati/API/LLM;
-   avvertenza che si tratta di paper trading.

Gestire il limite di lunghezza Telegram: dividere i messaggi lunghi in
parti o inviare un riepilogo compatto. Non inviare segreti o
informazioni sensibili.

Se Telegram non è configurato o non funziona, il ciclo deve comunque
completare e salvare il report.

------------------------------------------------------------------------

## 15. Report giornaliero

Salvare un report Markdown e JSON per ogni esecuzione, includendo:

-   run ID;
-   timestamp UTC e timezone locale;
-   mercato/sessione;
-   dati usati e relativa freschezza;
-   segnali;
-   target allocation;
-   risk checks;
-   ordini;
-   stato ordini/fill;
-   metriche account;
-   output AI, se disponibile;
-   errori e warning.

Il report deve distinguere chiaramente: - ordine calcolato; - ordine
approvato; - ordine inviato; - ordine accettato dal broker; - ordine
eseguito.

Non usare mai questi stati come sinonimi.

------------------------------------------------------------------------

## 16. Backtesting e valutazione

Prima di abilitare ordini paper, creare un backtester minimale per
testare la logica quantitativa.

Requisiti:

-   split temporale train/test o walk-forward;
-   nessun look-ahead bias;
-   commissioni e slippage configurabili;
-   gestione della liquidità;
-   benchmark buy-and-hold appropriato;
-   rendimenti giornalieri;
-   CAGR;
-   volatilità annualizzata;
-   Sharpe ratio;
-   Sortino ratio, se implementato correttamente;
-   maximum drawdown;
-   turnover;
-   numero di operazioni;
-   performance per asset class.

Non usare dati futuri per decidere prezzi di esecuzione. Specificare se
il segnale generato a fine giornata viene eseguito alla sessione
successiva. Evitare survivorship bias dove possibile e documentare i
limiti dei dati.

Il backtest non deve essere presentato come prova di redditività futura.

------------------------------------------------------------------------

## 17. CLI e modalità operative

Implementare comandi semplici, ad esempio:

``` bash
# Installazione in ambiente virtuale
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Controllo configurazione e credenziali
python -m quantagent.cli check

# Analisi senza inviare ordini
python -m quantagent.cli run --dry-run

# Esecuzione completa paper, solo se esplicitamente abilitata
python -m quantagent.cli run --paper

# Scarica/aggiorna dati
python -m quantagent.cli update-data

# Esegue backtest
python -m quantagent.cli backtest

# Genera report sullo stato Alpaca
python -m quantagent.cli portfolio
```

Se la struttura del package richiede un comando diverso, documentarlo e
mantenerlo coerente.

Il comando `run --dry-run` deve essere quello predefinito nei test e
negli esempi.

------------------------------------------------------------------------

## 18. GitHub Actions

Creare `.github/workflows/daily_agent.yml` con:

-   trigger `workflow_dispatch`;
-   trigger cron dal lunedì al venerdì;
-   `timezone` gestita esplicitamente nel codice, perché cron GitHub usa
    UTC;
-   avvio prudente dopo la chiusura della sessione precedente o quando i
    dati giornalieri sono effettivamente disponibili, non un orario
    arbitrario prima dell'apertura;
-   `permissions` minime;
-   `concurrency` per evitare esecuzioni sovrapposte;
-   timeout;
-   setup Python;
-   installazione dipendenze;
-   esecuzione test essenziali;
-   caricamento secrets come environment variables;
-   esecuzione `--dry-run` per default iniziale;
-   report come artifact;
-   persistenza dello stato con il meccanismo documentato;
-   notifica Telegram anche in caso di fallimento, se configurata.

Non abilitare automaticamente il trading quando il workflow viene
creato. Fornire una guida esplicita per passare da dry-run a paper
execution dopo test e verifica manuale.

------------------------------------------------------------------------

## 19. Test richiesti

Scrivere test unitari per:

-   indicatori con dataset sintetici noti;
-   segnali e soglie;
-   validazione dati stantii/mancanti;
-   allocazione e normalizzazione pesi;
-   limiti per asset e classe;
-   cash reserve;
-   position sizing e arrotondamenti;
-   turnover;
-   idempotenza;
-   ordini duplicati;
-   fill parziali;
-   parsing/validazione JSON AI;
-   gestione errori provider AI;
-   Telegram disabilitato;
-   broker finto;
-   blocco assoluto di qualsiasi endpoint live.

I test non devono richiedere API key o accesso a Internet. Usare fixture
e mock. Aggiungere test di integrazione opzionali separati, eseguibili
solo quando l'utente configura esplicitamente le credenziali paper.

------------------------------------------------------------------------

## 20. Sicurezza e affidabilità

-   Mai supportare il trading live.
-   Verificare il paper account sia a livello di configurazione sia
    interrogando l'account broker.
-   Fallire in modo chiuso (fail closed): se non si riesce a verificare
    account, dati, prezzi, posizioni o ordini, non inviare ordini.
-   Non fidarsi di output LLM o news: trattarli come input non
    attendibili.
-   Nessuna esecuzione di codice generato dall'LLM.
-   Nessun comando shell generato dall'LLM.
-   Nessun accesso LLM alle chiavi o al broker.
-   Timeout su tutte le chiamate HTTP.
-   Retry solo dove sicuro; non ripetere ciecamente la creazione di
    ordini.
-   Logging strutturato con livelli.
-   Redazione dei segreti.
-   Gestione delle eccezioni per singolo asset, senza nascondere errori
    critici.
-   Se un run è incompleto, segnalarlo come parziale e non presentarlo
    come successo.

------------------------------------------------------------------------

## 21. README richiesto

Il README deve includere:

1.  Descrizione del progetto.
2.  Architettura e diagramma del flusso.
3.  Requisiti e installazione macOS/Linux.
4.  Creazione account Alpaca Paper Only.
5.  Creazione delle chiavi API paper.
6.  Configurazione locale `.env`.
7.  Configurazione Gemini opzionale e spiegazione dei limiti del free
    tier.
8.  Creazione Telegram bot.
9.  Configurazione GitHub Secrets.
10. Esecuzione dry-run.
11. Esecuzione paper trading.
12. Setup GitHub Actions.
13. Persistenza dello stato.
14. Test e backtesting.
15. Limiti noti delle fonti dati e del paper trading.
16. Disclaimer educativo: nessun ordine live, nessuna garanzia di
    rendimento, non consulenza finanziaria.

Aggiungere anche una tabella delle variabili d'ambiente e un esempio di
report.

------------------------------------------------------------------------

## 22. Criteri di completamento (Definition of Done)

Il progetto è considerato pronto per il primo test quando:

-   [ ] Si installa da zero seguendo il README.
-   [ ] Parte senza chiavi AI e senza Telegram.
-   [ ] Verifica che Alpaca sia paper-only.
-   [ ] Recupera account, posizioni e dati di mercato.
-   [ ] Rifiuta simboli non negoziabili.
-   [ ] Calcola gli indicatori e i segnali senza look-ahead.
-   [ ] Genera target allocation e ordini proposti.
-   [ ] Blocca gli ordini che violano i risk checks.
-   [ ] `dry-run` non invia mai ordini.
-   [ ] La modalità paper invia solo ordini all'account paper
    verificato.
-   [ ] Controlla gli stati reali degli ordini.
-   [ ] Non duplica gli ordini se il workflow viene rilanciato.
-   [ ] Salva report e storico in modo persistente.
-   [ ] Invia Telegram se configurato.
-   [ ] Funziona anche se il provider AI è assente o ha raggiunto la
    quota.
-   [ ] Tutti i test unitari passano.
-   [ ] GitHub Actions può essere lanciato manualmente.
-   [ ] Il workflow schedulato non abilita trading automaticamente.

------------------------------------------------------------------------

## 23. Ordine di implementazione

Non generare tutto in un unico blocco non verificato. Procedi per
milestone e mantieni il repository funzionante dopo ciascuna:

### Milestone A --- Fondamenta

-   struttura progetto;
-   configurazione;
-   logging;
-   CLI;
-   database;
-   test e README iniziale.

### Milestone B --- Dati e quantitative engine

-   connessione dati;
-   validazione;
-   indicatori;
-   segnali;
-   test sintetici.

### Milestone C --- Portfolio e backtesting

-   allocator;
-   risk checks;
-   sizing;
-   backtester;
-   metriche.

### Milestone D --- Alpaca paper

-   broker interface;
-   verifica paper-only;
-   lettura account/posizioni;
-   ordini paper;
-   verifica fill;
-   test con mock.

### Milestone E --- AI e notifiche

-   provider AI opzionale;
-   schema output;
-   Telegram;
-   report.

### Milestone F --- Automazione

-   GitHub Actions;
-   secrets;
-   persistenza;
-   concurrency;
-   test workflow manuale.

Alla fine di ogni milestone: 1. esegui i test; 2. correggi gli errori;
3. aggiorna il README; 4. elenca cosa è implementato e cosa resta da
fare.

------------------------------------------------------------------------

## 24. Prima azione richiesta al coding agent

Inizia ispezionando il repository e il suo stato. Poi:

1.  Presenta un breve piano basato su questa specifica.
2.  Segnala eventuali incompatibilità o punti che richiedono una
    verifica nella documentazione ufficiale corrente.
3.  Implementa la Milestone A completa.
4.  Esegui i test e mostra i risultati.
5.  Attendi una verifica prima di procedere con le milestone successive,
    a meno che l'utente chieda esplicitamente di continuare.

**Priorità assolute:** paper-only, zero costi obbligatori, controlli di
rischio deterministici, nessun ordine duplicato, dati validati e codice
testabile.
