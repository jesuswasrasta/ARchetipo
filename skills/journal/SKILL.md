---
name: journal
description: >
  Salva sul journal quando Nando dice "salva sul journal",
  "ricordati questo", "chiudi sessione".
---

# Journal Workflow

## 1. Prepara

Se `JOURNAL.md` ha modifiche non committate, chiedi a Nando se
committarle prima di procedere. Se preferisce non farlo, procedi
comunque.
Se trovi entry fuori ordine cronologico, riordinale senza chiedere.

## 2. Insight (opzionale)

Se la sessione ha prodotto conoscenza riutilizzabile, proponi a
Nando di salvarla con `hindsight save` prima di scrivere il journal.
Se `hindsight` non è disponibile, salta questo passo.

## 3. Scrivi

Aggiungi la nuova entry in **cima** a `JOURNAL.md`, subito dopo
il frontmatter YAML (ordine cronologico inverso). Non modificare
il frontmatter.

```
## [YYYY-MM-DD] - [Titolo sessione]

**Attività:**
- [cosa è stato fatto]

**Decisioni:**
- [decisioni con motivazione e impatto]

**Appreso:**
- [opzionale]

**Da ricordare:**
- [opzionale]

---
```

## 4. Committa

Staged `JOURNAL.md` e committa con conventional commit in italiano.
Chiedi a Nando se vuole includere altri file.
