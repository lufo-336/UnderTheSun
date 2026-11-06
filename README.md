# UnderTheSun

Portale della Comunità Energetica Rinnovabile (CER) di **Buby Green** — progetto del Laboratorio Integrato ITS (Data Analyst / AI Specialist + Web Solutions Architect), team **PlusUltra**.

> Stato: **bozza pre-SAL 1**. Schema dati e contratto API si congelano al SAL 1 (ven 20 nov 2026).

## Cosa fa

- **Soci** (famiglie, imprese, enti): vedono consumi, energia condivisa, incentivo CER e CO2 evitata; simulano il risparmio; aprono segnalazioni.
- **Operatori**: monitorano gli impianti (produzione vs previsione, anomalie) e gestiscono le segnalazioni, classificate automaticamente.
- **Admin**: KPI della comunità, soci, verbali delle assemblee con sintesi.

## Team

| Persona | Ruolo |
|---|---|
| Luca Foglia | Team leader, architettura, DevOps (WSA) |
| Mirko Di Vincenzo | Frontend (WSA) |
| Stefano Spadaccini | Full-stack (WSA) |
| Geoffrey JR Etinosa | Full-stack (WSA) |
| Francesco Colohan | Data Analyst / AI |
| Giuseppe Tommasi | Data Analyst / AI |
| Cecilia Montebello | Data Analyst / AI |

## Struttura del repo

```
data/        script di generazione, notebook, batch (previsioni, anomalie, ripartizione CER) — team dati
backend/     API Node + Express — WSA
frontend/    React — WSA
docs/        architettura, contratto API, brief per il team dati
```

## Documentazione

- [Architettura e modello dati](docs/architettura.md)
- [Bozza contratto API](docs/api.md)
- [Brief per il team dati](docs/team-dati.md) ← **data analyst: partite da qui**
- [Guida rapida a Git](docs/git.md) ← **mai usato Git? leggi qui**

## Stack (proposta)

React · Node + Express · PostgreSQL · Python (pandas, scikit-learn) · LLM via API · Docker Compose

## Calendario

| Periodo | Obiettivo |
|---|---|
| 6–13 nov | Repo, ricerca fonti, bozza entità, wireframe pagine, Postgres in docker |
| 13–20 nov | Schema + data dictionary + contratto API, dati di esempio. **SAL 1 ven 20 nov** |
| 20 nov – 4 dic | WSA sviluppano su dati di esempio; team dati produce dataset completo, modelli, NLP |
| 4–11 dic | Integrazione con i dati reali |
| 11–18 dic | Margine, bugfix, documentazione, prova demo. **Presentazione finale ven 18 dic** |

## Regole di lavoro

- Niente push diretti su `main`: branch + Pull Request + almeno una review.
- Mai committare chiavi API o file `.env`.
- Mai committare dataset completi: si committano gli script (con seed fisso) e i CSV di esempio.
