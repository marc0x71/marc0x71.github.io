+++
title = "Git Flow: come organizzare il lavoro su un progetto reale"
date = 2026-03-04
description = "Se hai già usato Git per qualche progetto personale, probabilmente sei abituato a lavorare su un unico branch e fare commit quando hai finito qualcosa. Funziona bene finché sei da solo, ma quando entri in un team o il tuo progetto cresce, questa abitudine inizia a creare problemi: come si coordinano più persone? Come si gestisce un bug urgente in produzione mentre si stanno sviluppando nuove funzionalità?"

draft=true

[taxonomies]
tags = ["git"]

[extra]
comments = true
+++

Se hai già usato Git per qualche progetto personale, probabilmente sei abituato a lavorare su un unico branch e fare commit quando hai finito qualcosa. Funziona bene finché sei da solo, ma quando entri in un team o il tuo progetto cresce, questa abitudine inizia a creare problemi: come si coordinano più persone? Come si gestisce un bug urgente in produzione mentre si stanno sviluppando nuove funzionalità?

È qui che entra in gioco **Git Flow**: una metodologia di branching che definisce regole chiare su come organizzare i branch e quando fare i merge. Non è l'unico approccio esistente, ma è uno dei più diffusi ed è quindi cosa buona conoscerlo.

---

## L'idea di fondo

Git Flow si basa su una separazione netta tra ciò che è **in produzione** e ciò che è **in sviluppo**. Per farlo usa due branch principali:

- `main` → rappresenta il codice in produzione, ovvero quello che gli utenti stanno usando. Non si tocca direttamente mai.
- `develop` → è il ramo di lavoro quotidiano, dove convergono le nuove funzionalità prima di essere rilasciate.

Attorno a questi due ruotano branch temporanei con ruoli specifici: `feature`, `release` e `hotfix`. Vediamoli tutti.

---

## 1. Creare il branch di sviluppo

Il punto di partenza è creare il branch `develop` a partire da `main`:

```bash
$ git switch main
$ git switch -c develop
```

Da questo momento `develop` è la nostra base di lavoro. Ogni nuova funzionalità partirà da qui.

```
main    ●──────●
               └──────●  develop
```

---

## 2. Sviluppare una nuova funzionalità con i branch feature

Immagina di dover implementare il login. Invece di lavorare direttamente su `develop`, si crea un branch dedicato:

```bash
$ git switch develop
$ git switch -c feature/login
```

Questo approccio ha un vantaggio importante: il lavoro sulla feature è isolato. Se qualcosa va storto, non compromette il resto. Quando la funzionalità è pronta, si fa il merge su `develop`:

```bash
$ git switch develop
$ git merge --no-ff feature/login
```

Il flag `--no-ff` (**no fast-forward**) merita una spiegazione: senza di esso, Git potrebbe "appiattire" i commit della feature direttamente su `develop`, come se fossero stati fatti lì dall'inizio. Con `--no-ff` invece viene preservata la forma del branch nella cronologia, rendendo chiaro che quei commit appartengono a una specifica feature. È utile soprattutto quando si vuole capire la storia del progetto a posteriori.

Una volta integrata, il branch feature può essere eliminato: ha fatto il suo lavoro.

```bash
$ git branch -d feature/login
```

```
                   ●──────●──────●  feature/login
                  /               \
develop    ●─────●                 ●──────●
```

---

## 3. Preparare il rilascio con il branch release

Quando le funzionalità per la prossima versione sono pronte su `develop`, è il momento di prepararsi al rilascio. Si crea un branch `release`:

```bash
$ git switch develop
$ git switch -c release/0.1.0
```

Su questo branch si eseguono i test (QA), si correggono eventuali bug minori e si fanno le ultime verifiche. La regola importante è che su `release` **non si aggiungono nuove funzionalità**: quelle vanno sempre su `develop`. Il branch release serve solo a stabilizzare ciò che è già presente.

---

## 4. Fare il merge della release in main e develop

Superati i test, è il momento di portare il codice in produzione. Il merge va fatto su entrambi i branch principali:

```bash
$ git switch main
$ git merge --no-ff release/0.1.0
$ git tag v0.1.0

$ git switch develop
$ git merge --no-ff release/0.1.0
```

Il **tag** `v0.1.0` è un'etichetta che segna nella storia del progetto il punto esatto in cui è avvenuto il rilascio. Il formato segue il [Semantic Versioning](https://semver.org/), uno standard molto diffuso basato su tre numeri:

| Numero | Nome | Quando cambia |
|:-------|:-----|:--------------|
| `1`.0.0 | MAJOR | Cambiamenti incompatibili con le versioni precedenti |
| 0.`1`.0 | MINOR | Nuove funzionalità retrocompatibili |
| 0.0.`1` | PATCH | Bug fix retrocompatibili |

Quindi `v0.1.0` è la prima release con funzionalità, mentre `v0.1.1` sarà il primo fix su di essa.

A questo punto `main` rappresenta esattamente la versione `v0.1.0` in produzione: il branch `release/0.1.0` ha esaurito il suo scopo e può essere eliminato. Eventuali fixing su questa versione partiranno direttamente da `main` tramite un branch `hotfix`, come vedremo tra poco.

```bash
$ git branch -d release/0.1.0
```

```
main      ●────────────────────────────────●  ← tag v0.1.0
                                          /
release              ●──────●────────────●
                    /
develop   ●────────●────────────────────────●
```

---

## 5. Gestire un bug urgente in produzione con gli hotfix

Capita: il codice è in produzione e si scopre un bug critico. Non si può aspettare il prossimo ciclo di sviluppo. In questo caso si crea un branch `hotfix` direttamente da `main`, che è la fotografia esatta di ciò che è live:

```bash
$ git switch main
$ git switch -c hotfix/issue_123
```

Si corregge il bug, si fa il commit e poi si riporta la correzione su `main` **e** su `develop` (oppure sul branch `release` attivo, se ce n'è uno in corso):

```bash
# merge in main
$ git switch main
$ git merge --no-ff hotfix/issue_123
$ git tag v0.1.1

# se esiste un branch release attivo
$ git switch release/0.2.0
$ git merge --no-ff hotfix/issue_123

# altrimenti, direttamente in develop
$ git switch develop
$ git merge --no-ff hotfix/issue_123
```

Il merge su `develop` (o sul `release` attivo) è fondamentale: senza di esso il fix esisterebbe solo in produzione e sparirebbe nel prossimo rilascio. Terminato il merge, anche questo branch può essere rimosso:

```bash
$ git branch -d hotfix/issue_123
```

```
main      ●──────────────●──────────●  ← tag v0.1.1
          ↑ tag v0.1.0    \        /
hotfix                     ●──────●
                                  \
develop   ●────────────────────────●──────●
```

---

## Ricapitolando

| Branch | Parte da | Merge in | Scopo |
|:-------|:---------|:---------|:------|
| `develop` | `main` | — | Base di sviluppo continuo |
| `feature/*` | `develop` | `develop` | Sviluppo di una singola funzionalità |
| `release/*` | `develop` | `main` + `develop` | Stabilizzazione pre-rilascio |
| `hotfix/*` | `main` | `main` + `develop` (o `release`) | Fix urgenti in produzione |

---

## Una nota finale

Git Flow è particolarmente adatto a progetti con **release cadenzate** e **versioni multiple in produzione**. Se lavori su un progetto con deploy continuo o in un team piccolo, potresti trovare più comodo un approccio più leggero come **GitHub Flow** o il **trunk-based development**, che riducono il numero di branch attivi. Ma questa è un'altra storia.
