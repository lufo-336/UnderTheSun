# UnderTheSun — contratto API (bozza)

> Bozza da rivedere insieme (WSA + team dati) e **congelare al SAL 1 (20 nov)**.
> Dopo il congelamento i WSA sviluppano su dati di esempio con queste stesse forme di risposta.

## Convenzioni

| Tema | Regola |
|---|---|
| Base URL | `/api` |
| Formato | JSON, campi in `snake_case` |
| Autenticazione | `POST /api/auth/login` restituisce un JWT; le altre chiamate usano `Authorization: Bearer <token>` |
| Date e ore | ISO 8601 in **UTC** (`2026-03-12T10:00:00Z`). Il frontend converte in ora italiana. Le serie orarie in UTC evitano i problemi del cambio ora legale (giorni di 23 e 25 ore) |
| Unità | energia in `kwh`, CO2 in `kg`, soldi in `eur`, potenza in `kwp` (il suffisso è nel nome del campo) |
| Periodi | query `da` e `a` (date `YYYY-MM-DD`, `a` esclusa), `granularita` = `ora` \| `giorno` \| `mese` |
| Paginazione | `?pagina=1&per_pagina=50` → `{ "dati": [...], "totale": 123 }` |
| Errori | status HTTP corretto + `{ "errore": { "codice": "NON_TROVATO", "messaggio": "..." } }` |

## Ruoli

| Ruolo | Può vedere |
|---|---|
| `socio` | solo i propri dati (`/api/me/...`) |
| `operatore` | impianti, anomalie, segnalazioni, KPI aggregati; **non** i consumi dei singoli soci |
| `admin` | tutto, inclusi i dati dei singoli soci |

## Endpoint

### Auth

| Metodo | Percorso | Ruolo | Note |
|---|---|---|---|
| POST | `/api/auth/login` | pubblico | `{ email, password }` |
| GET | `/api/auth/me` | tutti | utente corrente |

```jsonc
// POST /api/auth/login → 200
{
  "token": "eyJhbGciOi...",
  "utente": { "id": 7, "email": "rossi@example.com", "ruolo": "socio", "socio_id": 42 }
}
```

### Area socio

| Metodo | Percorso | Note |
|---|---|---|
| GET | `/api/me/riepilogo?anno=2026` | totali e dettaglio mensile |
| GET | `/api/me/consumi?da=&a=&granularita=` | serie dei consumi |
| POST | `/api/me/simulazione` | simulatore di risparmio (calcolo al volo, non salva) |
| GET | `/api/me/segnalazioni` | le mie segnalazioni |
| POST | `/api/me/segnalazioni` | nuova segnalazione, classificata dall'LLM |

```jsonc
// GET /api/me/riepilogo?anno=2026 → 200
{
  "anno": 2026,
  "kwh_consumati": 2710.4,
  "kwh_condivisi": 812.9,
  "incentivo_eur": 97.55,
  "co2_evitata_kg": 0.0,
  "mesi": [
    { "mese": "2026-01", "kwh_consumati": 260.1, "kwh_condivisi": 31.2, "incentivo_eur": 3.74, "co2_evitata_kg": 0.0 }
  ]
}
```
> I valori numerici sono di forma, non realistici (CO2 a 0 finché il team dati non fissa il fattore ISPRA).

```jsonc
// GET /api/me/consumi?da=2026-03-01&a=2026-03-02&granularita=ora → 200
[
  { "ts": "2026-03-01T00:00:00Z", "kwh": 0.21 },
  { "ts": "2026-03-01T01:00:00Z", "kwh": 0.18 }
]
```

```jsonc
// POST /api/me/simulazione
// input: tutti i campi opzionali; se mancano si usano i dati reali del socio
{
  "consumo_annuo_kwh": 3200,
  "quota_spostata_ore_diurne_pct": 20
}
// → 200
{
  "kwh_condivisi_stimati": 1010.0,
  "incentivo_eur_stimato": 121.2,
  "co2_evitata_kg_stimata": 0.0,
  "confronto_attuale": { "kwh_condivisi": 812.9, "incentivo_eur": 97.55 }
}
```
> Input e formula del simulatore **da definire con il team dati** (vedi team-dati.md, "Calcolo CER").

