# System Tray Eager Init - Plano de Implementação

**Status**: 📋 Pronto para Implementar  
**Prioridade**: 🟠 Média  
**Complexidade**: 🟢 Baixa  
**Tempo Estimado**: 30 minutos  
**Confiança**: 90%

---

## 🎯 Objetivo

Fazer system tray funcionar mesmo com patch de sync incubation ativo, criando `StatusNotifierHost` **antes** de `shell.qml` carregar.

---

## 🔍 Diagnóstico Resumido

**Problema**: `SystemTray.items` fica vazio; ícones não aparecem.

**Causa Raiz**:
1. `StatusNotifierHost` conecta ao D-Bus watcher via `asyncReadProperty()`
2. Callback D-Bus é assíncrono, precisa event loop processar
3. **Sync incubation** em `LazyLoader`/`BoundComponent` bloqueia event loop durante startup
4. Callback D-Bus não processa antes do `Tray` ler o modelo
5. `items.count: 0` e nunca atualiza

**Validado**:
- ✅ Watcher tem itens: `busctl ... RegisteredStatusNotifierItems` retorna Spotify, Steam
- ✅ Problema entre watcher e SystemTray (Host pipeline bloqueado)

**Detalhes**: Ver `01-quickshell-stability-final.md` seção "2. System Tray Quebrado".

---

## ✅ Solução

Criar plugin `QsEnginePlugin` que roda em `constructGeneration()` (antes de shell carregar) e inicializa `StatusNotifierHost` eager.

### Fluxo Esperado

```
EngineGeneration::EngineGeneration()
  ↓
runConstructGeneration()  ← Plugin hooks aqui
  ↓
StatusNotifierPlugin::constructGeneration()
  ↓
StatusNotifierHost::instance()  [EAGER INIT]
  ↓
connectToWatcher(), asyncReadProperty()
  ↓
[EVENT LOOP processa callback D-Bus]
  ↓
Items adicionados ao modelo
  ↓
shell.qml carrega [sync incubation pode bloquear]
  ↓
Tray referencia SystemTray.items
  ↓
Modelo JÁ POPULADO ✅
```

---

## 📝 Implementação

### Arquivo Novo: `/data/projects/quickshell-patched/src/services/status_notifier/init.cpp`

```cpp
#include "../../core/plugin.hpp"
#include "host.hpp"

namespace {

class StatusNotifierPlugin: public QsEnginePlugin {
	void constructGeneration(EngineGeneration& /*unused*/) override {
		// Eager init: cria StatusNotifierHost antes do shell.qml carregar
		// Isso permite que o callback D-Bus seja processado antes do
		// sync incubation bloquear o event loop
		qs::service::sni::StatusNotifierHost::instance();
	}
};

QS_REGISTER_PLUGIN(StatusNotifierPlugin);

} // namespace
```

**Explicação**:

1. **`QsEnginePlugin`**: Interface do quickshell para plugins que rodam durante inicialização da engine
2. **`constructGeneration()`**: Hook chamado ANTES de `shell.qml` ser carregado
3. **`StatusNotifierHost::instance()`**: Singleton; primeira chamada cria e inicializa o Host
4. **Callback D-Bus**: Com Host já existindo, callback pode processar antes de sync incubation
5. **`QS_REGISTER_PLUGIN`**: Registra plugin automaticamente (macro do quickshell)

---

### Arquivo Modificar: `/data/projects/quickshell-patched/src/services/status_notifier/CMakeLists.txt`

**Localização**: Lista de fontes do `qt_add_library`

**Antes** (aproximadamente linha 26):
```cmake
qt_add_library(quickshell-service-statusnotifier STATIC
	qml.cpp

	watcher.cpp
	host.cpp
	item.cpp
	dbus_item_types.cpp
	${DBUS_INTERFACES}
)
```

**Depois**:
```cmake
qt_add_library(quickshell-service-statusnotifier STATIC
	qml.cpp
	init.cpp  # ← Adicionar

	watcher.cpp
	host.cpp
	item.cpp
	dbus_item_types.cpp
	${DBUS_INTERFACES}
)
```

---

## 🧪 Validação

