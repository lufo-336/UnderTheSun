# UnderTheSun — brief per il team dati

Ciao Francesco, Giuseppe, Cecilia: questo è il punto di partenza per la parte dati. **È una proposta, non un ordine**: sulla parte dati decidete voi, correggete tutto quello che non torna. L'unica cosa fissa sono le scadenze.

## 1. In una frase

Buby Green è, in pratica, una **Comunità Energetica Rinnovabile (CER)**: impianti fotovoltaici condivisi, soci che consumano, un incentivo del GSE sull'energia "condivisa" che viene ripartito tra i soci. **Il cliente non ci dà dati**: li generiamo noi, partendo il più possibile da fonti reali.

## 2. Cosa ci chiede il PDF (requisiti minimi per voi)

- Dati strutturati e non strutturati puliti e documentati (**data dictionary**).
- Almeno una componente di **analisi/KPI** e una di **elaborazione testi** (classificazione, estrazione, sintesi).
- Eventuale modello predittivo; uso responsabile dell'AI.
- Consegne: dataset, data dictionary, notebook/script documentati, output nel formato concordato, report dei risultati.

Valutazione per voi: correttezza e qualità dei dati, rigore dell'analisi, utilità e affidabilità delle componenti AI.

## 3. Scadenze

| Quando | Cosa vi serve avere |
|---|---|
| **ven 13 nov** | Ricerca fatta (sezione 4), elenco tabelle/colonne rivisto rispetto alla bozza in [architettura.md](architettura.md) |
| **ven 20 nov — SAL 1** | Data dictionary v1 + **CSV di esempio di 1 settimana** per ogni tabella (servono ai WSA per sviluppare). Schema congelato |
| **ven 4 dic** | Dataset completo (2 anni), script batch di previsioni/anomalie/ripartizione, classificazione testi funzionante |
| 4–11 dic | Integrazione con la piattaforma |
| **ven 18 dic** | Report dei risultati + demo |

## 4. Ricerca: dove guardare

| Tema | Fonte | Per cosa |
|---|---|---|
| Regole CER | **GSE** (sezione Comunità energetiche / CACER), DM CER 7/12/2023 n. 414 | Energia condivisa, tariffa incentivante, ripartizione |
| Produzione oraria FV | **PVGIS** (JRC, Commissione UE) — strumento "hourly data", anche via API | Serie orarie reali per una località e una potenza in kWp |
| Meteo orario storico | **Open-Meteo** (Historical Weather API, gratuita) | Irraggiamento, nuvolosità, temperatura → feature per le previsioni |
| Profili di consumo | **ARERA** (fasce F1/F2/F3, cliente domestico tipo ~2.700 kWh/anno), dataset pubblici di smart meter | Generare consumi orari plausibili per famiglie, imprese, enti |
| Fattore CO2 | **ISPRA**, fattori di emissione del settore elettrico nazionale (gCO2/kWh) | `co2_evitata = kwh × fattore` |
| Prezzi energia | **GME** (prezzo zonale orario, zona Nord) | Serve alla parte variabile della tariffa GSE |

Annotate fonte e anno di ogni numero che usate: finisce nella tabella `parametro` e nel report.

## 5. Calcolo CER (da verificare sulle fonti)

Ogni **ora**:

```
kwh_immessi   = somma della produzione immessa da tutti gli impianti
kwh_prelevati = somma dei consumi di tutti i soci
kwh_condivisi = min(kwh_immessi, kwh_prelevati)
```

Incentivo = `kwh_condivisi × tariffa`. Da quanto risulta dal DM CER, per impianti ≤ 200 kW la tariffa premio è circa `80 €/MWh + max(0; 180 − prezzo_zonale)`, con un tetto di 120 €/MWh, più una maggiorazione per il Nord; a questa si aggiunge un piccolo rimborso ARERA. **Verificate tutti i numeri sul sito GSE** prima di usarli.

Da decidere voi:
- **Ripartizione tra i soci**: la regola la sceglie la CER nel suo regolamento. Proposta semplice: in proporzione ai kWh prelevati da ciascun socio nelle ore di condivisione.
- **Cabina primaria**: in una CER vera vale solo la condivisione sotto la stessa cabina. Proposta: tutti sotto la stessa, per semplicità.
- **Tariffa**: oraria con prezzo zonale (più realistico) o media annua (più semplice).

Questo stesso calcolo alimenta il **simulatore** dell'area socio ("cosa succede se sposto il 20% dei consumi di giorno?"): ci serve da voi la formula, il backend la implementa.

## 6. Dataset da generare

Dimensioni proposte: **5–10 impianti, 100–300 soci, 2 anni di dati orari** (≈ 5 milioni di righe: niente di pesante per PostgreSQL). Impianti in comuni veri del Piemonte.