```jsonc
// POST /api/me/segnalazioni
{ "testo": "Da ieri l'app non mostra più i consumi di marzo", "impianto_id": null }
// → 201
{
  "id": 318, "ts": "2026-11-06T15:02:11Z", "canale": "portale",
  "testo": "Da ieri l'app non mostra più i consumi di marzo",
  "categoria": "informazioni", "urgenza": "bassa", "stato": "aperta", "impianto_id": null
}
```
> Se l'LLM non risponde: 201 comunque, con `categoria` e `urgenza` a `null` (riclassificazione successiva).

### Impianti e anomalie (operatore, admin)

| Metodo | Percorso | Note |
|---|---|---|
| GET | `/api/impianti` | elenco con stato sintetico |
| GET | `/api/impianti/:id` | dettaglio |
| GET | `/api/impianti/:id/produzione?da=&a=&granularita=` | produzione reale + prevista |
| GET | `/api/anomalie?impianto_id=&da=&a=&tipo=` | anomalie rilevate |

```jsonc
// GET /api/impianti → 200
[
  { "id": 3, "nome": "Tetto scuola Pinerolo", "comune": "Pinerolo", "lat": 44.88, "lon": 7.35,
    "kwp": 180.0, "data_attivazione": "2024-04-01", "anomalie_aperte": 1 }
]
```

```jsonc
// GET /api/impianti/3/produzione?da=2026-06-01&a=2026-06-02&granularita=ora → 200
[
  { "ts": "2026-06-01T10:00:00Z", "kwh": 121.4, "kwh_previsti": 124.0 },
  { "ts": "2026-06-01T11:00:00Z", "kwh": 3.1,   "kwh_previsti": 129.8 }
]
```

```jsonc
// GET /api/anomalie?impianto_id=3 → 200
[
  { "id": 55, "impianto_id": 3, "inizio": "2026-06-01T11:00:00Z", "fine": "2026-06-01T15:00:00Z",
    "tipo": "guasto", "score": 0.97 }
]
```

### Segnalazioni (operatore, admin)

| Metodo | Percorso | Note |
|---|---|---|
| GET | `/api/segnalazioni?stato=&categoria=&urgenza=&pagina=` | coda, ordinata per urgenza poi data |
| GET | `/api/segnalazioni/:id` | dettaglio |
| PATCH | `/api/segnalazioni/:id` | `{ stato?, categoria?, urgenza? }` — l'operatore può correggere l'LLM |

> Le correzioni dell'operatore sono preziose per il team dati: misurano quanto sbaglia la classificazione.

### Comunità e soci

| Metodo | Percorso | Ruolo | Note |
|---|---|---|---|
| GET | `/api/comunita/kpi?anno=` | operatore, admin | KPI aggregati |
| GET | `/api/comunita/energia?da=&a=&granularita=` | operatore, admin | immessa, prelevata, condivisa |
| GET | `/api/soci?pagina=` | admin | elenco soci |
| GET | `/api/soci/:id/riepilogo?anno=` | admin | stessa forma di `/api/me/riepilogo` |

```jsonc
// GET /api/comunita/kpi?anno=2026 → 200
{
  "anno": 2026, "soci": 240, "impianti": 8, "kwp_totali": 950.0,
  "kwh_prodotti": 1100000.0, "kwh_condivisi": 640000.0,
  "incentivo_eur": 76800.0, "co2_evitata_kg": 0.0,
  "anomalie_aperte": 2, "segnalazioni_aperte": 17
}
```

```jsonc
// GET /api/comunita/energia?da=2026-06-01&a=2026-06-02&granularita=ora → 200
[
  { "ts": "2026-06-01T10:00:00Z", "kwh_immessi": 610.2, "kwh_prelevati": 380.5, "kwh_condivisi": 380.5 }
]
```

### Assemblee (admin)

| Metodo | Percorso | Note |
|---|---|---|
| GET | `/api/verbali` | elenco con sintesi |
| GET | `/api/verbali/:id` | testo completo + sintesi |

```jsonc
// GET /api/verbali → 200
[ { "id": 4, "data_assemblea": "2026-04-18", "sintesi": "Approvato il bilancio 2025; ..." } ]
```

## Fuori da questa bozza

Registrazione soci, recupero password, gestione impianti da interfaccia (CRUD), notifiche. Si aggiungono solo se avanza tempo.