### Pré-requisitos
- Spotify ou Steam instalados e rodando
- Caelestia Shell rodando

### Cenários de Teste

| # | Cenário | Ação | Resultado Esperado |
|---|---------|------|-------------------|
| 1 | Tray com apps abertos | Iniciar shell com Spotify/Steam já abertos | Ícones aparecem no tray |
| 2 | Abrir app depois | Iniciar shell, depois abrir Spotify | Ícone aparece no tray após ~1s |
| 3 | Fechar app | Fechar Spotify | Ícone desaparece do tray |
| 4 | Reabrir app | Reabrir Spotify | Ícone reaparece |
| 5 | Crash Steam (regressão) | Abrir/fechar Steam rapidamente | Shell NÃO crasha (sync incubation ainda ativo) |
| 6 | Crash Thunar (regressão) | Abrir/fechar Thunar 2x rápido | Shell NÃO crasha |

### Comandos de Teste

```bash
# 1. Build e instalar
cd /data/projects/quickshell-patched/build
cmake .. -DCMAKE_BUILD_TYPE=Release  # Se necessário reconfigurar
make -j$(nproc)
sudo make install

# 2. Verificar watcher antes de testar
busctl --user call org.kde.StatusNotifierWatcher /StatusNotifierWatcher \
  org.freedesktop.DBus.Properties Get ss \
  org.kde.StatusNotifierWatcher RegisteredStatusNotifierItems
# Deve retornar lista com Spotify, Steam, etc

# 3. Reiniciar Caelestia
# Super+Shift+R ou logout/login

# 4. Verificar tray visualmente
# Ícones devem aparecer na barra

# 5. Debug (se não funcionar)
# Adicionar log temporário em shell.qml ou SystemTray QML:
# console.log("SystemTray.items.count:", SystemTray.items.count)

# 6. Verificar journalctl para logs do quickshell
journalctl --user -f | grep -i quickshell
```

### Critérios de Sucesso

✅ **PASS** se:
- Ícones aparecem no tray (cenário 1)
- Tray atualiza ao abrir/fechar apps (cenários 2-4)
- Crashes Steam/Thunar continuam evitados (cenários 5-6)
- `SystemTray.items.count > 0` (verificar via debug log)

❌ **FAIL** se:
- Tray continua vazio
- Crashes voltam (Steam/Thunar)
- Build quebra

---

## 🔄 Plano B: AsynchronousIfNested

Se eager init não funcionar (timing ainda insuficiente), considerar reverter sync incubation para `AsynchronousIfNested`:

### Trade-off
- ✅ Tray funciona automaticamente
- ❌ Crashes Steam/Thunar **podem** voltar
- ❓ Qt recomenda `AsynchronousIfNested` (mas temos race validada)

### Implementação Plano B

**Arquivo**: `src/core/lazyloader.cpp` (linha 166)
```cpp
// REVERTER de:
this->incubator = new QsQmlIncubator(QQmlIncubator::Synchronous, this);

// PARA:
this->incubator = new QsQmlIncubator(
    this->targetActive ? QQmlIncubator::Synchronous : QQmlIncubator::AsynchronousIfNested,
    this
);
```

**Arquivo**: `src/core/boundcomponent.cpp` (linha 117)
```cpp
// REVERTER de:
this->incubator = new QsQmlIncubator(QsQmlIncubator::Synchronous, this);

// PARA:
this->incubator = new QsQmlIncubator(QsQmlIncubator::AsynchronousIfNested, this);
```

**Testar**:
1. Verificar tray funciona
2. Testar Steam/Thunar intensivamente (abrir/fechar 10x)
3. Se crashar: voltar para Synchronous + eager init

**Confiança Plano B**: 60% (trade-off estabilidade vs funcionalidade)

---

## 🔄 Plano C: Tray Polling

Se nem eager init nem AsynchronousIfNested funcionarem, implementar polling no Tray QML:

```qml
// Em módulo Tray
Timer {
    interval: 1000
    running: SystemTray.items.count === 0
    repeat: true
    onTriggered: {
        // Force refresh
        SystemTray.host.refresh()  // Se método existir
    }
}
```

