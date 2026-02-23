# ESPECIFICAÇÃO DE CONFIGURAÇÃO PERSISTENTE

**Versão**: 1.0  
**Data**: 03/02/2026  
**Objetivo**: Padronizar gerenciamento de configurações JSON com validação, versionamento e persistência atômica

---

## 📋 ÍNDICE

1. [Visão Geral](#visão-geral)
2. [Arquitetura do Sistema](#arquitetura-do-sistema)
3. [Schema JSON](#schema-json)
4. [Versionamento e Migração](#versionamento-e-migração)
5. [Validação](#validação)
6. [Persistência Atômica](#persistência-atômica)
7. [Valores Padrão](#valores-padrão)
8. [Boas Práticas](#boas-práticas)
9. [Exemplos Práticos](#exemplos-práticos)
10. [Checklist de Implementação](#checklist-de-implementação)

---

## 🎯 VISÃO GERAL

### Princípios Fundamentais

O sistema de configuração do Contador de Pessoas v4 segue os seguintes princípios:

1. **Configuração como Código**: Schema versionado, validado e rastreável
2. **Persistência Atômica**: Escrita com temp → rename para evitar corrupção
3. **Defaults Robustos**: Sempre funcional mesmo sem arquivo de configuração
4. **Validação Rigorosa**: Schema version + validação de tipos e valores
5. **Thread-Safe**: Leitura concorrente segura, escrita serializada
6. **Backup Automático**: Configurações corrompidas são preservadas
7. **Dot Notation**: Acesso hierárquico simplificado (`rtsp.url`)

### Componentes do Sistema

```
ConfigManager (Singleton)
├── Load: JSON → Dict (com merge de defaults)
├── Validate: Schema version + campos obrigatórios
├── Get/Set: Acesso thread-safe com dot notation
└── Save: Escrita atômica (temp → rename)
```

---

## 🏗️ ARQUITETURA DO SISTEMA

### 1. ConfigManager (src/persist/config_manager.py)

**Responsabilidades**:
- Carregar configuração JSON do disco
- Validar schema_version e campos críticos
- Mesclar com valores padrão para campos ausentes
- Salvar alterações com escrita atômica
- Fornecer interface thread-safe para leitura/escrita

**Características**:
- **Padrão Singleton**: Uma única instância para toda a aplicação
- **Inicialização única**: Chamado UMA VEZ no startup
- **Thread-safe para leitura**: Config é dict imutável após load
- **Escrita serializada**: Sempre via método `save()`

### 2. Localização dos Arquivos

```python
# Windows: %APPDATA%/Espaço CMaker/Contador de Pessoas V4/
CONFIG_DIR = APPDATA_BASE / "config"
CONFIG_FILE = CONFIG_DIR / "app_config.json"

# Exemplo completo:
# C:/Users/Fabio/AppData/Roaming/Espaço CMaker/Contador de Pessoas V4/config/app_config.json
```

### 3. Fonte Única de Verdade (SSOT)

Todos os valores padrão são definidos em **um único lugar**:

```python
# src/shared/constants.py
DEFAULT_RTSP_URL = "rtsp://<usuario>:<senha>@<host>:554/stream"
DEFAULT_FPS_RENDER = 5
DEFAULT_LOG_LEVEL = "INFO"
CONFIG_SCHEMA_VERSION = 1
```

**Regra de Ouro**: Nunca hardcode valores padrão em módulos individuais. Sempre importar de `constants.py`.

---

## 📄 SCHEMA JSON

### Estrutura Completa (v1)

```json
{
  "schema_version": 1,
  "app_version": "4.0.9",
  
  "rtsp": {
    "url": "rtsp://<usuario>:<senha>@<host>:554/stream",
    "timeout_sec": 5,
    "transport": "tcp",
    "history": ["rtsp://...", "rtsp://..."]
  },
  
  "fps": {
    "render_limit": 5,
    "process_limit": 5
  },
  
  "processing": {
    "enable_detection": false,
    "enable_tracking": false,
    "enable_counting": false
  },
  
  "tripwire": {
    "p1": [100, 300],
    "p2": [900, 300],
    "center": [500, 300],
    "normal": [0, 1],
    "arrow_len_px": 30
  },
  
  "counting": {
    "mode": "ambos",
    "dead_zone_px": 5,
    "cooldown_sec": 2.0,
    "track_timeout_frames": 10,
    "max_displacement_px": 100
  },
  
  "yolo": {
    "model": "yolov8n.pt",
    "conf_threshold": 0.5,
    "imgsz": 416
  },
  
  "detector": {
    "confidence_threshold": 0.5,
    "class_names": ["Pessoa"]
  },
  
  "counter": {
    "mode": "AMBOS"
  },
  
  "audio": {
    "enabled": true,
    "mp3_path": "",
    "volume": 0.8,
    "coalescence_ms": 300,
    "fallback_beep": false
  },
  
  "logging": {
    "level": "INFO",
    "file_enabled": true,
    "console_enabled": false,
    "max_file_size_mb": 5,
    "max_files": 10,
    "ui_buffer_size": 5000
  },
  
  "reports": {
    "retention_months": 2
  },
  
  "ping": {
    "history_hours": 24
  }
}
```

### Campos Obrigatórios

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `schema_version` | int | Versão do schema (para migração) |
| `app_version` | string | Versão da aplicação que criou |
| `rtsp.url` | string | URL do stream RTSP |
| `fps.render_limit` | int | FPS de renderização (1-30) |
| `fps.process_limit` | int | FPS de processamento (1-30) |
| `logging.level` | string | Nível de log (DEBUG/INFO/WARNING/ERROR) |

### Campos Opcionais (com defaults)

Todos os outros campos são opcionais e preenchidos com valores padrão se ausentes.

---

## 🔄 VERSIONAMENTO E MIGRAÇÃO

### Schema Version

```python
# src/shared/constants.py
CONFIG_SCHEMA_VERSION = 1  # Incrementar quando schema mudar
```

### Estratégia de Migração

#### Cenário 1: Schema Compatível (Minor Changes)
**Exemplo**: Adicionar novo campo opcional

```python
# Novo campo em constants.py
DEFAULT_NEW_FEATURE_ENABLED = False

# Adicionar em _get_defaults()
"new_feature": {
    "enabled": DEFAULT_NEW_FEATURE_ENABLED
}

# Não incrementar CONFIG_SCHEMA_VERSION
# Merge automático com defaults funciona
```

#### Cenário 2: Breaking Change (Major Changes)
**Exemplo**: Renomear campo ou mudar tipo

```python
# Incrementar schema version
CONFIG_SCHEMA_VERSION = 2

# Adicionar migração em ConfigManager.load()
if loaded.get('schema_version') == 1:
    # Migrar de v1 para v2
    loaded = self._migrate_v1_to_v2(loaded)
    loaded['schema_version'] = 2
    self.logger.info("Migrated config from v1 to v2")

def _migrate_v1_to_v2(self, config: Dict) -> Dict:
    """Migrar configuração v1 → v2"""
    # Exemplo: renomear campo
    if 'old_field' in config:
        config['new_field'] = config.pop('old_field')
    
    # Exemplo: mudar tipo
    if isinstance(config.get('fps'), int):
        config['fps'] = {'render_limit': config['fps'], 'process_limit': config['fps']}
    
    return config
```

### Compatibilidade

| Tipo de Mudança | Schema Version | Migração | Exemplo |
|-----------------|----------------|----------|---------|
| Adicionar campo opcional | Manter | Automática (merge) | Novo recurso com default |
| Remover campo | Manter | Automática (ignorado) | Deprecação silenciosa |
| Renomear campo | **Incrementar** | Manual | `old_name` → `new_name` |
| Mudar tipo | **Incrementar** | Manual | `int` → `dict` |
| Mudar valor padrão | Manter | Automática | Ajuste de performance |

---

## ✅ VALIDAÇÃO

### Níveis de Validação

#### 1. Validação de Schema (Estrutural)

```python
def _validate_schema(self, config: Dict) -> List[str]:
    """Valida estrutura básica do schema"""
    errors = []
    
    # Schema version
    if config.get('schema_version') != CONFIG_SCHEMA_VERSION:
        errors.append(
            f"Invalid schema_version: {config.get('schema_version')}, "
            f"expected {CONFIG_SCHEMA_VERSION}"
        )
    
    # Campos obrigatórios
    required_fields = [
        'rtsp.url',
        'fps.render_limit',
        'fps.process_limit',
        'logging.level'
    ]
    
    for field in required_fields:
        if self._get_nested(config, field) is None:
            errors.append(f"Missing required field: {field}")
    
    return errors
```

#### 2. Validação de Tipos

```python
def _validate_types(self, config: Dict) -> List[str]:
    """Valida tipos de dados"""
    errors = []
    
    type_checks = {
        'rtsp.url': str,
        'rtsp.timeout_sec': (int, float),
        'fps.render_limit': int,
        'audio.enabled': bool,
        'logging.level': str,
        'tripwire.p1': list,
    }
    
    for path, expected_type in type_checks.items():
        value = self._get_nested(config, path)
        if value is not None and not isinstance(value, expected_type):
            errors.append(
                f"Invalid type for {path}: expected {expected_type}, "
                f"got {type(value)}"
            )
    
    return errors
```

#### 3. Validação de Valores (Range/Enum)

```python
def _validate_values(self, config: Dict) -> List[str]:
    """Valida ranges e enumerações"""
    errors = []
    
    # FPS deve estar entre 1 e 30
    fps_render = config.get('fps', {}).get('render_limit')
    if fps_render and not (1 <= fps_render <= 30):
        errors.append(f"fps.render_limit must be 1-30, got {fps_render}")
    
    # Log level deve ser válido
    log_level = config.get('logging', {}).get('level', '').upper()
    valid_levels = ['DEBUG', 'INFO', 'WARNING', 'ERROR', 'CRITICAL']
    if log_level not in valid_levels:
        errors.append(f"logging.level must be one of {valid_levels}, got {log_level}")
    
    # Volume deve estar entre 0.0 e 1.0
    volume = config.get('audio', {}).get('volume')
    if volume is not None and not (0.0 <= volume <= 1.0):
        errors.append(f"audio.volume must be 0.0-1.0, got {volume}")
    
    # Tripwire deve ter 2 pontos com coordenadas válidas
    p1 = config.get('tripwire', {}).get('p1')
    if p1 and (not isinstance(p1, list) or len(p1) != 2):
        errors.append(f"tripwire.p1 must be [x, y], got {p1}")
    
    return errors
```

### Estratégia de Tratamento de Erros

```python
def load(self) -> Dict[str, Any]:
    """Carregar com validação progressiva"""
    
    # 1. Carregar JSON
    try:
        loaded = json.load(f)
    except json.JSONDecodeError as e:
        self.logger.error(f"Invalid JSON: {e}")
        self._backup_corrupted_config()
        return self._get_defaults()
    
    # 2. Validar schema version
    if loaded.get('schema_version') != CONFIG_SCHEMA_VERSION:
        self.logger.warning("Schema version mismatch, using defaults")
        self._backup_corrupted_config()
        return self._get_defaults()
    
    # 3. Mesclar com defaults (preenche campos faltantes)
    config = self._merge_with_defaults(loaded)
    
    # 4. Validar tipos e valores (opcional, apenas warning)
    errors = self._validate_types(config) + self._validate_values(config)
    if errors:
        for error in errors:
            self.logger.warning(f"Config validation issue: {error}")
        # Continuar mesmo com erros (valores padrão já preenchidos)
    
    return config
```

**Filosofia**: Ser tolerante a erros não críticos, mas proteger contra corrupção total.

---

## 💾 PERSISTÊNCIA ATÔMICA

### Problema: Corrupção Durante Escrita

```python
# ❌ PERIGOSO: Escrita direta
with open(config_file, 'w') as f:
    json.dump(config, f)
    # Se o processo crashar aqui, arquivo fica corrompido
```

### Solução: Write-to-Temp + Atomic Rename

```python
# ✅ SEGURO: Escrita atômica
def save(self) -> bool:
    """Salva configuração com escrita atômica"""
    try:
        # 1. Criar diretório se não existir
        CONFIG_DIR.mkdir(parents=True, exist_ok=True)
        
        # 2. Escrever em arquivo temporário
        temp_path = self._config_path.with_suffix('.tmp')
        with open(temp_path, 'w', encoding='utf-8') as f:
            json.dump(self._config, f, indent=2, ensure_ascii=False)
        
        # 3. Renomear (operação atômica no Windows para mesmo disco)
        temp_path.replace(self._config_path)
        
        self.logger.info(f"Configuration saved successfully")
        return True
        
    except Exception as e:
        self.logger.error(f"Failed to save config: {e}", exc_info=True)
        
        # Limpar arquivo temporário se existir
        if temp_path.exists():
            temp_path.unlink()
        
        return False
```

**Garantias**:
- Se falhar durante escrita no `.tmp`, arquivo original permanece intacto
- `replace()` é atômico: ou sucede completamente ou falha sem efeito
- Nunca haverá arquivo parcialmente escrito

### Backup de Configurações Corrompidas

```python
def _backup_corrupted_config(self) -> None:
    """Preserva arquivo corrompido para análise"""
    if self._config_path.exists():
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        backup_path = self._config_path.with_suffix(f'.backup.{timestamp}')
        
        try:
            shutil.copy2(self._config_path, backup_path)
            self.logger.warning(f"Corrupted config backed up to {backup_path}")
        except Exception as e:
            self.logger.error(f"Failed to backup: {e}")
```

**Resultado**:
```
config/
├── app_config.json           # Atual (funcional)
├── app_config.backup.20260203_143052  # Backup automático
└── app_config.backup.20260201_091234  # Backup anterior
```

---

## 🔧 VALORES PADRÃO

### Estratégia de Defaults

#### 1. Definir em constants.py

```python
# src/shared/constants.py
DEFAULT_RTSP_URL = "rtsp://<usuario>:<senha>@<host>:554/stream"
DEFAULT_FPS_RENDER = 5
DEFAULT_LOG_LEVEL = "INFO"
DEFAULT_AUDIO_VOLUME = 0.8
```

#### 2. Construir dict completo

```python
def _get_defaults(self) -> Dict[str, Any]:
    """Retorna configuração padrão completa"""
    return {
        "schema_version": CONFIG_SCHEMA_VERSION,
        "app_version": APP_VERSION,
        
        "rtsp": {
            "url": DEFAULT_RTSP_URL,
            "timeout_sec": DEFAULT_RTSP_TIMEOUT_SEC,
            "transport": DEFAULT_RTSP_TRANSPORT,
            "history": []
        },
        
        "fps": {
            "render_limit": DEFAULT_FPS_RENDER,
            "process_limit": DEFAULT_FPS_PROCESS
        },
        
        # ... todos os outros campos
    }
```

#### 3. Merge Recursivo

```python
def _merge_with_defaults(self, loaded: Dict) -> Dict:
    """Mesclar config carregado com defaults"""
    defaults = self._get_defaults()
    
    def merge(base: Dict, updates: Dict) -> Dict:
        result = base.copy()
        for key, value in updates.items():
            if key in result and isinstance(result[key], dict) and isinstance(value, dict):
                # Merge recursivo para dicts aninhados
                result[key] = merge(result[key], value)
            else:
                # Sobrescrever valor
                result[key] = value
        return result
    
    return merge(defaults, loaded)
```

**Exemplo de Merge**:

```python
# Defaults:
{"fps": {"render_limit": 5, "process_limit": 5}}

# Loaded (incompleto):
{"fps": {"render_limit": 10}}

# Após merge:
{"fps": {"render_limit": 10, "process_limit": 5}}
#                ^^ do loaded    ^^ do defaults
```

---

## ✅ BOAS PRÁTICAS

### 1. Acesso à Configuração

```python
# ✅ Usar dot notation
rtsp_url = config_manager.get('rtsp.url')
fps_render = config_manager.get('fps.render_limit', 5)  # Com default

# ❌ Evitar acesso direto ao dict interno
rtsp_url = config_manager._config['rtsp']['url']  # Quebrará se estrutura mudar
```

### 2. Modificação de Configuração

```python
# ✅ Usar set() + save()
config_manager.set('rtsp.url', new_url)
config_manager.set('fps.render_limit', 10)
config_manager.save()

# ⚠️ Batch updates
config_manager.set('rtsp.url', url1)
config_manager.set('rtsp.timeout_sec', 10)
config_manager.set('fps.render_limit', 15)
config_manager.save()  # Salvar UMA VEZ ao final

# ❌ Modificação direta
config_manager._config['rtsp']['url'] = new_url  # NÃO faz save()
```

### 3. Validação de Entrada do Usuário

```python
# ✅ Validar antes de setar
def update_fps_from_ui(fps_value: str):
    try:
        fps = int(fps_value)
        if not (1 <= fps <= 30):
            show_error("FPS deve estar entre 1 e 30")
            return
        
        config_manager.set('fps.render_limit', fps)
        config_manager.save()
        logger.info(f"FPS updated to {fps}")
        
    except ValueError:
        show_error(f"Valor inválido: {fps_value}")
```

### 4. Hot Reload vs Reinício Obrigatório

Algumas configurações requerem reinício da aplicação:

```python
# Requer REINÍCIO (afeta inicialização):
- logging.level
- yolo.model
- reports.retention_months

# Hot reload OK (runtime):
- fps.render_limit
- fps.process_limit
- audio.volume
- audio.enabled
- tripwire.*
```

**Implementação**:

```python
def apply_config_changes(self, key: str, value: Any):
    """Aplica mudanças de configuração em runtime"""
    
    if key.startswith('fps.'):
        # Hot reload: atualizar limitador FPS
        self.frame_limiter.update_fps(value)
        logger.info(f"FPS updated to {value} (hot reload)")
    
    elif key.startswith('audio.'):
        # Hot reload: atualizar volume
        self.audio_service.set_volume(value)
    
    elif key.startswith('logging.'):
        # Requer reinício
        show_warning("Mudanças de logging requerem reiniciar a aplicação")
    
    else:
        logger.debug(f"Config changed: {key} = {value}")
```

### 5. Histórico de RTSP

```python
# Adicionar URL ao histórico apenas após sucesso
def on_rtsp_connected(url: str):
    config_manager.add_rtsp_to_history(url)
    # Limita a 10 URLs, mais recentes primeiro

# Exibir histórico na UI
history = config_manager.get('rtsp.history', [])
for url in history:
    combo_box.addItem(url)
```

### 6. Thread Safety

```python
# ✅ Leitura: Thread-safe (dict é imutável após load)
def worker_thread():
    url = config_manager.get('rtsp.url')  # Seguro
    process_stream(url)

# ✅ Escrita: Sempre na thread principal (GUI)
def on_button_click():
    config_manager.set('fps.render_limit', 10)
    config_manager.save()  # OK na thread GUI

# ❌ Escrita concorrente: Evitar
def thread_a():
    config_manager.set('key1', 'value1')
    config_manager.save()  # Pode colidir com thread_b

def thread_b():
    config_manager.set('key2', 'value2')
    config_manager.save()  # Pode colidir com thread_a
```

**Solução**: Usar fila ou serializar escritas:

```python
class ConfigManager:
    def __init__(self):
        self._write_lock = threading.Lock()
    
    def save(self) -> bool:
        with self._write_lock:
            # Escrita serializada
            return self._do_save()
```

### 7. Logging de Mudanças

```python
# ✅ Logar mudanças importantes (auditoria)
def set(self, key: str, value: Any) -> None:
    old_value = self.get(key)
    
    # Atualizar valor
    self._set_nested(key, value)
    
    # Log de auditoria
    if old_value != value:
        self.logger.info(f"Config changed: {key} = {old_value} → {value}")
    else:
        self.logger.debug(f"Config set: {key} = {value} (unchanged)")
```

### 8. Configuração de Exemplo

```python
# Fornecer arquivo de exemplo separado
# config/app_config.json.example

{
  "schema_version": 1,
  "rtsp": {
    "url": "rtsp://<usuario>:<senha>@<host>:554/stream",
    "timeout_sec": 5
  },
  "_comment": "Copy this file to app_config.json and edit values"
}
```

### 9. Documentação Inline

```json
{
  "fps": {
    "render_limit": 5,
    "_comment_render": "FPS for UI rendering (1-30, lower = less CPU)",
    "process_limit": 5,
    "_comment_process": "FPS for YOLO detection (1-30, lower = less GPU)"
  }
}
```

---

## 💡 EXEMPLOS PRÁTICOS

### Exemplo 1: Inicialização no Startup

```python
"""
src/Contador-Pessoas.py
Entry point da aplicação
"""
import logging
from persist import ConfigManager

def main():
    # 1. Setup básico
    setup_directories()
    logger = setup_basic_logging()
    
    # 2. Carregar configuração
    config_manager = ConfigManager()
    config = config_manager.load()
    
    logger.info("=" * 80)
    logger.info(f"Configuration loaded from: {config_manager._config_path}")
    logger.info(f"Schema version: {config.get('schema_version')}")
    logger.info(f"App version: {config.get('app_version')}")
    logger.info("=" * 80)
    
    # 3. Usar configuração
    rtsp_url = config_manager.get('rtsp.url')
    log_level = config_manager.get('logging.level', 'INFO')
    
    # 4. Configurar LoggerService com nível do config
    logger_service = LoggerService.setup(log_level=log_level)
    
    # 5. Inicializar serviços
    rtsp_service = RTSPService(url=rtsp_url)
    # ...
```

### Exemplo 2: Tab de Configuração (GUI)

```python
"""
src/gui/config_tab.py
Interface de configuração
"""
from PySide6.QtWidgets import QWidget, QLineEdit, QSpinBox, QComboBox
from persist import ConfigManager

class ConfigTab(QWidget):
    def __init__(self, config_manager: ConfigManager):
        super().__init__()
        self.config_manager = config_manager
        self.logger = logging.getLogger("gui.config")
        
        self._setup_ui()
        self._load_config_to_ui()
    
    def _load_config_to_ui(self):
        """Carrega valores de configuração para widgets"""
        # RTSP
        self.rtsp_url_edit.setText(
            self.config_manager.get('rtsp.url', '')
        )
        
        # FPS
        self.fps_render_spin.setValue(
            self.config_manager.get('fps.render_limit', 5)
        )
        self.fps_process_spin.setValue(
            self.config_manager.get('fps.process_limit', 5)
        )
        
        # Log Level
        log_level = self.config_manager.get('logging.level', 'INFO')
        index = self.log_level_combo.findText(log_level)
        if index >= 0:
            self.log_level_combo.setCurrentIndex(index)
        
        # Histórico RTSP
        history = self.config_manager.get('rtsp.history', [])
        for url in history:
            self.rtsp_url_edit.addItem(url)
    
    def on_apply_button_clicked(self):
        """Aplicar mudanças de configuração"""
        try:
            # Validar entrada
            rtsp_url = self.rtsp_url_edit.text().strip()
            if not rtsp_url.startswith('rtsp://'):
                self._show_error("URL RTSP inválida (deve começar com rtsp://)")
                return
            
            fps_render = self.fps_render_spin.value()
            fps_process = self.fps_process_spin.value()
            
            # Atualizar configuração
            self.config_manager.set('rtsp.url', rtsp_url)
            self.config_manager.set('fps.render_limit', fps_render)
            self.config_manager.set('fps.process_limit', fps_process)
            self.config_manager.set('logging.level', self.log_level_combo.currentText())
            
            # Salvar
            if self.config_manager.save():
                self.logger.info("Configuration saved successfully")
                self._show_success("Configuração salva com sucesso!")
                
                # Emitir sinal para atualizar serviços
                self.config_changed.emit()
            else:
                self._show_error("Falha ao salvar configuração")
        
        except Exception as e:
            self.logger.error(f"Failed to apply config: {e}", exc_info=True)
            self._show_error(f"Erro: {e}")
    
    def on_reset_button_clicked(self):
        """Resetar para valores padrão"""
        reply = QMessageBox.question(
            self, 
            "Confirmar Reset",
            "Resetar todas as configurações para valores padrão?",
            QMessageBox.Yes | QMessageBox.No
        )
        
        if reply == QMessageBox.Yes:
            self.config_manager._config = self.config_manager._get_defaults()
            self.config_manager.save()
            self._load_config_to_ui()
            self.logger.info("Configuration reset to defaults")
```

### Exemplo 3: Hot Reload de FPS

```python
"""
src/vision_pipeline.py
Pipeline com hot reload de FPS
"""
class VisionPipeline:
    def __init__(self, config_manager: ConfigManager):
        self.config_manager = config_manager
        self.logger = logging.getLogger("vision_pipeline")
        
        # Carregar FPS inicial
        self.fps_render = config_manager.get('fps.render_limit', 5)
        self.fps_process = config_manager.get('fps.process_limit', 5)
        
        self._last_render_time = 0
        self._last_process_time = 0
    
    def update_fps_limits(self):
        """Hot reload de FPS (chamado quando config muda)"""
        new_render = self.config_manager.get('fps.render_limit', 5)
        new_process = self.config_manager.get('fps.process_limit', 5)
        
        if new_render != self.fps_render:
            self.logger.info(f"FPS render updated: {self.fps_render} → {new_render}")
            self.fps_render = new_render
        
        if new_process != self.fps_process:
            self.logger.info(f"FPS process updated: {self.fps_process} → {new_process}")
            self.fps_process = new_process
    
    def process_frame(self, frame):
        """Processar frame com throttling de FPS"""
        current_time = time.time()
        
        # Throttling de renderização
        render_interval = 1.0 / self.fps_render
        if current_time - self._last_render_time < render_interval:
            return None  # Pular frame
        
        self._last_render_time = current_time
        
        # Throttling de processamento
        process_interval = 1.0 / self.fps_process
        should_process = (current_time - self._last_process_time) >= process_interval
        
        if should_process:
            # Detecção + Tracking + Contagem
            detections = self.detector.detect(frame)
            tracks = self.tracker.update(detections)
            events = self.counter.process(tracks)
            
            self._last_process_time = current_time
        
        return frame
```

### Exemplo 4: Migração de Schema v1 → v2

```python
"""
Exemplo de migração quando schema muda
"""
# constants.py (atualizado)
CONFIG_SCHEMA_VERSION = 2  # Incrementado de 1 para 2

# config_manager.py
def load(self) -> Dict[str, Any]:
    """Carregar com migração automática"""
    loaded = json.load(f)
    
    # Verificar se precisa migrar
    schema_version = loaded.get('schema_version', 0)
    
    if schema_version == 1:
        # Migrar v1 → v2
        self.logger.info("Migrating config from v1 to v2")
        loaded = self._migrate_v1_to_v2(loaded)
        loaded['schema_version'] = 2
        
        # Salvar versão migrada
        self._config = loaded
        self.save()
    
    elif schema_version != CONFIG_SCHEMA_VERSION:
        # Schema desconhecido
        self.logger.error(f"Unknown schema version: {schema_version}")
        self._backup_corrupted_config()
        return self._get_defaults()
    
    return self._merge_with_defaults(loaded)

def _migrate_v1_to_v2(self, config: Dict) -> Dict:
    """
    Migração v1 → v2:
    - Mudança: 'fps' era int, agora é dict com render_limit e process_limit
    """
    if 'fps' in config and isinstance(config['fps'], int):
        old_fps = config['fps']
        config['fps'] = {
            'render_limit': old_fps,
            'process_limit': old_fps
        }
        self.logger.info(f"Migrated fps: {old_fps} → {config['fps']}")
    
    return config
```

### Exemplo 5: Validação Customizada

```python
"""
Validação específica de negócio
"""
def _validate_business_rules(self, config: Dict) -> List[str]:
    """Validações de lógica de negócio"""
    errors = []
    
    # Regra: FPS de processamento não pode ser maior que renderização
    fps_render = config.get('fps', {}).get('render_limit', 5)
    fps_process = config.get('fps', {}).get('process_limit', 5)
    
    if fps_process > fps_render:
        errors.append(
            f"fps.process_limit ({fps_process}) cannot exceed "
            f"fps.render_limit ({fps_render})"
        )
    
    # Regra: Tripwire deve ter comprimento mínimo
    p1 = config.get('tripwire', {}).get('p1', [0, 0])
    p2 = config.get('tripwire', {}).get('p2', [0, 0])
    
    distance = ((p2[0] - p1[0])**2 + (p2[1] - p1[1])**2)**0.5
    min_length = config.get('tripwire', {}).get('min_length_px', 50)
    
    if distance < min_length:
        errors.append(
            f"Tripwire length ({distance:.1f}px) is below minimum ({min_length}px)"
        )
    
    # Regra: URL RTSP deve conter credenciais ou ser localhost
    rtsp_url = config.get('rtsp', {}).get('url', '')
    if rtsp_url and 'localhost' not in rtsp_url and '@' not in rtsp_url:
        errors.append(
            "RTSP URL must contain credentials (user:pass@host) or be localhost"
        )
    
    return errors
```

---

## ✔️ CHECKLIST DE IMPLEMENTAÇÃO

Use este checklist ao criar ou revisar sistema de configuração:

### Setup Inicial
- [ ] ConfigManager implementado como Singleton
- [ ] Arquivo de configuração em `APPDATA_BASE/config/app_config.json`
- [ ] Todos os valores padrão definidos em `constants.py`
- [ ] Arquivo `.example` fornecido com comentários

### Schema
- [ ] `schema_version` presente e versionado
- [ ] `app_version` gravado na configuração
- [ ] Campos obrigatórios identificados
- [ ] Campos opcionais têm defaults

### Validação
- [ ] Validação de schema_version
- [ ] Validação de tipos de dados
- [ ] Validação de ranges e enums
- [ ] Validação de regras de negócio
- [ ] Erros de validação loggados

### Persistência
- [ ] Escrita atômica com temp → rename
- [ ] Backup automático de arquivos corrompidos
- [ ] Encoding UTF-8 explícito
- [ ] Indentação JSON (indent=2)
- [ ] ensure_ascii=False para caracteres especiais

### Migração
- [ ] Estratégia de migração documentada
- [ ] Função de migração por versão
- [ ] Backup criado antes de migrar
- [ ] Migração loggada
- [ ] Testes de migração implementados

### Acesso
- [ ] Método get() com dot notation
- [ ] Método set() com dot notation
- [ ] Valores padrão em get()
- [ ] Método save() explícito

### Thread Safety
- [ ] Leitura thread-safe
- [ ] Escrita serializada (lock ou thread única)
- [ ] Documentação de concorrência

### Hot Reload
- [ ] Campos hot-reload identificados
- [ ] Campos que requerem reinício documentados
- [ ] Sinais/callbacks para notificar mudanças
- [ ] UI atualizada após save()

### Logging
- [ ] Load loggado (sucesso e falhas)
- [ ] Save loggado
- [ ] Mudanças de valores loggadas
- [ ] Validação loggada
- [ ] Migrações loggadas

### Testes
- [ ] Teste: load com arquivo ausente
- [ ] Teste: load com JSON inválido
- [ ] Teste: load com schema version errado
- [ ] Teste: merge com defaults
- [ ] Teste: save atômico
- [ ] Teste: validação de tipos
- [ ] Teste: validação de valores
- [ ] Teste: migração entre versões
- [ ] Teste: concorrência (threads)

---

## 📚 REFERÊNCIAS

### Documentação Python
- [json module](https://docs.python.org/3/library/json.html)
- [pathlib](https://docs.python.org/3/library/pathlib.html)
- [threading](https://docs.python.org/3/library/threading.html)

### Padrões
- **Singleton Pattern**: Uma única instância de ConfigManager
- **Atomic Write**: Temp file + rename para evitar corrupção
- **Merge Strategy**: Combinar defaults com valores carregados
- **Schema Versioning**: Versionamento explícito para migração

### Arquivos do Projeto
- [src/persist/config_manager.py](src/persist/config_manager.py) - Implementação
- [src/shared/constants.py](src/shared/constants.py) - Valores padrão
- [config/app_config.json.example](config/app_config.json.example) - Exemplo

### Convenções
- **12-Factor App**: [Config](https://12factor.net/config)
- **Semantic Versioning**: [SemVer](https://semver.org/)

---

## 📝 HISTÓRICO DE VERSÕES

| Versão | Data | Mudanças |
|--------|------|----------|
| 1.0 | 2026-02-03 | Versão inicial da especificação |

---

## 🤝 CONTRIBUINDO

Para sugestões de melhorias nesta especificação, considere:

1. **Novos Padrões**: Validações específicas de domínio
2. **Exemplos Práticos**: Casos de uso reais do projeto
3. **Otimizações**: Técnicas para reduzir I/O
4. **Ferramentas**: Scripts para validação/migração de configs

---

**Última atualização**: 03/02/2026  
**Mantido por**: LLM (Claude Sonnet 4.5)  
**Projeto**: Contador de Pessoas v4.0.9
