---
title: ARchetipo Journal
description: Diario delle sessioni di sviluppo
---

## [2026-06-01] - Documentazione supporto OpenCode

**Attività:**
- Analizzate le modifiche del commit `ac4890f` che introduce il supporto subagenti OpenCode
- Documentato l'approccio in `docs/OpenCode.md` con: architettura agenti, guardrail di fase, installazione, workflow completo (inception→spec→plan→design→implement→autopilot), comportamento del Guardiano, esempi pratici in italiano, risoluzione problemi

**Decisioni:**
- `docs/OpenCode.md` segue lo stile della documentazione italiana esistente (README.it.md)
- Nessuna modifica agli agenti, CLI o skill — pura documentazione

**Appreso:**
- OpenCode supporta `mode: primary` e `mode: subagent` con permessi granular (deny/allow per edit e task)
- I 6 subagenti sono `hidden: true` per non intasare la lista agenti
- La temperatura del primary agent è volutamente bassa (0.1) per comportamento deterministico
- I permessi edit fungono da guardrail di fase: ad esempio `archetipo-design` può scrivere solo in `docs/mockups/**`, `archetipo-implement` non può toccare `.archetipo/*`

---