**Complexidade**: 🟢 Baixa  
**Confiança**: 70%  
**Trade-off**: +1s latency para ícones aparecerem

---

## 📊 Análise de Risco

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Timing ainda insuficiente (callback não processa a tempo) | 15% | 🟡 Médio | Plano B (AsynchronousIfNested) ou C (polling) |
| Crashes voltam (regressão sync incubation) | 5% | 🔴 Alto | Testes extensivos; reverter se crashar |
| Build quebra | 5% | 🟢 Baixo | CMakeLists.txt simples; verificar syntax |
| Performance degradation | 3% | 🟢 Baixo | Eager init rápido (< 10ms) |

**Risco Geral**: 🟢 Baixo (90% confiança de sucesso)

---

## 📚 Referências Técnicas

### Código Quickshell
- `src/core/plugin.hpp`: Interface `QsEnginePlugin`
- `src/core/generation.cpp`: `runConstructGeneration()` (linha ~90)
- `src/services/status_notifier/host.cpp`: `StatusNotifierHost::instance()`
- `src/services/status_notifier/qml.cpp`: QML module registration

### Exemplos de Plugins Existentes
```bash
# Ver outros plugins no quickshell
grep -r "QS_REGISTER_PLUGIN" /data/projects/quickshell-patched/src/
# Exemplos: src/wayland/init.cpp, src/x11/init.cpp, src/window/init.cpp
```

### D-Bus StatusNotifier Protocol
- [org.kde.StatusNotifierWatcher](https://www.freedesktop.org/wiki/Specifications/StatusNotifierItem/StatusNotifierWatcher/)
- [org.kde.StatusNotifierItem](https://www.freedesktop.org/wiki/Specifications/StatusNotifierItem/)

---

## ⏭️ Próximos Passos

### 1. Criar init.cpp (5 min)
```bash
cd /data/projects/quickshell-patched/src/services/status_notifier
# Criar init.cpp conforme código acima
```

### 2. Modificar CMakeLists.txt (2 min)
```bash
# Editar CMakeLists.txt, adicionar init.cpp
```

### 3. Build e Instalar (10 min)
```bash
cd /data/projects/quickshell-patched/build
make -j$(nproc)
sudo make install
```

### 4. Testar (15 min)
- Executar todos os 6 cenários de teste
- Verificar tray visualmente
- Verificar crashes não voltaram

### 5. Validar Resultado

**Se PASS**:
- ✅ Marcar implementação como concluída
- ✅ Atualizar `01-quickshell-stability-final.md` com status ✅
- ✅ Commitar alteração no quickshell-patched
- ✅ Deletar docs obsoletos (22-tray-*.md)

**Se FAIL** (tray continua vazio):
- ❌ Adicionar debug logs temporários
- ❌ Verificar timing com `journalctl`
- ❌ Implementar Plano B ou C

**Se FAIL** (crashes voltam):
- ❌ REVERTER implementação imediatamente
- ❌ Manter sync incubation sem eager init
- ❌ Documentar em troubleshooting

### 6. Limpeza
```bash
# Após validação, deletar docs obsoletos
cd /data/projects/quickshell-patched/docs/plan-and-implementation
rm -f 22-tray-diagnostic-plan.md 22-tray-diff-analysis.md 22-tray-eager-init.md
```

---

## 📋 Checklist de Implementação

- [ ] Criar `init.cpp` com código proposto
- [ ] Modificar `CMakeLists.txt` para adicionar `init.cpp`
- [ ] Build: `cd build && make -j$(nproc)`
- [ ] Instalar: `sudo make install`
- [ ] Reiniciar shell: Super+Shift+R
- [ ] Verificar ícones aparecem no tray
- [ ] Testar abrir/fechar Spotify
- [ ] Testar Steam rapid open/close (verificar não crasha)
- [ ] Testar Thunar 2x fast close (verificar não crasha)
- [ ] Commit se PASS

---

**Autor**: Claude (AI Assistant)  
**Data**: 2026-02-03  
**Baseado em**: Análise de código, diagnóstico executado, arquitetura quickshell plugin system  
**Confiança**: ⭐⭐⭐⭐⭐ (90% - solução bem fundamentada e simples)
