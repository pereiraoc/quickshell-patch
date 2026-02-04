# HDMI Disconnect Fix - Plano de Implementação

**Status**: 📋 Pronto para Implementar  
**Prioridade**: 🔴 Alta  
**Complexidade**: 🟡 Média  
**Tempo Estimado**: 2-3 horas  
**Confiança**: 85%

---

## 🎯 Objetivo

Corrigir crash ao desconectar HDMI enquanto layer surface (barra) está ativa, permitindo uso confiável de multi-monitor.

---

## 🔍 Diagnóstico Resumido

**Crash**:
```
general protection fault in libQt6Core.so.6.10.1
#0  QObject::metaObject()
#5  QQmlIncubatorPrivate::incubate
#12 QQuickWindowPrivate::polishItems
```

**Causa Raiz**:
1. Layer surface criada em `wl_output` específico (HDMI)
2. HDMI desconecta → compositor destrói `wl_output` → Qt destrói `QScreen`
3. `onScreenDestroyed()` só zera `mScreen`, não migra janela
4. `QWindow` continua referenciando screen destruído
5. Event loop → `polishItems()` → acesso a objeto destruído → **use-after-free** → SIGSEGV

**Detalhes**: Ver `01-quickshell-stability-final.md` seção "3. HDMI Disconnect Crash".

---

## ✅ Solução

Modificar `ProxyWindowBase::onScreenDestroyed()` para **migrar janela para primary screen** quando screen atual é destruído.

---

## 📝 Implementação

### Arquivo: `/data/projects/quickshell-patched/src/window/proxywindow.cpp`

**Localização**: Método `onScreenDestroyed()` (aproximadamente linha 400)

**Código Atual**:
```cpp
void ProxyWindowBase::onScreenDestroyed() { 
    this->mScreen = nullptr;
}
```

**Código Proposto**:
```cpp
void ProxyWindowBase::onScreenDestroyed() { 
    this->mScreen = nullptr;
    
    // Migrar janela para primary screen se ainda existe
    if (this->window != nullptr) {
        auto* primaryScreen = QGuiApplication::primaryScreen();
        
        if (primaryScreen != nullptr && primaryScreen != this->window->screen()) {
            qCDebug(logProxyWindow) 
                << "Screen destroyed for window" << this->window
                << "migrating to primary screen" << primaryScreen;
            
            this->window->setScreen(primaryScreen);
            
            // Force update geometry to avoid stale references
            if (this->window->isVisible()) {
                this->window->requestUpdate();
            }
        } else if (primaryScreen == nullptr) {
            // No screens available - hide window to prevent crashes
            qCWarning(logProxyWindow) 
                << "Screen destroyed for window" << this->window
                << "but no valid primary screen available, hiding window";
            
            this->window->hide();
        }
        // else: primaryScreen == current screen (shouldn't happen, but safe)
    }
}
```

**Justificativa das Mudanças**:

1. **`window->setScreen(primaryScreen)`**: Migra QWindow para primary screen válido
   - Qt atualiza todos os handles internos
   - Layer surface pode ser automaticamente recriada no novo output pelo Qt Wayland

2. **`requestUpdate()`**: Force update para garantir que geometria seja recalculada
   - Evita stale references na árvore QML
   - Dispara redraw com novo screen

3. **`hide()` fallback**: Se não há screens disponíveis, esconde janela
   - Previne crash se todos os monitors forem desconectados
   - Window pode ser mostrada novamente quando screen voltar

4. **Logs com `qCDebug`/`qCWarning`**: Para debugging e troubleshooting

---

## 🧪 Validação

### Pré-requisitos
- HDMI conectado ao sistema
- Caelestia Shell rodando com barra visível no HDMI

### Cenários de Teste

| # | Cenário | Ação | Resultado Esperado |
|---|---------|------|-------------------|
| 1 | Desconectar HDMI com bar visível | Remover cabo HDMI | Shell não crasha; barra desaparece ou move para laptop |
| 2 | Reconectar HDMI | Conectar cabo HDMI | Barra reaparece no HDMI (ou laptop, dependendo de config) |
| 3 | Hotplug rápido | Desconectar/reconectar 3x rápido | Sem crashes |
| 4 | Monitor único | Iniciar sem HDMI | Barra no laptop, sem regressões |
| 5 | HDMI já conectado | Iniciar com HDMI | Barras em ambas as telas conforme config |
| 6 | Conectar HDMI depois | Iniciar sem HDMI, conectar depois | Barra aparece no HDMI |

### Comandos de Teste

```bash
# 1. Build e instalar
cd /data/projects/quickshell-patched/build
make -j$(nproc)
sudo make install

# 2. Backup logs anteriores
mkdir -p ~/hdmi-test-logs
cp -r ~/.cache/quickshell/crashes ~/hdmi-test-logs/crashes-before-$(date +%Y%m%d-%H%M%S)

# 3. Reiniciar Caelestia
# Super+Shift+R ou logout/login

# 4. Monitor logs em tempo real
journalctl --user -f | grep -i quickshell | tee ~/hdmi-test-logs/test-$(date +%Y%m%d-%H%M%S).log

# 5. Em outro terminal: verificar outputs
watch -n1 'hyprctl monitors && echo "---" && hyprctl layers'

# 6. Executar cenários de teste (desconectar/reconectar HDMI)

# 7. Verificar crashes
ls -lt ~/.cache/quickshell/crashes/ | head -10

# 8. Se crashar, pegar backtrace
coredumpctl info $(coredumpctl list | grep quickshell | head -1 | awk '{print $5}')
```

### Critérios de Sucesso

