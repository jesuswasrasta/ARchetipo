# Changelog

Tutti i cambiamenti rilevanti di ARchetipo sono documentati in questo file.

Il formato segue [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
con categorizzazione [Conventional Commits](https://www.conventionalcommits.org/).

## [Unreleased]

### Added

- **Supporto subagenti OpenCode.** Creati 6 file agente markdown in `agents/` (`archetipo-inception`, `archetipo-spec`, `archetipo-plan`, `archetipo-design`, `archetipo-implement`, `archetipo-autopilot`) che definiscono subagenti per il Task tool di OpenCode, con permission calibrate come guardrail di fase.
- `archetipo init --tool opencode` ora installa anche gli agenti in `.opencode/agents/`, oltre alle skill in `.opencode/skills/`.
- La tabella di compatibilità di `archetipo-autopilot` segna OpenCode come **Supported**.

### Changed

- `toolDef` in `cli/internal/cli/init_project_cmd.go` espone il nuovo campo `AgentsPath` per installare agenti tool-specifici.
- `scripts/build-npm.mjs` sincronizza la directory `agents/` nel pacchetto npm `@techreloaded/archetipo`.

### Security

- Ogni agente ha permessi che riflettono il vincolo di fase (guardrail), ad esempio:
  - `archetipo-design` può editare solo `docs/mockups/**`, mai source code
  - `archetipo-implement` può editare source code ma non `.archetipo/*` (backlog/piani)
  - `archetipo-autopilot` può editare solo il file di stato, mai source code o PRD

## [0.4.0] - 2025-04-25

### Added

- Skill `archetipo-autopilot`: pipeline autonoma plan→implement su tutte le spec TODO, con gestione stato, resumption e strategie di errore configurabili.
- Skill `archetipo-design`: mockup isolati in `docs/mockups/` con HTML/CSS/JS standalone, personaggio ✨ Livia.
- Supporto multipiattaforma: 6 pacchetti npm per piattaforma (`@techreloaded/archetipo-{os}-{cpu}`).
- Shim Node.js (`npm/archetipo/bin/archetipo.js`) che risolve il binario nativo Go per la piattaforma corrente.

### Changed

- `archetipo-plan` ora spawna `archetipo-design` come subagente per la generazione mockup (dove disponibile), con fallback inline.
- `archetipo-implement` introduce worker-backed execution preferita rispetto a in-context, con wave di task e code review strutturata (personaggio 🔍 Cesare).
- Il template config YAML sposta `backlog` e `planning` da `paths` a `file`, con messaggio di errore esplicito per config legacy.

## [0.3.0] - 2025-04-10

### Added

- Connector GitHub: backlog e planning via GitHub Issues + Projects v2 (GraphQL), con `gh` CLI come unica dipendenza runtime.
- Conformance suite (`cli/internal/connector/conformance/`) eseguita su tutti e tre i connector (filefs, github, inmemory).
- Comando `archetipo view`: server web locale con board Kanban, SSE broker e file watcher.
- Comando `archetipo update`: notifica automatica di nuove versioni npm disponibili.

### Changed

- La CLI usa un pattern connector registry (`Register`/`New`) con tre implementazioni registrate via package `builtin`.
- Tutte le query GraphQL del connector github vivono in `templates.go` isolate, con snapshot test.
- `archetipo init` supporta selezione interattiva del tool e connector, con flag `--tool` ripetibile e `--yes`.

## [0.2.0] - 2025-03-28

### Added

- Skill `archetipo-plan` con team virtuale (Emanuele, Leonardo, Ugo, Mina) per pianificazione tecnica.
- Skill `archetipo-implement` con team virtuale (Ugo, Mina, Cesare) per implementazione, test e code review.
- Comandi CLI: `spec plan`, `spec start`, `task done`, `spec review`, `spec move`.

### Changed

- Le skill caricano `shared-runtime.md` una sola volta all'attivazione, non ripetutamente.
- I template statici vengono tradotti nella lingua rilevata (Language Policy), mantenendo i placeholder intatti.
- L'envelope JSON unificato (`iox`) separa stdout (successo) da stderr (errore), con branch su `error.code`.

## [0.1.0] - 2025-03-15

### Added

- Prima release pubblica.
- Skill `archetipo-inception` per product discovery e generazione PRD (personaggi 💎 Andrea, 🧭 Costanza, ✏️ Lisa).
- Skill `archetipo-spec` per creazione backlog iniziale ed estensione da PRD esistente.
- CLI Go (`archetipo`) con comandi `config show`, `prd write`, `spec add|show|next|list`, `init`, `version`.
- Connector `file`: backlog YAML, PRD markdown, piani YAML in `.archetipo/`.
- Runtime condiviso: `.archetipo/config.yaml`, `.archetipo/shared-runtime.md`.
- Pacchetto npm `@techreloaded/archetipo` con installazione globale.
