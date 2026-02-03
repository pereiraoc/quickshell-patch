# Plano de Estabilidade — Crashes do Quickshell

**Última atualização**: 2026-02-03

Este documento é a referência única para todos os cenários de crash conhecidos e as **decisões** (alterações no Quickshell ou workarounds) que os resolvem. Cada item tem uma decisão única e tradeoffs explícitos. Detalhes de implementação ficam nos docs referenciados.

---

## Visão geral

| # | Crash | Decisão | Status | Detalhe |
|---|-------|---------|--------|---------|
| 1 | Steam / Thunar (race QML incubation) | Patch sync incubation + steam.desktop + Thunar env vars | ✅ Implementado | [03](03-quickshell-patches.md) |
| 2 | HDMI desconectar | Reverter `surface.cpp`; manter sync incubation | 📋 Planejado | [04](04-hdmi-disconnect-crash-fix.md) |
| 3 | Qt6 QML file watching (.desktop) | Patch: opção para desabilitar file watching em desktop entries | 📋 Planejado | Esta seção |
| 4 | Fechar muitas janelas (workspace) | Apenas workaround (delay 50 ms no script); sem patch no Quickshell | ✅ Definido | Esta seção |

---

## 1. Steam / Thunar (incubation)

- **Gatilho:** Abrir Steam ou Thunar; fechar Thunar duas vezes rápido.
- **Decisão:** Patch de incubação síncrona em `lazyloader.cpp` e `boundcomponent.cpp`; steam.desktop com `+open_steam_url 0`; Thunar com env vars no .desktop (stage4/45).
- **Tradeoffs:** Tray quebrado (postergado 22); UI um pouco menos responsiva em updates dinâmicos; mantemos um patch no fork.
- **Status:** Implementado. [03-quickshell-patches.md](03-quickshell-patches.md).

---

## 2. HDMI desconectar

- **Gatilho:** Desconectar monitor HDMI com layer surface (barra) ativa.
- **Decisão:** Reverter a modificação não comitada em `surface.cpp` (remover `output=nullptr`). Não adotar upstream puro nem `output=nullptr`; confiar nos fixes upstream já no fork (proxywindow, layer shell bridge).
- **Tradeoffs:** Mantemos o fork com o patch de sync incubation (não testamos upstream puro como padrão); ganho é multi-monitor correto e HDMI estável sem workarounds externos.
- **Status:** Teste executado (2026-02-02): **crash persistiu** após reverter `surface.cpp`. Diagnóstico documentado em [04](04-hdmi-disconnect-crash-fix.md): causa raiz em `onScreenDestroyed()` que só zera `mScreen` sem migrar a janela para outro screen, deixando QWindow/árvore QML a referenciar screen destruído; no próximo polishItems/incubate ocorre use-after-free. **Pendência:** corrigir depois no fork (não upstream), implementando em `onScreenDestroyed()` migração da janela para primary screen ou esconder/destruir janela. Ver [04](04-hdmi-disconnect-crash-fix.md) seções "Resultado do teste" e "Diagnóstico do motivo real".

---

## 3. Qt6 QML file watching (.desktop)

- **Gatilho:** Modificação de arquivos `.desktop` com o shell rodando (instalação de pacotes, apps tocando em .desktop).
- **Decisão:** Implementar no Quickshell uma opção (build-time ou config) para **desabilitar** file watching em desktop entries. Quando desativada, o launcher não atualiza a lista de .desktop em tempo real — usuário precisa reiniciar o shell (ou recarregar) para ver novos ícones.
- **Tradeoffs:** Com a opção desativada: não crasha ao instalar pacotes ou ao mexer em .desktop; lista de apps do launcher fica “congelada” até reinício/recarga. Workaround atual (scripts matam shell antes de atualizar .desktop) continua válido onde já existe; o patch reduz crashes em uso normal sem script.
- **Status:** Planejado; patch a implementar. Workaround atual: scripts em caelestia-arch-setup matam shell antes de atualizar .desktop.

---

## 4. Fechar muitas janelas (workspace)

- **Gatilho:** Fechar várias janelas em sequência muito rápida (ex.: “limpar workspaces”).
- **Decisão:** **Não** implementar patch no Quickshell para este cenário. Manter apenas workaround: delay de 50 ms entre cada `closewindow` no script de workspace navigation (caelestia-arch-setup).
- **Tradeoffs:** Quem usa o script de “limpar workspaces” fica protegido. Fechar muitas janelas manualmente ou por outro fluxo muito rápido pode ainda crashar; aceitamos esse risco e não adicionamos complexidade no Quickshell por enquanto.
- **Status:** Definido; sem alteração planejada no código.

---

## Patches ativos (resumo)

| Arquivo | Alteração | Referência |
|---------|-----------|------------|
| `src/core/lazyloader.cpp` | `QQmlIncubator::Synchronous` | 03 |
| `src/core/boundcomponent.cpp` | `QQmlIncubator::Synchronous` | 03 |
| `src/wayland/wlr_layershell/surface.cpp` | Não manter `output=nullptr` (reverter se presente) | 04 |

---

## Ordem recomendada

1. **Imediato:** Executar 04 (reverter `surface.cpp`).
2. **Próximo:** Implementar item 3 (opção para desabilitar file watching em desktop entries).
3. **Item 4:** Nenhuma ação no Quickshell; manter workaround no script.

[00-sequencia-implementacao.md](00-sequencia-implementacao.md)