✅ **PASS** se:
- Nenhum crash em nenhum dos 6 cenários
- Logs mostram "migrating to primary screen" ao desconectar HDMI
- Barra desaparece ou move para laptop (não fica "fantasma")
- UI continua responsiva após hotplug

❌ **FAIL** se:
- Crash em qualquer cenário
- Barra fica "fantasma" (visível mas não clicável)
- UI congela após hotplug
- Regressões em monitor único

---

## 🔄 Plano B: Destruir/Recriar Layer Surface

Se solução simples (migrar screen) não funcionar, implementar destruição/recriação de layer surface:

### Problema Possível
Qt Wayland pode não gerenciar layer surface corretamente ao trocar screen. O `wl_output` bound na layer surface pode ficar inválido.

### Solução Alternativa

**Arquivo**: `src/wayland/wlr_layershell/surface.cpp`

Adicionar método para destruir/recriar layer surface:

```cpp
void LayerSurfaceBridge::recreateForScreen(QScreen* newScreen) {
    if (!newScreen) return;
    
    // Salvar estado atual
    auto savedState = this->state;
    
    // Destruir surface atual
    if (this->surface) {
        this->surface->destroy();
        delete this->surface;
        this->surface = nullptr;
    }
    
    // Recriar com novo output
    auto* qwindow = this->window;
    qwindow->setScreen(newScreen);
    
    auto* waylandWindow = dynamic_cast<QtWaylandClient::QWaylandWindow*>(qwindow->handle());
    if (!waylandWindow) return;
    
    auto* waylandScreen = dynamic_cast<QtWaylandClient::QWaylandScreen*>(newScreen->handle());
    if (!waylandScreen) return;
    
    // Obter novo output
    wl_output* output = waylandScreen->output();
    
    // Recriar layer surface com novo output
    auto* shell = LayerShellIntegration::instance();
    this->surface = new LayerSurface(shell, waylandWindow, savedState, output);
}
```

**Chamar de** `onScreenDestroyed()`:
```cpp
void ProxyWindowBase::onScreenDestroyed() {
    this->mScreen = nullptr;
    
    if (this->window != nullptr) {
        auto* primaryScreen = QGuiApplication::primaryScreen();
        if (primaryScreen != nullptr) {
            this->window->setScreen(primaryScreen);
            
            // Se layer surface, recriar
            auto* bridge = LayerSurfaceBridge::get(this->window);
            if (bridge) {
                bridge->recreateForScreen(primaryScreen);
            }
        }
    }
}
```

**Complexidade**: 🔴 Alta (mexe em Wayland protocol)  
**Confiança**: 60%  
**Usar**: Apenas se solução simples não funcionar

---

## 📊 Análise de Risco

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Qt não gerencia layer surface ao trocar screen | 30% | 🔴 Alto | Implementar Plano B (recriar surface) |
| Bindings QML ainda referenciam screen destruído | 20% | 🟡 Médio | `requestUpdate()` força refresh; logs ajudam debug |
| Regressão em monitor único | 10% | 🟠 Médio | Testes de validação incluem cenário |
| Crash em edge case (todos monitors desconectados) | 15% | 🔴 Alto | Fallback `hide()` implementado |
| Performance degradation | 5% | 🟢 Baixo | Operação rápida (trocar ponteiro) |

**Risco Geral**: 🟡 Médio (85% confiança de sucesso)

---

## 📚 Referências Técnicas

### Qt Documentation
- [QWindow::setScreen()](https://doc.qt.io/qt-6/qwindow.html#setScreen)
- [QScreen](https://doc.qt.io/qt-6/qscreen.html)
- [QGuiApplication::primaryScreen()](https://doc.qt.io/qt-6/qguiapplication.html#primaryScreen)

### Wayland Protocol
- [wlr-layer-shell-unstable-v1](https://wayland.app/protocols/wlr-layer-shell-unstable-v1)
  - Layer surface bound to specific `wl_output`
  - Output destroy: "Compositors must destroy layer shells when output is destroyed"

### Código Quickshell
- `src/window/proxywindow.cpp`: Gerenciamento de windows/screens
- `src/wayland/wlr_layershell/surface.cpp`: Layer surface Wayland
- `src/window/qmlscreen.hpp`: Screen info para QML (aviso sobre dangling)

### Commits Upstream Relacionados
- `611cd76`: Screen destroy signal fix
- `2032248`: Layer shell bridge UAF fix
- `bcc3d42`: Custom incubation controller

---

## ⏭️ Próximos Passos

### 1. Implementar (30 min)
```bash
cd /data/projects/quickshell-patched
# Editar src/window/proxywindow.cpp conforme código proposto
```

### 2. Build e Instalar (10 min)
```bash
cd build
make -j$(nproc)
sudo make install
```

### 3. Testar (1-2h)
- Executar todos os 6 cenários de teste
- Monitorar logs
- Verificar crashes

### 4. Validar Resultado

**Se PASS**:
- ✅ Marcar implementação como concluída
- ✅ Atualizar `01-quickshell-stability-final.md` com status ✅
- ✅ Documentar em `docs/02-arquitetura.md` (patches ativos)
- ✅ Commitar alteração no quickshell-patched

**Se FAIL**:
- ❌ Analisar logs e backtrace
- ❌ Identificar edge case específico
- ❌ Implementar Plano B se necessário
- ❌ Re-testar

### 5. Após Validação
- Seguir para próxima implementação: Tray Eager Init
- Ver `03-tray-eager-init-implementation.md`

---

**Autor**: Claude (AI Assistant)  
**Data**: 2026-02-03  
**Baseado em**: Análise de código, crash logs, documentação Qt e Wayland  
**Confiança**: ⭐⭐⭐⭐ (85% - solução bem fundamentada, possíveis edge cases)
