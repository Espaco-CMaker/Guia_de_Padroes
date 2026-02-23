# ESPECIFICAÇÃO DE LOGS - Contador de Pessoas v4

**Versão**: 1.0  
**Data**: 03/02/2026  
**Objetivo**: Padronizar a criação de logs para manutenção por LLMs futuras

---

## 📋 ÍNDICE

1. [Visão Geral](#visão-geral)
2. [Arquitetura do Sistema de Logging](#arquitetura-do-sistema-de-logging)
3. [Níveis de Log](#níveis-de-log)
4. [Padrões de Mensagens](#padrões-de-mensagens)
5. [Boas Práticas](#boas-práticas)
6. [Configuração e Constantes](#configuração-e-constantes)
7. [Exemplos Práticos](#exemplos-práticos)
8. [Checklist de Implementação](#checklist-de-implementação)

---

## 🎯 VISÃO GERAL

### Princípios Fundamentais

O sistema de logging do Contador de Pessoas v4 segue os seguintes princípios:

1. **Centralização**: Um único serviço (`LoggerService`) gerencia todo o logging
2. **Múltiplos Destinos**: Logs são enviados para arquivo, UI buffer e console
3. **Rotação Automática**: Arquivos de log rotacionam automaticamente ao atingir limite
4. **Níveis Configuráveis**: Cada destino pode ter seu próprio nível de logging
5. **Thread-Safe**: Sistema funciona corretamente em ambiente multi-thread
6. **Performance**: Buffer circular otimizado para UI sem impacto na performance

### Componentes do Sistema

```
LoggerService (Singleton)
├── RotatingFileHandler → LOGS_DIR/app.log
├── UILogHandler → Circular Buffer (5000 registros)
└── StreamHandler → Console (opcional)
```

---

## 🏗️ ARQUITETURA DO SISTEMA DE LOGGING

### 1. LoggerService (src/logger_service/logger_service.py)

**Responsabilidades**:
- Inicializar logging nativo Python com múltiplos handlers
- Gerenciar buffer circular para UI
- Rotação de arquivos de log
- Fornecer interface para consulta de logs

**Características**:
- **Padrão Singleton**: Uma única instância para toda a aplicação
- **Inicialização única**: Chamado UMA VEZ no startup (`Contador-Pessoas.py`)
- **Thread-safe**: Usa `logging` nativo do Python

### 2. Handlers Configurados

#### A. RotatingFileHandler
```python
# Configuração
Path: LOGS_DIR / "app.log"
MaxBytes: DEFAULT_LOG_MAX_FILE_SIZE_MB * 1024 * 1024  # 5MB padrão
BackupCount: DEFAULT_LOG_MAX_FILES  # 10 arquivos padrão
Encoding: utf-8
Level: Configurável (DEFAULT_LOG_LEVEL)
```

**Rotação**:
- Automática ao atingir `maxBytes`
- Cria backups: `app.log.1`, `app.log.2`, ..., `app.log.10`
- Arquivo mais antigo é deletado quando excede `backupCount`

#### B. UILogHandler
```python
# Configuração
Buffer: deque(maxlen=5000)  # DEFAULT_LOG_UI_BUFFER_SIZE
Level: logging.DEBUG  # Captura tudo
Format: Dict com timestamp, level, logger, message
```

**Características**:
- Buffer circular (FIFO) - logs antigos são descartados automaticamente
- Otimizado para leitura rápida pela UI
- Armazena objetos dict, não strings formatadas

#### C. StreamHandler (Console)
```python
# Configuração
Destino: sys.stderr
Level: Configurável (DEFAULT_LOG_LEVEL)
Opcional: Desabilitado em produção se necessário
```

### 3. Formato das Mensagens

```python
# Formato padrão para arquivo e console
"%(asctime)s | %(levelname)-7s | %(name)-15s | %(message)s"
# Exemplo:
# 2026-02-03 14:30:45 | INFO    | gui.camera      | RTSP stream started

# Formato datetime
"%Y-%m-%d %H:%M:%S"
```

**Componentes**:
- `%(asctime)s`: Timestamp completo
- `%(levelname)-7s`: Nível (padding de 7 caracteres à direita)
- `%(name)-15s`: Nome do logger (padding de 15 caracteres à direita)
- `%(message)s`: Mensagem do log

---

## 📊 NÍVEIS DE LOG

### Hierarquia (do menos ao mais crítico)

| Nível | Valor | Uso | Exemplo |
|-------|-------|-----|---------|
| **DEBUG** | 10 | Informações detalhadas para diagnóstico | `Frame 150: 3 detections, 2 tracks active` |
| **INFO** | 20 | Confirmação de operações normais | `RTSP stream started successfully` |
| **WARNING** | 30 | Situação inesperada mas controlada | `Failed to connect, retrying (3/5)...` |
| **ERROR** | 40 | Erro que impede uma operação específica | `Failed to load model: file not found` |
| **CRITICAL** | 50 | Erro grave que pode encerrar o aplicativo | `Database corruption detected, shutting down` |

### Quando Usar Cada Nível

#### DEBUG
**Usar para**:
- Informações detalhadas de fluxo interno
- Valores de variáveis em pontos críticos
- Contadores e métricas frame-a-frame
- Diagnóstico de problemas complexos

**Não usar para**:
- Informações que sempre devem ser visíveis
- Dados sensíveis (senhas, tokens)

```python
# ✅ Bom uso
self.logger.debug(f"Frame {frame_count}: {len(detections)} detections, {len(tracks)} tracks")
self.logger.debug(f"Processing latency: {elapsed_ms:.2f}ms")

# ❌ Evitar
self.logger.debug("Application started")  # Use INFO
```

#### INFO
**Usar para**:
- Início/fim de operações importantes
- Mudanças de estado do sistema
- Confirmações de sucesso
- Eventos do ciclo de vida da aplicação

```python
# ✅ Bom uso
self.logger.info("VisionPipeline initialized successfully")
self.logger.info(f"RTSP stream connected: {url}")
self.logger.info("Tripwire updated in pipeline")
self.logger.info("Configuration saved to disk")

# ❌ Evitar
self.logger.info(f"Variable x = {x}")  # Use DEBUG
self.logger.info("Error occurred")  # Use ERROR
```

#### WARNING
**Usar para**:
- Situações anormais mas recuperáveis
- Fallbacks ativados
- Configurações subótimas detectadas
- Recursos indisponíveis mas não críticos

```python
# ✅ Bom uso
self.logger.warning("Failed to initialize AudioService, continuing without audio")
self.logger.warning(f"Connection timeout, retrying ({retry_count}/{max_retries})...")
self.logger.warning("Using default configuration (config file not found)")
self.logger.warning("Frame dropped due to processing delay")

# ❌ Evitar
self.logger.warning("Application started")  # Use INFO
self.logger.warning("Fatal error")  # Use ERROR ou CRITICAL
```

#### ERROR
**Usar para**:
- Falhas em operações específicas
- Exceções capturadas que impedem funcionalidade
- Erros de I/O, rede, parsing
- Falhas não fatais

```python
# ✅ Bom uso
self.logger.error(f"Failed to load YOLO model: {e}", exc_info=True)
self.logger.error("Database query failed, data may be incomplete")
self.logger.error(f"Invalid configuration value: {key}={value}")

# ❌ Evitar
self.logger.error("Retrying connection...")  # Use WARNING
self.logger.error("System shutdown")  # Use CRITICAL se fatal
```

#### CRITICAL
**Usar para**:
- Erros que impedem continuação do aplicativo
- Corrupção de dados críticos
- Falhas de segurança graves
- Situações que exigem intervenção imediata

```python
# ✅ Bom uso
self.logger.critical("Failed to initialize core services, application cannot start")
self.logger.critical("Database file corrupted, manual recovery required")
self.logger.critical("Out of memory, shutting down")

# ❌ Evitar
self.logger.critical("Connection failed")  # Use ERROR ou WARNING
```

---

## 📝 PADRÕES DE MENSAGENS

### Estrutura Básica

```python
# Padrão geral: [Ação] [Contexto] [Detalhes opcionais]
self.logger.info("RTSP stream started")  # Simples
self.logger.info(f"Model loaded: {model_name}")  # Com contexto
self.logger.error(f"Failed to connect to {url}: {error}")  # Com detalhes
```

### Templates por Categoria

#### 1. Inicialização e Shutdown

```python
# Início de componente
self.logger.info(f"{ComponentName} initialized")
self.logger.info(f"{ComponentName} initialized (param1={val1}, param2={val2})")

# Fim de componente
self.logger.info(f"Shutting down {ComponentName}")
self.logger.info(f"{ComponentName} shutdown complete")

# Exemplos reais
self.logger.info("ByteTracker initialized (timeout=10 frames)")
self.logger.info("Shutting down VisionPipeline")
```

#### 2. Operações de Arquivo/Rede

```python
# Sucesso
self.logger.info(f"File loaded: {filepath}")
self.logger.info(f"Connected to {url}")
self.logger.info(f"Configuration saved to {path}")

# Falha
self.logger.error(f"Failed to load file: {filepath} - {error}")
self.logger.error(f"Connection failed: {url} - {error}")
self.logger.warning(f"File not found: {filepath}, using defaults")

# Exemplos reais
self.logger.info(f"File handler configured: {LOG_FILE}")
self.logger.error(f"Failed to connect to RTSP stream: timeout after {timeout}s")
```

#### 3. Processamento de Frames

```python
# Métricas periódicas (DEBUG)
self.logger.debug(f"Frame {frame_count}: {detections} detections, {tracks} tracks")
self.logger.debug(f"Processing time: {elapsed_ms:.2f}ms")

# Eventos importantes (INFO)
self.logger.info(f"Track {track_id} crossed tripwire: direction={direction}")

# Problemas (WARNING/ERROR)
self.logger.warning("Frame dropped: processing queue full")
self.logger.error(f"Frame processing failed: {error}")

# Exemplos reais
self.logger.debug(f"Frame {self._frame_count}: {len(detections)} detections, {len(tracks)} active tracks")
```

#### 4. Configuração e Validação

```python
# Carregamento
self.logger.info(f"Configuration loaded (version {version})")
self.logger.info(f"Using {setting_name}: {value}")

# Validação
self.logger.warning(f"Invalid config value '{key}': {value}, using default {default}")
self.logger.error(f"Configuration validation failed: {errors}")

# Exemplos reais
self.logger.info(f"LoggerService config: level={log_level}, console={console_enabled}")
self.logger.warning(f"Failed to load tripwire from config, using defaults: {e}")
```

#### 5. Eventos do Usuário

```python
# Ações
self.logger.info(f"User action: {action_name}")
self.logger.info(f"Setting changed: {setting_name} = {old_value} → {new_value}")

# Exemplos reais
self.logger.info("Tripwire updated in pipeline")
self.logger.info("Pipeline reset")
```

#### 6. Exceções

```python
# Com traceback completo (usar exc_info=True)
try:
    # código
except Exception as e:
    self.logger.error(f"Failed to {operation}: {e}", exc_info=True)

# Sem traceback (erro esperado)
except ValueError as e:
    self.logger.warning(f"Invalid input: {e}")

# Exemplos reais
self.logger.error(f"Failed to initialize pipeline: {e}", exc_info=True)
self.logger.error(f"Error processing frame: {e}", exc_info=True)
```

---

## ✅ BOAS PRÁTICAS

### 1. Nomenclatura de Loggers

```python
# ✅ Usar hierarquia de módulos
self.logger = logging.getLogger("gui.camera")
self.logger = logging.getLogger("detect.detector")
self.logger = logging.getLogger("track.tracker")
self.logger = logging.getLogger("persist.config_manager")

# ❌ Evitar nomes genéricos
self.logger = logging.getLogger("main")
self.logger = logging.getLogger("utils")
self.logger = logging.getLogger("app")

# Padrão recomendado
# Para arquivo src/detect/detector.py:
self.logger = logging.getLogger("detect.detector")

# Para arquivo src/gui/camera_tab.py:
self.logger = logging.getLogger("gui.camera")
```

### 2. Mensagens Claras e Concisas

```python
# ✅ Específica e acionável
self.logger.error(f"Failed to load YOLO model '{model_path}': file not found")

# ❌ Vaga e inútil
self.logger.error("Error")
self.logger.error("Something went wrong")
self.logger.error("Exception occurred")
```

### 3. Contexto Suficiente

```python
# ✅ Com contexto
self.logger.warning(f"Connection timeout for {url} after {timeout}s, retrying ({attempt}/{max_attempts})")

# ❌ Sem contexto
self.logger.warning("Timeout")
```

### 4. Performance

```python
# ✅ f-strings são avaliadas apenas se nível ativo
self.logger.debug(f"Expensive operation result: {expensive_function()}")
# Não é problema porque DEBUG pode ser desabilitado

# ⚠️ Para operações MUITO caras, verificar antes
if self.logger.isEnabledFor(logging.DEBUG):
    result = very_expensive_operation()
    self.logger.debug(f"Result: {result}")

# ✅ Evitar logs a cada frame em loops intensos
# Em vez disso, logar a cada N frames
if frame_count % 30 == 0:
    self.logger.debug(f"Frame {frame_count}: status update")
```

### 5. Sensibilidade de Dados

```python
# ❌ NUNCA logar dados sensíveis
self.logger.info(f"Connecting with password: {password}")
self.logger.debug(f"API key: {api_key}")

# ✅ Ofuscar ou omitir
self.logger.info(f"Connecting to {url} (credentials provided)")
self.logger.debug(f"API key: {'*' * 8}...{api_key[-4:]}")

# ✅ Para URLs com credenciais
from urllib.parse import urlparse
parsed = urlparse(url)
safe_url = f"{parsed.scheme}://{parsed.hostname}:{parsed.port}{parsed.path}"
self.logger.info(f"Connecting to {safe_url}")
```

### 6. Tratamento de Exceções

```python
# ✅ Incluir traceback para diagnóstico
try:
    critical_operation()
except Exception as e:
    self.logger.error(f"Critical operation failed: {e}", exc_info=True)
    raise  # Re-raise se não puder recuperar

# ✅ Para erros esperados, sem traceback
try:
    value = int(user_input)
except ValueError as e:
    self.logger.warning(f"Invalid input '{user_input}': {e}")
    return default_value

# ❌ Capturar e silenciar sem logar
try:
    something()
except:
    pass  # NUNCA FAZER ISSO
```

### 7. Frequência de Logs

```python
# ✅ Throttling para eventos repetitivos
class MyClass:
    def __init__(self):
        self._last_log_time = 0
        self._log_interval = 5.0  # segundos
    
    def process_frame(self):
        current_time = time.time()
        if current_time - self._last_log_time >= self._log_interval:
            self.logger.info(f"Processed {self.frame_count} frames")
            self._last_log_time = current_time

# ✅ Logs periódicos baseados em contadores
if self.frame_count % 100 == 0:
    self.logger.debug(f"Frame {self.frame_count}: {stats}")

# ❌ Log a cada iteração em loop rápido
for frame in stream:
    self.logger.debug(f"Processing frame {i}")  # Pode gerar milhares de logs
```

### 8. Logs de Startup

```python
# ✅ Log estruturado de inicialização (padrão do projeto)
logger.info("=" * 80)
logger.info(f"Application: {APP_FULL_NAME}")
logger.info(f"Python: {sys.version.split()[0]}")
logger.info(f"OS: {platform.system()} {platform.release()}")
logger.info(f"OpenCV: {cv2.__version__}")
logger.info(f"Storage: {APPDATA_BASE}")
logger.info("=" * 80)
```

### 9. Logs de Shutdown

```python
# ✅ Sequência ordenada
def shutdown(self):
    self.logger.info("Shutting down VisionPipeline")
    try:
        self.detector.cleanup()
        self.tracker.cleanup()
        self.counter.cleanup()
        self.logger.info("VisionPipeline shutdown complete")
    except Exception as e:
        self.logger.error(f"Error during shutdown: {e}", exc_info=True)
```

---

## ⚙️ CONFIGURAÇÃO E CONSTANTES

### Constantes Padrão (src/shared/constants.py)

```python
# Logging
DEFAULT_LOG_LEVEL = "INFO"  # DEBUG, INFO, WARNING, ERROR, CRITICAL
DEFAULT_LOG_MAX_FILE_SIZE_MB = 5  # Tamanho máximo antes de rotacionar
DEFAULT_LOG_MAX_FILES = 10  # Número de backups mantidos
DEFAULT_LOG_UI_BUFFER_SIZE = 5000  # Tamanho do buffer circular para UI

# Paths
LOGS_DIR = APPDATA_BASE / "logs"
LOG_FILE = LOGS_DIR / "app.log"
```

### Inicialização (src/Contador-Pessoas.py)

```python
# Uma vez no startup
from logger_service import LoggerService

# Carregar configurações
log_level = config.get("log_level", DEFAULT_LOG_LEVEL)
console_enabled = config.get("log_console_enabled", True)

# Configurar LoggerService
logger_service = LoggerService.setup(
    log_level=log_level,
    enable_console=console_enabled
)

logger.info("LoggerService configured")
```

### Uso em Módulos

```python
# Em qualquer módulo, simplesmente importar logging padrão
import logging

class MyClass:
    def __init__(self):
        # Nome do logger segue hierarquia do módulo
        self.logger = logging.getLogger("module.class_name")
    
    def my_method(self):
        self.logger.info("Operation completed")
```

### Acesso ao Buffer UI

```python
# Para exibir logs na interface
from logger_service import LoggerService

# Obter últimos N logs
logs = LoggerService.get_ui_logs(limit=200)

# Estrutura de cada log:
# {
#     'timestamp': datetime,
#     'level': str,  # "DEBUG", "INFO", etc.
#     'logger': str,  # "gui.camera"
#     'message': str  # Mensagem do log
# }

# Limpar buffer (raramente necessário)
LoggerService.clear_ui_logs()

# Forçar rotação de arquivo (automático, mas pode ser chamado)
LoggerService.rotate_logs()
```

---

## 💡 EXEMPLOS PRÁTICOS

### Exemplo 1: Classe Detector

```python
"""
src/detect/detector.py
"""
import logging
from pathlib import Path
from typing import List
from shared.models import Detection

class Detector:
    def __init__(self, model_path: str, confidence: float = 0.5):
        # Logger hierárquico
        self.logger = logging.getLogger("detect.detector")
        
        self.model_path = Path(model_path)
        self.confidence = confidence
        self.model = None
        
        # Log de inicialização com parâmetros
        self.logger.info(f"Detector initializing (model={model_path}, conf={confidence})")
    
    def load_model(self) -> None:
        """Carrega modelo YOLO"""
        try:
            # Log antes de operação cara
            self.logger.info(f"Loading YOLO model from {self.model_path}")
            
            if not self.model_path.exists():
                # Erro específico, não genérico
                raise FileNotFoundError(f"Model file not found: {self.model_path}")
            
            # Simular carregamento
            self.model = "loaded"  # Placeholder
            
            # Confirmar sucesso
            self.logger.info("YOLO model loaded successfully")
            
        except FileNotFoundError as e:
            # Erro esperado, sem traceback
            self.logger.error(f"Model load failed: {e}")
            raise
        except Exception as e:
            # Erro inesperado, com traceback
            self.logger.error(f"Unexpected error loading model: {e}", exc_info=True)
            raise
    
    def detect(self, frame) -> List[Detection]:
        """Detecta objetos no frame"""
        if self.model is None:
            self.logger.error("Cannot detect: model not loaded")
            return []
        
        try:
            # Operação
            detections = []  # Placeholder
            
            # Log DEBUG apenas para diagnóstico
            self.logger.debug(f"Detected {len(detections)} objects")
            
            return detections
            
        except Exception as e:
            # Log ERROR com contexto
            self.logger.error(f"Detection failed: {e}", exc_info=True)
            return []
    
    def cleanup(self) -> None:
        """Libera recursos"""
        self.logger.info("Cleaning up detector resources")
        self.model = None
        self.logger.debug("Detector cleanup complete")
```

### Exemplo 2: Service RTSP

```python
"""
src/capture/rtsp_service.py
"""
import logging
import time
from typing import Optional
from urllib.parse import urlparse

class RTSPService:
    def __init__(self, url: str, timeout: int = 5):
        self.logger = logging.getLogger("capture.rtsp")
        
        # Ofuscar credenciais no log
        parsed = urlparse(url)
        safe_url = f"{parsed.scheme}://{parsed.hostname}:{parsed.port}{parsed.path}"
        
        self.url = url
        self.timeout = timeout
        self._connected = False
        self._retry_count = 0
        self._max_retries = 5
        
        self.logger.info(f"RTSPService created (url={safe_url}, timeout={timeout}s)")
    
    def connect(self) -> bool:
        """Conecta ao stream RTSP"""
        while self._retry_count < self._max_retries:
            try:
                self.logger.info(f"Connecting to RTSP stream (attempt {self._retry_count + 1}/{self._max_retries})")
                
                # Simular conexão
                time.sleep(0.5)
                
                # Sucesso
                self._connected = True
                self._retry_count = 0
                self.logger.info("RTSP stream connected successfully")
                return True
                
            except TimeoutError as e:
                # Erro esperado, pode ocorrer
                self._retry_count += 1
                self.logger.warning(
                    f"Connection timeout ({self.timeout}s), "
                    f"retrying ({self._retry_count}/{self._max_retries})..."
                )
                time.sleep(2)  # Backoff
                
            except Exception as e:
                # Erro inesperado
                self.logger.error(f"Connection failed: {e}", exc_info=True)
                return False
        
        # Esgotadas as tentativas
        self.logger.error(f"Failed to connect after {self._max_retries} attempts")
        return False
    
    def read_frame(self) -> Optional[bytes]:
        """Lê próximo frame do stream"""
        if not self._connected:
            self.logger.error("Cannot read frame: not connected")
            return None
        
        try:
            # Leitura do frame
            frame = b"frame_data"  # Placeholder
            return frame
            
        except Exception as e:
            # Log ERROR mas não trava a aplicação
            self.logger.error(f"Frame read failed: {e}")
            self._connected = False  # Marcar como desconectado
            return None
    
    def disconnect(self) -> None:
        """Desconecta do stream"""
        if self._connected:
            self.logger.info("Disconnecting from RTSP stream")
            self._connected = False
            self.logger.debug("RTSP stream disconnected")
        else:
            self.logger.debug("Disconnect called but not connected")
```

### Exemplo 3: ConfigManager

```python
"""
src/persist/config_manager.py
"""
import logging
import json
from pathlib import Path
from typing import Dict, Any

class ConfigManager:
    def __init__(self, config_path: Path):
        self.logger = logging.getLogger("persist.config_manager")
        self.config_path = config_path
        self.config: Dict[str, Any] = {}
        
        self.logger.info(f"ConfigManager initialized (path={config_path})")
    
    def load(self) -> Dict[str, Any]:
        """Carrega configuração do disco"""
        try:
            if not self.config_path.exists():
                # Situação esperada na primeira execução
                self.logger.warning(
                    f"Config file not found: {self.config_path}, "
                    "using defaults"
                )
                return self._get_defaults()
            
            self.logger.info(f"Loading configuration from {self.config_path}")
            
            with open(self.config_path, 'r', encoding='utf-8') as f:
                self.config = json.load(f)
            
            # Validar
            errors = self._validate()
            if errors:
                self.logger.warning(f"Configuration validation issues: {errors}")
                self._apply_fixes(errors)
            
            version = self.config.get('config_version', 'unknown')
            self.logger.info(f"Configuration loaded successfully (version {version})")
            
            return self.config
            
        except json.JSONDecodeError as e:
            # Arquivo corrompido
            self.logger.error(f"Invalid JSON in config file: {e}")
            self.logger.warning("Using default configuration")
            return self._get_defaults()
            
        except Exception as e:
            # Erro inesperado
            self.logger.error(f"Failed to load configuration: {e}", exc_info=True)
            return self._get_defaults()
    
    def save(self) -> bool:
        """Salva configuração no disco"""
        try:
            # Criar diretório se não existir
            self.config_path.parent.mkdir(parents=True, exist_ok=True)
            
            self.logger.info(f"Saving configuration to {self.config_path}")
            
            with open(self.config_path, 'w', encoding='utf-8') as f:
                json.dump(self.config, f, indent=2, ensure_ascii=False)
            
            self.logger.info("Configuration saved successfully")
            return True
            
        except PermissionError as e:
            # Erro comum em Windows
            self.logger.error(f"Permission denied saving config: {e}")
            return False
            
        except Exception as e:
            self.logger.error(f"Failed to save configuration: {e}", exc_info=True)
            return False
    
    def get(self, key: str, default: Any = None) -> Any:
        """Obtém valor da configuração"""
        value = self.config.get(key, default)
        
        # Log DEBUG apenas se valor não for padrão
        if value != default:
            self.logger.debug(f"Config get: {key} = {value}")
        
        return value
    
    def set(self, key: str, value: Any) -> None:
        """Define valor na configuração"""
        old_value = self.config.get(key)
        self.config[key] = value
        
        # Log INFO de mudanças (útil para auditoria)
        if old_value != value:
            self.logger.info(f"Config changed: {key} = {old_value} → {value}")
        else:
            self.logger.debug(f"Config set: {key} = {value} (unchanged)")
    
    def _get_defaults(self) -> Dict[str, Any]:
        """Retorna configuração padrão"""
        self.logger.debug("Loading default configuration")
        return {
            "config_version": 1,
            "log_level": "INFO",
            # ... outros padrões
        }
    
    def _validate(self) -> List[str]:
        """Valida configuração carregada"""
        errors = []
        
        # Exemplo de validação
        log_level = self.config.get("log_level", "INFO")
        valid_levels = ["DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"]
        
        if log_level not in valid_levels:
            error_msg = f"Invalid log_level '{log_level}', expected one of {valid_levels}"
            errors.append(error_msg)
            self.logger.warning(error_msg)
        
        return errors
    
    def _apply_fixes(self, errors: List[str]) -> None:
        """Corrige erros de validação com valores padrão"""
        self.logger.info("Applying automatic fixes to configuration")
        # Implementar correções
```

### Exemplo 4: GUI Component

```python
"""
src/gui/camera_tab.py
"""
import logging
from PySide6.QtWidgets import QWidget
from PySide6.QtCore import QTimer

class CameraTab(QWidget):
    def __init__(self, rtsp_service, vision_pipeline):
        super().__init__()
        self.logger = logging.getLogger("gui.camera")
        
        self.rtsp_service = rtsp_service
        self.vision_pipeline = vision_pipeline
        
        self._frame_count = 0
        self._last_stats_time = 0
        
        self.logger.info("CameraTab initialized")
        self._setup_ui()
    
    def _setup_ui(self) -> None:
        """Configura interface"""
        self.logger.debug("Setting up camera tab UI")
        # Setup code...
        self.logger.debug("Camera tab UI setup complete")
    
    def start_stream(self) -> None:
        """Inicia captura de stream"""
        try:
            self.logger.info("Starting RTSP stream")
            
            if not self.rtsp_service.connect():
                self.logger.error("Failed to start stream: connection failed")
                self._show_error("Não foi possível conectar ao stream RTSP")
                return
            
            # Iniciar pipeline de visão
            self.vision_pipeline.initialize()
            
            # Iniciar timer de atualização
            self.timer = QTimer()
            self.timer.timeout.connect(self._update_frame)
            self.timer.start(33)  # ~30 FPS
            
            self.logger.info("Stream started successfully")
            
        except Exception as e:
            self.logger.error(f"Failed to start stream: {e}", exc_info=True)
            self._show_error(f"Erro ao iniciar stream: {e}")
    
    def stop_stream(self) -> None:
        """Para captura de stream"""
        self.logger.info("Stopping RTSP stream")
        
        try:
            if hasattr(self, 'timer'):
                self.timer.stop()
            
            self.rtsp_service.disconnect()
            self.logger.info("Stream stopped")
            
        except Exception as e:
            self.logger.error(f"Error stopping stream: {e}", exc_info=True)
    
    def _update_frame(self) -> None:
        """Atualiza frame na UI"""
        try:
            frame = self.rtsp_service.read_frame()
            if frame is None:
                self.logger.warning("Received null frame, stream may have disconnected")
                self.stop_stream()
                return
            
            # Processar frame
            result = self.vision_pipeline.process_frame(frame)
            
            # Atualizar UI
            # ...
            
            self._frame_count += 1
            
            # Log de estatísticas a cada 5 segundos
            import time
            current_time = time.time()
            if current_time - self._last_stats_time >= 5.0:
                self.logger.debug(f"Stream stats: {self._frame_count} frames processed")
                self._last_stats_time = current_time
            
        except Exception as e:
            self.logger.error(f"Frame update failed: {e}", exc_info=True)
    
    def _show_error(self, message: str) -> None:
        """Exibe erro na UI"""
        self.logger.debug(f"Showing error dialog: {message}")
        # Show dialog...
```

---

## ✔️ CHECKLIST DE IMPLEMENTAÇÃO

Use este checklist ao criar ou revisar logs em qualquer módulo:

### Setup Inicial
- [ ] Logger criado com nome hierárquico (`logging.getLogger("module.class")`)
- [ ] Logger é atributo de instância (`self.logger`)
- [ ] Não chamar `LoggerService.setup()` (apenas em startup)

### Mensagens
- [ ] Nível apropriado (DEBUG/INFO/WARNING/ERROR/CRITICAL)
- [ ] Mensagem clara e específica
- [ ] Contexto suficiente incluído
- [ ] Sem dados sensíveis (senhas, tokens, etc.)
- [ ] f-strings usadas para formatação

### Exceções
- [ ] `exc_info=True` em logs de exceções inesperadas
- [ ] Exceções esperadas sem traceback completo
- [ ] Nunca capturar exceção sem logar

### Performance
- [ ] Logs DEBUG em loops intensos são periódicos (a cada N iterações)
- [ ] Operações caras não são avaliadas desnecessariamente
- [ ] Throttling implementado para eventos frequentes

### Padrões
- [ ] Inicialização logada com INFO
- [ ] Shutdown logado com INFO
- [ ] Mudanças de configuração logadas
- [ ] Operações de I/O logadas (sucesso e falha)

### Testes
- [ ] Logs verificados em diferentes níveis (DEBUG, INFO, WARNING, ERROR)
- [ ] Mensagens são úteis para diagnóstico
- [ ] Sem logs duplicados ou redundantes
- [ ] Buffer UI não transborda com logs excessivos

---

## 📚 REFERÊNCIAS

### Documentação Python
- [Logging HOWTO](https://docs.python.org/3/howto/logging.html)
- [Logging Cookbook](https://docs.python.org/3/howto/logging-cookbook.html)
- [logging module](https://docs.python.org/3/library/logging.html)

### Arquivos do Projeto
- [src/logger_service/logger_service.py](src/logger_service/logger_service.py) - Implementação do LoggerService
- [src/shared/constants.py](src/shared/constants.py) - Constantes de logging
- [src/gui/log_tab.py](src/gui/log_tab.py) - Interface de visualização de logs
- [src/Contador-Pessoas.py](src/Contador-Pessoas.py) - Exemplo de inicialização

### Convenções
- **PEP 282**: [A Logging System](https://www.python.org/dev/peps/pep-0282/)
- **12-Factor App**: [Logs](https://12factor.net/logs)

---

## 📝 HISTÓRICO DE VERSÕES

| Versão | Data | Mudanças |
|--------|------|----------|
| 1.0 | 2026-02-03 | Versão inicial da especificação |

---

## 🤝 CONTRIBUINDO

Para sugestões de melhorias nesta especificação, considere:

1. **Novos Padrões**: Padrões de mensagens que facilitam diagnóstico
2. **Exemplos Práticos**: Casos de uso reais encontrados no projeto
3. **Otimizações**: Técnicas para reduzir overhead de logging
4. **Ferramentas**: Scripts para análise/filtro de logs

---

**Última atualização**: 03/02/2026  
**Mantido por**: LLM (Claude Sonnet 4.5)  
**Projeto**: Contador de Pessoas v4.0.9
