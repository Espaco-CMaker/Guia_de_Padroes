# ESPECIFICAÇÃO DE INTEGRAÇÃO TELEGRAM

**Versão**: 1.0  
**Data**: 03/02/2026  
**Objetivo**: Padronizar a integração com Telegram em aplicações Python com foco em segurança de dados sensíveis

---

## 📋 ÍNDICE

1. [Visão Geral](#visão-geral)
2. [Arquitetura da Integração](#arquitetura-da-integração)
3. [Configuração e Credenciais](#configuração-e-credenciais)
4. [Implementação do Bot](#implementação-do-bot)
5. [Envio de Mensagens](#envio-de-mensagens)
6. [Envio de Fotos/Arquivos](#envio-de-fotosarquivos)
7. [Tratamento de Erros](#tratamento-de-erros)
8. [Boas Práticas de Segurança](#boas-práticas-de-segurança)
9. [Exemplos Práticos](#exemplos-práticos)
10. [Checklist de Implementação](#checklist-de-implementação)

---

## 🎯 VISÃO GERAL

### Princípios Fundamentais

A integração com Telegram segue os seguintes princípios:

1. **Segurança de Credenciais**: Nunca logar ou exibir tokens/chat_ids
2. **Encapsulamento**: Uma classe dedicada (`TelegramBot`) gerencia toda comunicação
3. **Validação**: Verificar habilitação antes de enviar qualquer mensagem
4. **Tratamento Robusto**: Falhas de envio não devem interromper aplicação
5. **Configuração Centralizada**: Token e chat_id em arquivo separado (`.env` ou `config.ini`)
6. **Throttling**: Evitar envio excessivo de mensagens repetidas
7. **Logging Seguro**: Registrar operações sem expor dados sensíveis

### Componentes do Sistema

```
TelegramBot (Classe Principal)
├── Autenticação (token, chat_id)
├── Método enviar_mensagem()
├── Método enviar_foto()
├── Formatação de captions
└── Tratamento de erros

ConfigManager (Leitura Segura)
├── Carrega token de arquivo
├── Carrega chat_id de arquivo
└── Valida credenciais antes de usar

LogManager (Rastreamento)
├── Registra envios com sucesso
├── Registra falhas sem expor credenciais
└── Monitora erros de conexão
```

---

## 🏗️ ARQUITETURA DA INTEGRAÇÃO

### 1. Classe TelegramBot

**Responsabilidades**:
- Gerenciar credenciais de forma segura
- Implementar métodos para envio de mensagens e fotos
- Validar credenciais antes de operações
- Tratamento gracioso de falhas
- Logging de operações

**Características**:
- Atributo `enabled`: boolean indicando se credenciais são válidas
- URL base da API pré-construída apenas se habilitado
- Métodos retornam boolean (sucesso/falha)
- Nunca levanta exceções em caso de falha de envio

```python
class TelegramBot:
    def __init__(self, token: str, chat_id: str, log: Optional[LogManager] = None):
        """
        Inicializa bot Telegram.
        
        Args:
            token: Token do bot (@BotFather)
            chat_id: ID do chat/grupo para receber mensagens
            log: Gerenciador de logs (opcional)
        """
        self.token = (token or "").strip()
        self.chat_id = (chat_id or "").strip()
        self.enabled = bool(self.token and self.chat_id)
        self.base_url = f"https://api.telegram.org/bot{self.token}" if self.enabled else ""
        self.log = log
```

### 2. Validação de Credenciais

Credenciais são validadas apenas **uma única vez** na inicialização:

```python
# ✅ Padrão correto
class TelegramBot:
    def __init__(self, token: str, chat_id: str):
        self.token = (token or "").strip()
        self.chat_id = (chat_id or "").strip()
        self.enabled = bool(self.token and self.chat_id)  # Validação única
        
        if not self.enabled:
            # Log seguro: não inclui token/chat_id
            logger.warning("Telegram disabled: credentials not provided")
```

### 3. Tratamento de Falhas

Todas as operações retornam `bool` (True=sucesso, False=falha):

```python
def enviar_mensagem(self, texto: str) -> bool:
    if not self.enabled:
        return False
    try:
        # Operação
        r = requests.post(...)
        return r.status_code == 200
    except Exception as e:
        # Logar erro sem revelar credenciais
        if self.log:
            self.log.log("ERROR", f"Erro ao enviar mensagem Telegram: {e}")
        return False
```

---

## 🔐 CONFIGURAÇÃO E CREDENCIAIS

### 1. Obtenção de Credenciais

#### A. Token do Bot

1. Abrir Telegram e buscar por `@BotFather`
2. Enviar comando `/newbot`
3. Seguir instruções para nomear bot
4. **BotFather fornecerá token no formato**: `123456789:ABCdefGHIjklmnoPQRstuvWXYZabcdefg`
5. ⚠️ **GUARDAR TOKEN COM SEGURANÇA** - equivalente a uma senha

#### B. Chat ID

1. Adicionar bot ao chat/grupo desejado
2. Enviar qualquer mensagem
3. Chamar `https://api.telegram.org/bot<TOKEN>/getUpdates`
4. Procurar por `"chat":{id:xxxxx}` - este é o chat_id
5. ⚠️ **GUARDAR CHAT_ID COM SEGURANÇA** - identifica onde mensagens serão recebidas

### 2. Armazenamento Seguro

#### Opção A: Arquivo config.ini (Padrão)

```ini
[TELEGRAM]
bot_token = 123456789:ABCdefGHIjklmnoPQRstuvWXYZabcdefg
chat_id = -1001234567890
alert_mode = detections
```

**Segurança**:
- Arquivo deve ter permissões restritas (0600)
- Nunca commitir em repositório git
- Incluir em `.gitignore`

```
# .gitignore
config.ini
*.env
secrets.txt
```

#### Opção B: Variáveis de Ambiente (Recomendado para Produção)

```bash
export TELEGRAM_TOKEN="123456789:ABCdefGHIjklmnoPQRstuvWXYZabcdefg"
export TELEGRAM_CHAT_ID="-1001234567890"
```

```python
import os

token = os.getenv("TELEGRAM_TOKEN", "")
chat_id = os.getenv("TELEGRAM_CHAT_ID", "")
bot = TelegramBot(token, chat_id)
```

### 3. Validação de Credenciais

```python
# ✅ Bom: Validação silenciosa
def validate_credentials(token: str, chat_id: str) -> bool:
    """Valida se credenciais estão presentes"""
    token = (token or "").strip()
    chat_id = (chat_id or "").strip()
    return bool(token and chat_id)

# ❌ Ruim: Tentar fazer request de teste
def validate_credentials(token: str, chat_id: str) -> bool:
    try:
        r = requests.get(f"https://api.telegram.org/bot{token}/getMe")
        return r.status_code == 200
    except:
        return False  # Expõe token em logs se falhar
```

---

## 🤖 IMPLEMENTAÇÃO DO BOT

### 1. Classe Mínima Funcional

```python
"""
telegram_bot.py - Integração com Telegram API
"""
import logging
from typing import Optional
import requests

class TelegramBot:
    def __init__(self, token: str, chat_id: str, log: Optional[object] = None):
        """
        Inicializa instância do bot Telegram.
        
        Args:
            token: Token do bot (@BotFather)
            chat_id: ID do chat/grupo
            log: Gerenciador de logs (opcional)
        """
        self.logger = logging.getLogger("telegram")
        
        self.token = (token or "").strip()
        self.chat_id = (chat_id or "").strip()
        self.enabled = bool(self.token and self.chat_id)
        self.base_url = f"https://api.telegram.org/bot{self.token}" if self.enabled else ""
        self.log = log
        
        if not self.enabled:
            self.logger.warning("Telegram disabled: credentials not provided")
        else:
            self.logger.info("TelegramBot initialized")
    
    def enviar_mensagem(self, texto: str) -> bool:
        """
        Envia mensagem de texto para o chat.
        
        Args:
            texto: Mensagem a enviar
            
        Returns:
            True se sucesso, False caso contrário
        """
        if not self.enabled:
            return False
        
        try:
            url = f"{self.base_url}/sendMessage"
            data = {"chat_id": self.chat_id, "text": texto}
            r = requests.post(url, data=data, timeout=10)
            
            success = r.status_code == 200
            if success:
                self.logger.debug("Message sent successfully")
            else:
                self.logger.warning(f"Send failed: HTTP {r.status_code}")
            
            return success
            
        except Exception as e:
            if self.log:
                self.log.log("ERROR", f"Erro ao enviar mensagem Telegram: {e}")
            else:
                self.logger.error(f"Send error: {e}")
            return False
    
    def enviar_foto(self, foto_path: str, caption: str = "") -> bool:
        """
        Envia foto para o chat.
        
        Args:
            foto_path: Caminho para arquivo JPEG
            caption: Legenda (até 1024 caracteres)
            
        Returns:
            True se sucesso, False caso contrário
        """
        if not self.enabled:
            return False
        
        try:
            url = f"{self.base_url}/sendPhoto"
            with open(foto_path, "rb") as photo:
                files = {"photo": photo}
                data = {"chat_id": self.chat_id, "caption": caption}
                r = requests.post(url, files=files, data=data, timeout=30)
            
            success = r.status_code == 200
            if success:
                self.logger.debug("Photo sent successfully")
            else:
                self.logger.warning(f"Send failed: HTTP {r.status_code}")
            
            return success
            
        except FileNotFoundError:
            self.logger.error(f"Photo file not found: {foto_path}")
            return False
        except Exception as e:
            if self.log:
                self.log.log("ERROR", f"Erro ao enviar foto Telegram: {e}")
            else:
                self.logger.error(f"Send error: {e}")
            return False
```

### 2. Integração com LogManager

```python
# main.py ou startup
from telegram_bot import TelegramBot
from log_manager import LogManager

log_manager = LogManager("log.txt")

# Carregar credenciais de config
config = configparser.ConfigParser()
config.read("config.ini")

token = config["TELEGRAM"].get("bot_token", "")
chat_id = config["TELEGRAM"].get("chat_id", "")

# Criar bot (será desabilitado se credenciais vazias)
telegram = TelegramBot(token, chat_id, log=log_manager)

# Usar em detectors
detector = RTSPObjectDetector(
    cam_id=1,
    rtsp_url="rtsp://...",
    log=log_manager,
    telegram=telegram
)
```

---

## 💬 ENVIO DE MENSAGENS

### 1. Formato de Mensagens

Usar emojis e delimitadores para melhor formatação:

```python
# ✅ Bom: Formatado e legível
caption = (
    f"✅ ALERTA DE MOVIMENTO\n"
    f"{'━' * 12}\n"
    f"📹 Câmera 1\n"
    f"⏰ 03/02/2026 14:30:45\n"
    f"🔍 Detecção: pessoa\n"
    f"🟢 Confiança: 95.2%\n"
    f"v1.0\n"
    f"{'━' * 12}"
)

# ❌ Ruim: Sem formatação
caption = "Alerta camera 1 deteccao pessoa conf 95.2%"
```

### 2. Tipos de Mensagens Recomendadas

#### A. Inicialização do Sistema

```python
def formatar_msg_inicio(cameras_ativas: int, versao: str) -> str:
    """Formata mensagem de inicialização."""
    return (
        f"✅ SISTEMA INICIADO\n"
        f"{'━' * 12}\n"
        f"🎥 Câmeras ativas: {cameras_ativas}\n"
        f"🚀 Status: Monitorando\n"
        f"v{versao}\n"
        f"{'━' * 12}"
    )

# Enviar na inicialização
if telegram.enabled:
    msg = telegram.formatar_msg_inicio(cameras_ativas=4, versao="1.0")
    telegram.enviar_mensagem(msg)
```

#### B. Alertas Críticos

```python
# Mensagem de erro crítico
def formatar_alerta_critico(cam_id: int, erro: str, timestamp: str) -> str:
    """Formata alerta de erro crítico."""
    return (
        f"🔴 ALERTA CRÍTICO\n"
        f"{'━' * 12}\n"
        f"📹 Câmera {cam_id}\n"
        f"⏰ {timestamp}\n"
        f"⚠️ {erro}\n"
        f"{'━' * 12}"
    )
```

#### C. Encerramento do Sistema

```python
def formatar_msg_encerramento(total_deteccoes: int, versao: str) -> str:
    """Formata mensagem de encerramento."""
    return (
        f"⏹️ SISTEMA ENCERRADO\n"
        f"{'━' * 12}\n"
        f"👤 Detecções registradas: {total_deteccoes}\n"
        f"✓ Monitoramento finalizado\n"
        f"v{versao}\n"
        f"{'━' * 12}"
    )
```

### 3. Throttling de Mensagens

Evitar spam de mensagens idênticas:

```python
class TelegramBot:
    def __init__(self, token: str, chat_id: str):
        # ... código anterior ...
        self._last_message_hash = None
        self._last_message_time = 0.0
        self._min_interval_s = 5.0  # Mínimo 5s entre mensagens iguais
    
    def enviar_mensagem(self, texto: str) -> bool:
        if not self.enabled:
            return False
        
        # Throttling: evitar mensagens duplicadas muito próximas
        import hashlib
        import time
        
        msg_hash = hashlib.md5(texto.encode()).hexdigest()
        now = time.time()
        
        if (msg_hash == self._last_message_hash and 
            (now - self._last_message_time) < self._min_interval_s):
            self.logger.debug("Message throttled (duplicate within interval)")
            return False
        
        self._last_message_hash = msg_hash
        self._last_message_time = now
        
        try:
            url = f"{self.base_url}/sendMessage"
            data = {"chat_id": self.chat_id, "text": texto}
            r = requests.post(url, data=data, timeout=10)
            return r.status_code == 200
        except Exception as e:
            self.logger.error(f"Send error: {e}")
            return False
```

---

## 📸 ENVIO DE FOTOS/ARQUIVOS

### 1. Envio de Fotos com Legenda

```python
def enviar_foto(self, foto_path: str, caption: str = "") -> bool:
    """
    Envia foto com caption formatado.
    
    Limites Telegram:
    - Tamanho máximo: 50 MB
    - Formatos: JPEG, PNG, GIF, WebP
    - Caption máximo: 1024 caracteres
    """
    if not self.enabled:
        return False
    
    try:
        # Validar arquivo
        from pathlib import Path
        foto_file = Path(foto_path)
        if not foto_file.exists():
            self.logger.error(f"Photo file not found: {foto_path}")
            return False
        
        if foto_file.stat().st_size > 50_000_000:
            self.logger.error(f"Photo too large: {foto_file.stat().st_size} bytes")
            return False
        
        # Limitar caption
        caption = caption[:1024]
        
        url = f"{self.base_url}/sendPhoto"
        with open(foto_path, "rb") as photo:
            files = {"photo": photo}
            data = {"chat_id": self.chat_id, "caption": caption}
            r = requests.post(url, files=files, data=data, timeout=30)
        
        return r.status_code == 200
        
    except Exception as e:
        self.logger.error(f"Photo send error: {e}")
        return False
```

### 2. Formatação de Captions com Contexto

```python
def formatar_caption_deteccao(self, 
                             cam_id: int,
                             timestamp: str,
                             classes: List[str],
                             confianca: float,
                             fps: float,
                             latencia_ms: float) -> str:
    """
    Formata caption para foto de detecção com métricas.
    
    Args:
        cam_id: ID da câmera
        timestamp: Timestamp formatado
        classes: Lista de classes detectadas
        confianca: Confiança média (0-1)
        fps: Frames por segundo
        latencia_ms: Latência em millisegundos
    """
    # Determinar emoji baseado em confiança
    if confianca >= 0.7:
        emoji = "🟢"  # Verde
    elif confianca >= 0.5:
        emoji = "🟡"  # Amarelo
    else:
        emoji = "🟠"  # Laranja
    
    detection_text = ", ".join(classes) if classes else "objeto"
    
    return (
        f"🟢 ALERTA DE DETECÇÃO\n"
        f"{'━' * 12}\n"
        f"📹 Câmera {cam_id}\n"
        f"⏰ {timestamp}\n"
        f"🔍 Detectado: {detection_text}\n"
        f"{emoji} Confiança: {confianca*100:.1f}%\n"
        f"📡 FPS: {fps:.1f} | Latência: {latencia_ms:.1f}ms\n"
        f"{'━' * 12}"
    )
```

### 3. Validação de Arquivo Antes de Envio

```python
# ✅ Sempre validar antes de enviar
from pathlib import Path

def enviar_foto_segura(self, foto_path: str, caption: str = "") -> bool:
    """Envia foto com validações de segurança."""
    if not self.enabled:
        return False
    
    try:
        foto = Path(foto_path)
        
        # 1. Verificar existência
        if not foto.exists():
            self.logger.error(f"Photo not found: {foto_path}")
            return False
        
        # 2. Verificar tamanho
        tamanho = foto.stat().st_size
        if tamanho > 50_000_000:  # 50 MB
            self.logger.error(f"Photo too large: {tamanho} bytes")
            return False
        
        if tamanho == 0:
            self.logger.error("Photo file is empty")
            return False
        
        # 3. Verificar extensão
        sufixo = foto.suffix.lower()
        permitidos = {".jpg", ".jpeg", ".png", ".gif", ".webp"}
        if sufixo not in permitidos:
            self.logger.error(f"Invalid photo format: {sufixo}")
            return False
        
        # 4. Enviar
        url = f"{self.base_url}/sendPhoto"
        with open(foto_path, "rb") as photo:
            files = {"photo": photo}
            data = {
                "chat_id": self.chat_id,
                "caption": caption[:1024]  # Limitar a 1024 chars
            }
            r = requests.post(url, files=files, data=data, timeout=30)
        
        return r.status_code == 200
        
    except Exception as e:
        self.logger.error(f"Photo send error: {e}")
        return False
```

---

## ⚠️ TRATAMENTO DE ERROS

### 1. Estratégia de Tratamento

```python
# ✅ Padrão correto: Nunca impedir operação principal
class TelegramBot:
    def enviar_mensagem(self, texto: str) -> bool:
        if not self.enabled:
            return False
        
        try:
            url = f"{self.base_url}/sendMessage"
            data = {"chat_id": self.chat_id, "text": texto}
            r = requests.post(url, data=data, timeout=10)
            return r.status_code == 200
            
        except requests.Timeout:
            # Timeout - logging, mas não falha
            self.logger.warning("Telegram timeout (10s)")
            return False
            
        except requests.ConnectionError:
            # Sem conexão - logging, mas não falha
            self.logger.warning("Telegram connection error (no internet?)")
            return False
            
        except Exception as e:
            # Erro inesperado - log completo
            self.logger.error(f"Telegram error: {e}", exc_info=True)
            return False
```

### 2. Erros Específicos Telegram

```python
def enviar_mensagem(self, texto: str) -> bool:
    """Trata erros específicos da API Telegram."""
    if not self.enabled:
        return False
    
    try:
        url = f"{self.base_url}/sendMessage"
        data = {"chat_id": self.chat_id, "text": texto}
        r = requests.post(url, data=data, timeout=10)
        
        if r.status_code == 200:
            return True
        
        # Erros específicos da API
        elif r.status_code == 401:
            self.logger.error("Invalid token (401)")
            self.enabled = False  # Desabilitar futuro envio
            return False
            
        elif r.status_code == 403:
            self.logger.error("Forbidden - invalid chat_id? (403)")
            self.enabled = False
            return False
            
        elif r.status_code == 429:
            self.logger.warning("Rate limited by Telegram (429) - aguardando...")
            return False
            
        elif r.status_code >= 500:
            self.logger.warning(f"Telegram server error ({r.status_code})")
            return False
        
        else:
            self.logger.warning(f"Unexpected status: {r.status_code}")
            return False
            
    except requests.Timeout:
        self.logger.warning("Telegram timeout")
        return False
        
    except Exception as e:
        self.logger.error(f"Unexpected error: {e}")
        return False
```

### 3. Logging de Falhas

```python
# ✅ Bom: Logar erro sem expor credenciais
self.logger.error("Failed to send Telegram message: Connection timeout")
self.logger.error("Failed to send Telegram photo: Invalid chat_id")

# ❌ Ruim: Expõe credenciais em stacktrace
self.logger.error(f"Failed: {self.token} {self.chat_id}")
```

---

## 🔐 BOAS PRÁTICAS DE SEGURANÇA

### 1. Proteção de Credenciais

```python
# ✅ NUNCA fazer isso:
# 1. Logar o token completo
logger.debug(f"Token: {self.token}")  # ❌

# 2. Exibir em mensagens de erro
except Exception as e:
    print(f"Error with {self.token}: {e}")  # ❌

# 3. Incluir em URL visível
logger.info(f"Connecting to {self.base_url}")  # ❌ (contém token)

# 4. Commitar em repositório
git add config.ini  # ❌ (contém credenciais)

# ✅ FAZER isso:
# 1. Validar silenciosamente
self.enabled = bool(self.token and self.chat_id)

# 2. Logar sucesso/falha sem credenciais
if success:
    logger.info("Message sent to Telegram")
else:
    logger.warning("Failed to send Telegram message")

# 3. Usar arquivo .gitignore
echo "config.ini" >> .gitignore

# 4. Usar variáveis de ambiente
token = os.getenv("TELEGRAM_TOKEN", "")
```

### 2. Validação de Entrada

```python
# Sempre validar e sanitizar entrada
def enviar_mensagem(self, texto: str) -> bool:
    """Envia mensagem com validações."""
    if not self.enabled:
        return False
    
    # 1. Validar tipo
    if not isinstance(texto, str):
        self.logger.warning("Invalid message type")
        return False
    
    # 2. Limpar/limitar
    texto = (texto or "").strip()
    if not texto:
        self.logger.warning("Empty message")
        return False
    
    # 3. Limitar tamanho (Telegram: 4096 chars)
    texto = texto[:4096]
    
    # 4. Remover dados sensíveis (heurística)
    if self._contains_sensitive_data(texto):
        self.logger.warning("Message contains sensitive data")
        return False
    
    try:
        url = f"{self.base_url}/sendMessage"
        data = {"chat_id": self.chat_id, "text": texto}
        r = requests.post(url, data=data, timeout=10)
        return r.status_code == 200
    except Exception as e:
        self.logger.error(f"Send error: {e}")
        return False

def _contains_sensitive_data(self, texto: str) -> bool:
    """Detecta possível exposição de dados sensíveis."""
    sensitive_patterns = [
        "senha",
        "password",
        "token",
        "api_key",
        "secret",
    ]
    text_lower = texto.lower()
    return any(p in text_lower for p in sensitive_patterns)
```

### 3. Permisspções de Arquivo

```bash
# config.ini com credenciais deve ter permissões restritas
chmod 600 config.ini

# Verificar permissões
ls -la config.ini
# Deve exibir: -rw------- (600)
```

### 4. Rotação de Credenciais

Implementar suporte para atualizar token sem reiniciar:

```python
class TelegramBot:
    def atualizar_credenciais(self, token: str, chat_id: str) -> bool:
        """
        Atualiza credenciais e revalida.
        
        Returns:
            True se novos credenciais são válidos
        """
        self.token = (token or "").strip()
        self.chat_id = (chat_id or "").strip()
        self.enabled = bool(self.token and self.chat_id)
        
        if self.enabled:
            self.base_url = f"https://api.telegram.org/bot{self.token}"
            self.logger.info("Telegram credentials updated")
        else:
            self.base_url = ""
            self.logger.warning("Telegram disabled: invalid credentials")
        
        return self.enabled
```

---

## 💡 EXEMPLOS PRÁTICOS

### Exemplo 1: Classe Detector com Telegram

```python
"""
detector.py - Integração de detector YOLO com notificações Telegram
"""
import logging
from datetime import datetime
from typing import Optional
from pathlib import Path

class RTSPObjectDetector:
    def __init__(self, cam_id: int, telegram: TelegramBot, log: LogManager):
        """
        Inicializa detector com suporte Telegram.
        
        Args:
            cam_id: ID da câmera
            telegram: Instância TelegramBot
            log: Instância LogManager
        """
        self.logger = logging.getLogger("detect.detector")
        self.cam_id = cam_id
        self.telegram = telegram
        self.log = log
        self.telegram_mode = "detections"  # all | detections | none
        
        self.logger.info(f"Detector initialized (cam={cam_id}, telegram={self.telegram.enabled})")
    
    def _save_and_notify(self, frame, event_id: str, confidence: float):
        """
        Salva foto e envia notificação Telegram.
        
        Args:
            frame: Frame OpenCV
            event_id: ID único do evento
            confidence: Confiança da detecção
        """
        # Salvar foto
        ts = datetime.now().strftime("%Y%m%d_%H%M%S")
        filename = f"{ts}_CAM{self.cam_id}_EVT{event_id}.jpg"
        foto_path = Path("fotos") / filename
        foto_path.parent.mkdir(exist_ok=True)
        
        import cv2
        cv2.imwrite(str(foto_path), frame)
        
        self.log.log(
            "INFO",
            f"Photo saved: {filename}",
            self.cam_id
        )
        
        # Notificar Telegram
        if self.telegram_mode in ("all", "detections") and self.telegram.enabled:
            timestamp = datetime.now().strftime("%d/%m/%Y %H:%M:%S")
            
            caption = (
                f"🟢 ALERTA DE DETECÇÃO\n"
                f"{'━' * 12}\n"
                f"📹 Câmera {self.cam_id}\n"
                f"⏰ {timestamp}\n"
                f"🔍 Detectado: pessoa\n"
                f"🟢 Confiança: {confidence*100:.1f}%\n"
                f"{'━' * 12}"
            )
            
            success = self.telegram.enviar_foto(str(foto_path), caption)
            
            if success:
                self.log.log("INFO", "Photo sent to Telegram", self.cam_id)
            else:
                self.log.log("WARN", "Failed to send photo to Telegram", self.cam_id)
```

### Exemplo 2: Startup e Shutdown

```python
"""
main.py - Inicialização e shutdown com notificações Telegram
"""
import logging
import configparser
from pathlib import Path

class Application:
    def __init__(self):
        self.logger = logging.getLogger("main")
        
        # Carregar configuração
        config = configparser.ConfigParser()
        config.read("config.ini")
        
        # Inicializar Telegram
        token = config["TELEGRAM"].get("bot_token", "")
        chat_id = config["TELEGRAM"].get("chat_id", "")
        self.telegram = TelegramBot(token, chat_id)
        
        # Inicializar cameras (detectors)
        self.detectors = []
        for cam_id in range(1, 5):
            enabled = config[f"CAM{cam_id}"].getboolean("enabled", False)
            if not enabled:
                continue
            
            rtsp_url = config[f"CAM{cam_id}"].get("rtsp_url", "")
            detector = RTSPObjectDetector(cam_id, self.telegram)
            self.detectors.append(detector)
        
        self.logger.info(f"Application initialized with {len(self.detectors)} cameras")
    
    def start(self):
        """Inicia monitoramento e envia notificação."""
        self.logger.info("Application starting")
        
        # Enviar mensagem de inicialização
        if self.telegram.enabled:
            msg = (
                f"✅ SISTEMA INICIADO\n"
                f"{'━' * 12}\n"
                f"🎥 Câmeras ativas: {len(self.detectors)}\n"
                f"🚀 Status: Monitorando\n"
                f"v1.0\n"
                f"{'━' * 12}"
            )
            self.telegram.enviar_mensagem(msg)
        
        # Iniciar detectors
        for detector in self.detectors:
            detector.start()
    
    def stop(self):
        """Para monitoramento e envia notificação."""
        self.logger.info("Application stopping")
        
        # Parar detectors
        for detector in self.detectors:
            detector.stop()
        
        # Total de detecções
        total_deteccoes = sum(d.detections_total for d in self.detectors)
        
        # Enviar mensagem de encerramento
        if self.telegram.enabled:
            msg = (
                f"⏹️ SISTEMA ENCERRADO\n"
                f"{'━' * 12}\n"
                f"👤 Detecções registradas: {total_deteccoes}\n"
                f"✓ Monitoramento finalizado\n"
                f"v1.0\n"
                f"{'━' * 12}"
            )
            self.telegram.enviar_mensagem(msg)
        
        self.logger.info("Application stopped")

# Uso
if __name__ == "__main__":
    app = Application()
    try:
        app.start()
        # ... running ...
    finally:
        app.stop()
```

### Exemplo 3: Tratamento de Erros Críticos

```python
# Em LogManager
class LogManager:
    def __init__(self, telegram: TelegramBot):
        self.telegram = telegram
        self._sending_alert = False
    
    def log(self, level: str, msg: str, cam: Optional[int] = None) -> None:
        """Log com suporte a alertas críticos Telegram."""
        
        # Registrar em arquivo
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        line = f"[{timestamp}] [{level}] [CAM{cam}] {msg}\n"
        
        with open("log.txt", "a") as f:
            f.write(line)
        
        # Enviar alerta crítico para Telegram
        if level in ("ERROR", "CRITICAL") and self.telegram.enabled:
            if not self._sending_alert:  # Evitar recursão
                try:
                    self._sending_alert = True
                    
                    # Determinar emoji
                    if "RTSP" in msg or "conexão" in msg.lower():
                        emoji = "🔴"  # Crítico
                    else:
                        emoji = "🟠"  # Erro
                    
                    caption = (
                        f"{emoji} ALERTA DO SISTEMA\n"
                        f"{'━' * 12}\n"
                        f"📹 Câmera: {cam or 'N/D'}\n"
                        f"⏰ {timestamp}\n"
                        f"⚠️ {msg}\n"
                        f"{'━' * 12}"
                    )
                    
                    self.telegram.enviar_mensagem(caption)
                    
                finally:
                    self._sending_alert = False
```

---

## ✔️ CHECKLIST DE IMPLEMENTAÇÃO

Use este checklist ao integrar Telegram em novo projeto:

### Setup Inicial
- [ ] Criar bot com @BotFather e obter token
- [ ] Adicionar bot ao chat/grupo e obter chat_id
- [ ] Armazenar credenciais em `config.ini` ou variáveis de ambiente
- [ ] Adicionar `config.ini` e `.env` ao `.gitignore`
- [ ] Criar classe `TelegramBot` com métodos `enviar_mensagem()` e `enviar_foto()`
- [ ] Inicializar Telegram com validação silenciosa (`enabled` boolean)

### Envio de Mensagens
- [ ] Sempre validar `telegram.enabled` antes de enviar
- [ ] Nunca logar token ou chat_id
- [ ] Usar f-strings e emojis para formatação clara
- [ ] Limitar caption a 1024 caracteres (fotos) e 4096 (mensagens)
- [ ] Implementar throttling para evitar spam

### Envio de Fotos
- [ ] Validar existência de arquivo antes de enviar
- [ ] Validar tamanho (máximo 50 MB)
- [ ] Validar extensão (.jpg, .png, .gif, .webp)
- [ ] Incluir caption com contexto (câmera, timestamp, confiança)
- [ ] Usar timeout adequado (30s para fotos)

### Tratamento de Erros
- [ ] Nunca deixar falha Telegram interromper aplicação
- [ ] Logar falhas com contexto (sem credenciais)
- [ ] Tratar TimeoutError específico
- [ ] Tratar ConnectionError específico
- [ ] Detectar e logar erros 401/403 (credenciais inválidas)
- [ ] Implementar retry logic se necessário

### Segurança
- [ ] Validar entrada (tipo, tamanho, formato)
- [ ] Remover dados sensíveis de mensagens
- [ ] Ofuscar URLs RTSP se necessário
- [ ] Usar variáveis de ambiente para credenciais em produção
- [ ] Implementar rotação/atualização de credenciais

### Logging
- [ ] Logar inicialização do TelegramBot
- [ ] Logar sucesso/falha de envios (sem credenciais)
- [ ] Logar tempo de resposta para diagnóstico
- [ ] Incluir metricas (FPS, latência, confiança) em mensagens
- [ ] Registrar eventos críticos que merecem alerta

### Testes
- [ ] Testar envio de mensagem simples
- [ ] Testar envio de foto com caption
- [ ] Testar com credenciais inválidas (deve desabilitar silenciosamente)
- [ ] Testar sem conexão de internet
- [ ] Testar com arquivo de foto inexistente
- [ ] Testar com arquivo de foto muito grande
- [ ] Testar throttling de mensagens duplicadas

---

## 📚 REFERÊNCIAS

### Documentação Oficial
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [BotFather Guide](https://core.telegram.org/bots#6-botfather)
- [sendMessage Method](https://core.telegram.org/bots/api#sendmessage)
- [sendPhoto Method](https://core.telegram.org/bots/api#sendphoto)

### Bibliotecas Python
- [python-telegram-bot](https://python-telegram-bot.readthedocs.io/) - Wrapper oficial
- [requests](https://docs.python-requests.org/) - HTTP para API raw

### Conversão de IDs
- [Get Chat ID](https://www.siteguarding.com/en/how-to-find-telegram-bot-chat-id)
- [Converter de @username para ID](https://t.me/username_to_id_bot)

---

## 📝 HISTÓRICO DE VERSÕES

| Versão | Data | Mudanças |
|--------|------|----------|
| 1.0 | 03/02/2026 | Versão inicial da especificação |

---

## 🤝 CONTRIBUINDO

Para sugestões de melhorias nesta especificação, considere:

1. **Novos Padrões**: Padrões de mensagens que facilitam diagnóstico
2. **Exemplos Práticos**: Casos de uso reais encontrados em implementações
3. **Otimizações**: Técnicas para melhorar performance/segurança
4. **Tratamento de Erros**: Novos cenários de erro descobertos

---

**Última atualização**: 03/02/2026  
**Mantido por**: LLM (Claude Sonnet 4.5)  
**Projeto**: AlertaIntruso v4.3.19
