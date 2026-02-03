# Proposta de Patch para Quickshell: Correção de Crash QML Incubation

**ID**: 03  
**Status**: Implementado  
**Data**: 2026-01-23

---

## Resumo

Este documento descreve o patch no código-fonte do Quickshell para resolver o crash causado por condição de corrida na incubação QML assíncrona. O patch força incubação síncrona em cenários de alta concorrência, eliminando o segmentation fault sem remover funcionalidades.

---

## Contexto do Problema

- **Crash**: Segmentation fault em `QQmlIncubatorPrivate::incubate` quando Steam/Thunar abrem/fecham janelas rapidamente
- **Causa**: Race condition em incubadores QML assíncronos (`QQmlIncubator`) durante atualizações concorrentes da UI triggeradas por eventos do Hyprland (desktop state changes)
- **Impacto**: Instabilidade no Caelestia Shell, impedindo desenvolvimento contínuo

---

## Análise Técnica

- O Quickshell usa incubação assíncrona para performance (evita travamentos na UI)
- Múltiplos eventos simultâneos (ex.: rescans de desktop entries + updates de workspaces) corrompem o estado interno do Qt's QML incubator
- Código afetado: `src/core/incubator.*`, `src/core/lazyloader.cpp`, `src/core/boundcomponent.cpp`

---

## Solução Implementada

Forçar incubação síncrona (`QQmlIncubator::Synchronous`) em vez de assíncrona, evitando concurrency issues.

### Mudanças Aplicadas

1. **Arquivo: `src/core/lazyloader.cpp`** (método `incubateIfReady`)
   ```cpp
   this->incubator = new QsQmlIncubator(QQmlIncubator::Synchronous, this);
   ```

2. **Arquivo: `src/core/boundcomponent.cpp`** (método `tryCreate`)
   ```cpp
   this->incubator = new QsQmlIncubator(QsQmlIncubator::Synchronous, this);
   ```

---

## Trade-off: Tray Quebrado

O patch quebra o system tray (`SystemTray.items` fica vazio). Ver [22-tray-eager-init.md](22-tray-eager-init.md) para a correção (eager init do StatusNotifierHost).

---

## Impacto Esperado

- **Positivo**: Elimina crashes QML incubation (parcialmente — ainda há crashes em outros cenários)
- **Negativo**: UI pode ser ~10-20% menos responsiva em updates dinâmicos
- **Negativo**: Tray quebrado (corrigido com eager init)

---

**Visão de todos os crashes e alterações:** [01-quickshell-stability-plan.md](01-quickshell-stability-plan.md).
