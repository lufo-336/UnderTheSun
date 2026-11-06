# Guida rapida a Git per UnderTheSun

Per chi non ha mai usato Git. Bastano i comandi di questa pagina.

## Cosa sono

- **Git**: tiene la storia di ogni modifica ai file. Ogni "salvataggio" si chiama **commit**.
- **GitHub**: il sito dove sta la copia condivisa del repo (https://github.com/lufo-336/UnderTheSun).
- **Branch**: una "copia di lavoro" separata. Ognuno lavora sul suo branch, poi le modifiche entrano in `main` tramite una **Pull Request (PR)** che un collega approva.

> Su `main` **non si può fare push diretto**: GitHub lo blocca. Si passa sempre da branch + PR.

## 1. Preparazione (una volta sola)

1. Crea un account su GitHub e **accetta l'invito** al repo (arriva via email, oppure da https://github.com/notifications).
2. Installa Git:
   - **Windows**: https://git-scm.com/download/win (lascia le opzioni predefinite)
   - **macOS**: apri il Terminale e scrivi `git --version`, ti propone l'installazione
   - **Linux**: dal gestore pacchetti (`sudo apt install git`, `sudo dnf install git`)
3. Apri un terminale (su Windows: **Git Bash**) e configura nome ed email (la stessa di GitHub):
   ```bash
   git config --global user.name "Nome Cognome"
   git config --global user.email "tua@email.it"
   ```
4. Scarica il repo nella cartella dove vuoi tenerlo:
   ```bash
   git clone https://github.com/lufo-336/UnderTheSun.git
   cd UnderTheSun
   ```
   Al primo push si apre il browser per il login a GitHub: accetta e basta.

## 2. Il giro di ogni giorno

```bash
# 1. parti da main aggiornato
git switch main
git pull

# 2. crea il tuo branch (nome breve, senza spazi)
git switch -c dati/generazione-consumi

# 3. lavora sui file... poi guarda cosa hai cambiato
git status

# 4. scegli cosa salvare e fai il commit
git add data/generazione/consumi.py
git commit -m "Generazione consumi orari per tipo socio"

# 5. manda il branch su GitHub
git push -u origin dati/generazione-consumi
```

Poi su GitHub compare il bottone **"Compare & pull request"**:

1. Aprila, scrivi due righe su cosa hai fatto.
2. Chiedi a un collega di approvarla (a destra, "Reviewers").
3. Approvata → **Merge pull request** → **Delete branch**.
4. Sul tuo PC torna a `main` aggiornato: `git switch main && git pull`.

Puoi fare tutti i commit e push che vuoi sullo stesso branch prima di aprire la PR: dopo il primo `push -u` basta `git push`.

## 3. Regole del team

- **Nomi dei branch** con prefisso: `dati/...`, `backend/...`, `frontend/...`, `docs/...`.
- **Commit piccoli** con messaggio che dice *cosa* cambia: "Aggiunge anomalie di tipo neve", non "fix" o "aggiornamento".
- **PR piccole**: meglio tre PR da un'ora che una da una settimana.
- **Mai committare**:
  - chiavi API, password, file `.env`
  - dataset completi (vanno in `data/output/`, già escluso): si committano gli script e i CSV di esempio
  - file enormi (GitHub rifiuta file oltre 100 MB)
- **Notebook Jupyter**: prima del commit fai *Kernel → Restart & Clear Output* (o "Clear All Outputs" in VS Code). Gli output gonfiano i file e rendono le PR illeggibili.

## 4. Problemi comuni

**"push declined" / "protected branch" su `main`**
Stai provando a fare push su `main`. Sposta il lavoro su un branch:
```bash
git switch -c dati/nome-del-lavoro
git push -u origin dati/nome-del-lavoro
```
Poi rimetti a posto il tuo `main` locale (le modifiche sono salve nel branch appena creato):
```bash
git switch main
git reset --hard origin/main
```

**Il mio branch è indietro rispetto a `main`**
```bash
git switch il-mio-branch
git pull origin main
```

**Conflitto** (`CONFLICT` dopo un pull)
Due persone hanno cambiato le stesse righe. Apri i file indicati, trovi blocchi così:
```
<<<<<<< HEAD
la tua versione
=======
la versione degli altri
>>>>>>> main
```
Tieni la versione giusta, cancella le righe con `<<<<<<<`, `=======`, `>>>>>>>`, poi:
```bash
git add file-sistemato
git commit
```
VS Code mostra i conflitti con i bottoni "Accept Current / Incoming / Both". Nel dubbio, chiedi prima di forzare.

**Ho sbagliato il messaggio dell'ultimo commit (non ancora pushato)**
```bash
git commit --amend -m "Messaggio giusto"
```

**Ho aggiunto un file che non volevo (non ancora committato)**
```bash
git restore --staged nome-file
```

**Ho committato una chiave API**
Avvisa subito Luca e **revoca la chiave** dal sito del fornitore: anche se cancelli il file, resta nella storia del repo (che è pubblico).

## 5. Alternative senza terminale

- **VS Code**: pannello *Source Control* (icona con i rami a sinistra). Commit, push, pull e cambio branch dalla barra in basso.
- **GitHub Desktop** (https://desktop.github.com): tutto con i bottoni.

I concetti restano gli stessi: branch → commit → push → PR → review → merge.

## 6. Mini glossario

| Termine | Significato |
|---|---|
| repo | la cartella del progetto con tutta la sua storia |
| clone | scaricare il repo la prima volta |
| commit | un salvataggio con messaggio |
| push | mandare i tuoi commit su GitHub |
| pull | scaricare i commit degli altri |
| branch | linea di lavoro separata |
| PR (Pull Request) | richiesta di unire il tuo branch a `main` |
| merge | unione di un branch in un altro |
| conflitto | due modifiche sulle stesse righe, va scelta a mano |
