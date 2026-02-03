# Sequência de Implementação - Quickshell Patched

**Última atualização**: 2026-02-03

Este documento define a ordem de implementação dos planos no quickshell-patched.

**Referência única de crashes e alterações:** [01-quickshell-stability-plan.md](01-quickshell-stability-plan.md) — todos os cenários de crash e as alterações que os resolvem (sem duplicação).

---

## Planos

| ID | Plano | Status | Prioridade | Motivo |
|----|-------|--------|------------|--------|
| 01 | [Estabilidade / Crashes (visão geral)](01-quickshell-stability-plan.md) | — | — | Referência única |
| 03 | [Sync incubation (Steam/Thunar)](03-quickshell-patches.md) | ✅ Implementado | Alta | Item 1 do [01](01-quickshell-stability-plan.md) |
| 04 | [HDMI Disconnect](04-hdmi-disconnect-crash-fix.md) | 📋 Pendente | 🔴 Alta | Revert feito; crash persistiu; ver [04](04-hdmi-disconnect-crash-fix.md) diagnóstico e pendência (fix em onScreenDestroyed depois) |
| — | File watching (.desktop) | 📋 Planejado | Média | Item 3 do [01](01-quickshell-stability-plan.md) — patch a implementar |
| — | Fechar muitas janelas | ✅ Definido | — | Item 4 do [01](01-quickshell-stability-plan.md) — só workaround, sem patch |
| 22 | [Tray Eager Init](22-tray-eager-init.md) | ⏳ Postergado | Baixa | Tray quebrado com sync incubation |

---

## Ordem de Execução Recomendada

1. **Pendente:** 04 — implementar fix em `onScreenDestroyed()` (diagnóstico em 04; não resolver agora).
2. **Próximo:** Item 3 (file watching) — patch com opção para desabilitar.
3. **Futuro:** 22 (Tray). Item 4: nenhuma ação no Quickshell (workaround no script).

---

## Dependências

| Plano | Depende de | Observação |
|-------|------------|------------|
| 04 | — | Fix pendente: onScreenDestroyed migrar janela (ver 04 diagnóstico) |
| 22 | 03 | Sync incubation quebra o tray |
| File watching | — | Patch a implementar; ver 01 |

---

## Patches ativos

Ver tabela “Patches ativos” em [01-quickshell-stability-plan.md](01-quickshell-stability-plan.md).