| Tabella | Granularità | Come generarla |
|---|---|---|
| `impianto` | 1 riga per impianto | Comuni reali, lat/lon, kWp tra ~20 e ~200 |
| `socio` | 1 riga per socio | Mix famiglie/imprese/enti, date di adesione scaglionate. Nomi finti |
| `produzione_oraria` | impianto × ora | PVGIS per lat/lon/kWp + meteo + **anomalie iniettate** |
| `meteo_orario` | impianto × ora | Open-Meteo |
| `consumo_orario` | socio × ora | Profilo per tipo socio + stagionalità + weekend + rumore |
| `bolletta` | socio × mese | Aggregato dai consumi × prezzo |
| `segnalazione` | ~200–300 | Testi generati (sezione 7) |
| `verbale` | 6–10 | Testi generati |
| `parametro` | chiave/valore | Fattore CO2, tariffe, con fonte |

Calcolate voi, con script batch (sezione 8): `previsione`, `anomalia`, `ripartizione_mensile`.

### Anomalie da iniettare

Iniettatele apposta **e salvate la "verità"** in un file a parte (`anomalie_vere.csv`): così potete misurare quante ne trova il vostro modello (precision/recall). Idee:

| Tipo | Come appare nei dati |
|---|---|
| Guasto inverter | Produzione a 0 in ore di sole, per ore o giorni |
| Sporcizia | Calo graduale di qualche % per settimane, torna normale dopo la "pulizia" |
| Guasto stringa | Calo fisso (es. −30%) da un certo giorno |
| Neve | Produzione quasi nulla dopo nevicata, con meteo coerente |

## 7. Testi

| Tipo | Quanti | Note |
|---|---|---|
| Email / segnalazioni dei soci | 200–300 | Etichettate con `categoria` e `urgenza` |
| Comunicazioni guasti (interne) | qualche decina | Coerenti con le anomalie iniettate |
| Verbali delle assemblee | 6–10 | Per la sintesi automatica |

Categorie proposte: `guasto_impianto`, `bolletta_pagamenti`, `incentivi_ripartizione`, `adesione_recesso`, `informazioni`, `reclamo`. Urgenza: `alta`, `media`, `bassa`.

**Coerenza**: se l'impianto 3 ha un guasto il 12 marzo, ci devono essere email di soci che se ne accorgono nei giorni dopo. Alla demo questo fa la differenza.

Si possono generare con un LLM (attenzione: variare stile, lunghezza, errori di battitura, toni — email tutte uguali rendono la classificazione finta-facile).

**Classificazione**: la piattaforma chiama un LLM quando arriva una segnalazione. Da voi servono: categorie definitive, **prompt**, e la **misura dell'accuratezza** sul set etichettato (es. "84% di categorie giuste"). Se preferite un classificatore classico (scikit-learn) si può valutare: ditelo presto.

## 8. Modelli e batch

| Componente | Idea di partenza | Metrica |
|---|---|---|
| Previsione produzione | Baseline: produzione attesa da irraggiamento × kWp; poi un modello ML sulle feature meteo | MAE, confronto con baseline |
| Previsione consumi | Profilo medio per tipo socio/ora/giorno; poi ML se serve | MAE |
| Rilevamento anomalie | Confronto reale vs atteso: scarto anomalo per N ore → anomalia | Precision/recall su `anomalie_vere.csv` |
| Ripartizione CER | Formula della sezione 5, per socio e mese | — |

Gli script batch girano offline e **scrivono i risultati direttamente nel database** (connessione letta dalla variabile d'ambiente `DATABASE_URL`, mai scritta nel codice).

## 9. Regole per i file

- **CSV**: UTF-8, separatore `,`, decimali con `.`, intestazioni in `snake_case`, un file per tabella con il nome della tabella.
- **Date/ore in UTC**, formato ISO (`2026-03-12T10:00:00Z`). Evita il problema del cambio ora (giorni di 23/25 ore).
- **Unità nel nome della colonna**: `kwh`, `kwp`, `eur`, `co2_kg`.
- **ID interi**, coerenti tra le tabelle.
- **Seed fisso** negli script di generazione: chiunque rigenera esattamente lo stesso dataset.
- **Nel repo vanno gli script e i CSV di esempio, non i dataset completi** (troppo grandi per git). I dataset completi vanno in `data/output/`, già escluso.

Struttura proposta:

```
data/
  generazione/      script che creano il dataset (seed fisso)
  notebook/         esplorazione e analisi
  batch/            previsioni, anomalie, ripartizione, classificazione
  esempi/           CSV di 1 settimana per i WSA
  output/           dataset completi (non committati)
  data-dictionary.md
```

## 10. Data dictionary

Per ogni colonna: nome, tipo, unità, descrizione, esempio, fonte o regola di generazione. Esempio:

| Tabella | Colonna | Tipo | Unità | Descrizione | Esempio | Fonte |
|---|---|---|---|---|---|---|
| produzione_oraria | kwh | float | kWh | Energia prodotta nell'ora che inizia a `ts` | 121.4 | PVGIS + anomalie iniettate |

## 11. Domande aperte (rispondete voi)

1. Cabina primaria unica o più cabine?
2. Tariffa GSE oraria con prezzo zonale o valore medio?
3. Le comunicazioni guasti sono segnalazioni con canale `interno` o una tabella a parte?
4. LLM o classificatore classico per i testi?
5. Regola di ripartizione dell'incentivo tra i soci?
6. Quali input deve avere il simulatore?
