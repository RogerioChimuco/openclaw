# Análise Técnica: Integração OAuth do GitHub Copilot no OpenClaw

## Sumário Executivo

Este documento detalha como o projeto OpenClaw implementa a integração com o GitHub Copilot sem uso de chaves de API tradicionais, utilizando OAuth2 Device Flow para autenticação e obtendo tokens de acesso aos modelos de IA através da assinatura do GitHub Copilot do usuário.

## Índice

1. [Visão Geral da Arquitetura](#1-visão-geral-da-arquitetura)
2. [Fluxo de Autenticação OAuth](#2-fluxo-de-autenticação-oauth)
3. [Troca de Token Copilot](#3-troca-de-token-copilot)
4. [Descoberta e Listagem de Modelos](#4-descoberta-e-listagem-de-modelos)
5. [Rastreamento de Uso e Limites](#5-rastreamento-de-uso-e-limites)
6. [Implementação em Python/Django](#6-implementação-em-pythondjango)

---

## 1. Visão Geral da Arquitetura

### 1.1 Componentes Principais

O OpenClaw implementa a integração com o GitHub Copilot através de três componentes principais:

1. **Autenticação OAuth Device Flow** (`src/providers/github-copilot-auth.ts`)
2. **Troca de Token Copilot** (`src/providers/github-copilot-token.ts`)
3. **Gestão de Modelos** (`src/providers/github-copilot-models.ts`)
4. **Monitoramento de Uso** (`src/infra/provider-usage.fetch.copilot.ts`)

### 1.2 Fluxo de Dados

```
┌─────────────────┐
│  Usuário CLI    │
└────────┬────────┘
         │
         ├─── 1. openclaw models auth login --provider github-copilot
         │
         ▼
┌─────────────────────────────────────────────────┐
│   GitHub OAuth Device Flow                      │
│   (github-copilot-auth.ts)                     │
│                                                 │
│   CLIENT_ID: Iv1.b507a08c87ecfe98              │
│   URL: https://github.com/login/device/code    │
└────────┬────────────────────────────────────────┘
         │
         ├─── 2. Usuário autoriza no navegador
         │
         ▼
┌─────────────────────────────────────────────────┐
│   Obter GitHub Access Token                     │
│   URL: https://github.com/login/oauth/         │
│        access_token                             │
└────────┬────────────────────────────────────────┘
         │
         ├─── 3. Token armazenado em auth-profiles.json
         │
         ▼
┌─────────────────────────────────────────────────┐
│   Troca por Copilot Token                       │
│   (github-copilot-token.ts)                    │
│                                                 │
│   URL: https://api.github.com/                  │
│        copilot_internal/v2/token               │
└────────┬────────────────────────────────────────┘
         │
         ├─── 4. Copilot Token com endpoint proxy
         │
         ▼
┌─────────────────────────────────────────────────┐
│   Uso dos Modelos IA                            │
│   Base URL: https://api.individual.             │
│             githubcopilot.com                   │
│                                                 │
│   Modelos: gpt-4o, gpt-4.1, o1, o3-mini, etc   │
└─────────────────────────────────────────────────┘
```

---

## 2. Fluxo de Autenticação OAuth

### 2.1 Implementação do Device Flow

O OpenClaw usa o **OAuth 2.0 Device Authorization Grant** (RFC 8628) para autenticação sem navegador no mesmo dispositivo.

#### Arquivo: `src/providers/github-copilot-auth.ts`

```typescript
// Credenciais OAuth do GitHub
const CLIENT_ID = "Iv1.b507a08c87ecfe98";
const DEVICE_CODE_URL = "https://github.com/login/device/code";
const ACCESS_TOKEN_URL = "https://github.com/login/oauth/access_token";
```

### 2.2 Passo a Passo do Device Flow

#### Passo 1: Requisitar Código de Dispositivo

```typescript
async function requestDeviceCode(params: { scope: string }): Promise<DeviceCodeResponse> {
  const body = new URLSearchParams({
    client_id: CLIENT_ID,
    scope: params.scope, // "read:user"
  });

  const res = await fetch(DEVICE_CODE_URL, {
    method: "POST",
    headers: {
      Accept: "application/json",
      "Content-Type": "application/x-www-form-urlencoded",
    },
    body,
  });

  const json = await res.json();
  // Retorna: device_code, user_code, verification_uri, expires_in, interval
  return json;
}
```

**Resposta Exemplo:**
```json
{
  "device_code": "3584d83530557fdd1f46af8289938c8ef79f9dc5",
  "user_code": "WDJB-MJHT",
  "verification_uri": "https://github.com/login/device",
  "expires_in": 900,
  "interval": 5
}
```

#### Passo 2: Usuário Autoriza no Navegador

O usuário acessa `verification_uri` e insere o `user_code` exibido no terminal.

#### Passo 3: Polling para Access Token

```typescript
async function pollForAccessToken(params: {
  deviceCode: string;
  intervalMs: number;
  expiresAt: number;
}): Promise<string> {
  while (Date.now() < params.expiresAt) {
    const res = await fetch(ACCESS_TOKEN_URL, {
      method: "POST",
      headers: {
        Accept: "application/json",
        "Content-Type": "application/x-www-form-urlencoded",
      },
      body: new URLSearchParams({
        client_id: CLIENT_ID,
        device_code: params.deviceCode,
        grant_type: "urn:ietf:params:oauth:grant-type:device_code",
      }),
    });

    const json = await res.json();
    
    if (json.access_token) {
      return json.access_token; // Token do GitHub obtido!
    }
    
    // Tratamento de erros
    if (json.error === "authorization_pending") {
      await sleep(params.intervalMs);
      continue;
    }
    if (json.error === "slow_down") {
      await sleep(params.intervalMs + 2000);
      continue;
    }
    
    throw new Error(`OAuth error: ${json.error}`);
  }
}
```

#### Passo 4: Armazenamento Seguro

```typescript
// Salvar em ~/.openclaw/credentials/auth-profiles.json
upsertAuthProfile({
  profileId: "github-copilot:github",
  credential: {
    type: "token",
    provider: "github-copilot",
    token: accessToken,
  },
});
```

---

## 3. Troca de Token Copilot

### 3.1 Por que Trocar o Token?

O token OAuth do GitHub não pode ser usado diretamente para acessar modelos Copilot. É necessário trocá-lo por um **Copilot API Token** específico.

#### Arquivo: `src/providers/github-copilot-token.ts`

### 3.2 Endpoint de Troca

```typescript
const COPILOT_TOKEN_URL = "https://api.github.com/copilot_internal/v2/token";

export async function resolveCopilotApiToken(params: {
  githubToken: string;
}): Promise<{
  token: string;
  expiresAt: number;
  baseUrl: string;
}> {
  const res = await fetch(COPILOT_TOKEN_URL, {
    method: "GET",
    headers: {
      Accept: "application/json",
      Authorization: `Bearer ${params.githubToken}`,
    },
  });

  const json = await res.json();
  // json.token = Copilot token especial
  // json.expires_at = timestamp de expiração
  
  return {
    token: json.token,
    expiresAt: json.expires_at * 1000,
    baseUrl: deriveCopilotApiBaseUrlFromToken(json.token),
  };
}
```

### 3.3 Formato do Token Copilot

O token retornado é uma string delimitada por ponto e vírgula com pares chave-valor:

```
ghu_xxxxxxxxxxxx;exp=1234567890;proxy-ep=https://proxy.individual.githubcopilot.com;...
```

### 3.4 Extração do Base URL

```typescript
export function deriveCopilotApiBaseUrlFromToken(token: string): string | null {
  // O token contém: proxy-ep=https://proxy.individual.githubcopilot.com
  const match = token.match(/(?:^|;)\s*proxy-ep=([^;\s]+)/i);
  const proxyEp = match?.[1]?.trim();
  
  if (!proxyEp) {
    return "https://api.individual.githubcopilot.com"; // fallback
  }
  
  // Converter proxy.* -> api.*
  const host = proxyEp
    .replace(/^https?:\/\//, "")
    .replace(/^proxy\./i, "api.");
  
  return `https://${host}`;
}
```

### 3.5 Cache de Token

```typescript
// Cachear em: ~/.openclaw/credentials/github-copilot.token.json
export type CachedCopilotToken = {
  token: string;
  expiresAt: number; // milliseconds
  updatedAt: number;
};

function isTokenUsable(cache: CachedCopilotToken): boolean {
  // Margem de segurança de 5 minutos
  return cache.expiresAt - Date.now() > 5 * 60 * 1000;
}
```

---

## 4. Descoberta e Listagem de Modelos

### 4.1 Modelos Disponíveis por Assinatura

O OpenClaw mantém uma lista padrão de modelos Copilot, mas a disponibilidade depende do plano do usuário.

#### Arquivo: `src/providers/github-copilot-models.ts`

```typescript
const DEFAULT_MODEL_IDS = [
  "gpt-4o",
  "gpt-4.1",
  "gpt-4.1-mini",
  "gpt-4.1-nano",
  "o1",
  "o1-mini",
  "o3-mini",
] as const;

export function buildCopilotModelDefinition(modelId: string): ModelDefinitionConfig {
  return {
    id: modelId,
    name: modelId,
    api: "openai-responses", // Copilot usa API compatível com OpenAI
    reasoning: false,
    input: ["text", "image"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128_000,
    maxTokens: 8192,
  };
}
```

### 4.2 Verificação de Disponibilidade

O OpenClaw **não consulta uma API de lista de modelos**. Em vez disso:

1. Mantém uma lista ampla de modelos conhecidos
2. Tenta usar o modelo solicitado
3. Se o modelo não estiver disponível na assinatura, o GitHub Copilot retorna erro
4. O usuário pode então remover modelos indisponíveis da configuração

### 4.3 Planos e Modelos

A disponibilidade de modelos varia por plano:

- **GitHub Copilot Free**: Modelos básicos limitados
- **GitHub Copilot Individual**: gpt-4o, claude-sonnet, etc.
- **GitHub Copilot Business**: Modelos avançados
- **GitHub Copilot Enterprise**: Todos os modelos incluindo o1, o3-mini

---

## 5. Rastreamento de Uso e Limites

### 5.1 API de Uso do Copilot

#### Arquivo: `src/infra/provider-usage.fetch.copilot.ts`

```typescript
export async function fetchCopilotUsage(
  token: string,
  timeoutMs: number,
): Promise<ProviderUsageSnapshot> {
  const res = await fetch("https://api.github.com/copilot_internal/user", {
    headers: {
      Authorization: `token ${token}`,
      "Editor-Version": "vscode/1.96.2",
      "User-Agent": "GitHubCopilotChat/0.26.7",
      "X-Github-Api-Version": "2025-04-01",
    },
  });

  const data = await res.json();
  return {
    provider: "github-copilot",
    displayName: "GitHub Copilot",
    windows: [
      {
        label: "Premium",
        usedPercent: 100 - data.quota_snapshots.premium_interactions.percent_remaining,
      },
      {
        label: "Chat",
        usedPercent: 100 - data.quota_snapshots.chat.percent_remaining,
      },
    ],
    plan: data.copilot_plan,
  };
}
```

### 5.2 Estrutura de Resposta

```json
{
  "copilot_plan": "individual",
  "quota_snapshots": {
    "premium_interactions": {
      "percent_remaining": 75.5
    },
    "chat": {
      "percent_remaining": 92.3
    }
  }
}
```

### 5.3 Headers Importantes

- `Editor-Version`: Identifica como VSCode
- `User-Agent`: Identifica como extensão Copilot
- `X-Github-Api-Version`: Versão da API

---

## 6. Implementação em Python/Django

### 6.1 Estrutura do Projeto Django

```
copilot_integration/
├── __init__.py
├── models.py          # Modelos de dados
├── oauth.py           # Lógica OAuth Device Flow
├── copilot_api.py     # Cliente API Copilot
├── views.py           # Views/endpoints Django
├── urls.py            # Rotas
└── utils.py           # Utilidades
```

### 6.2 Implementação do OAuth Device Flow

#### `oauth.py`

```python
import time
import requests
from typing import Dict, Optional
from django.conf import settings

class GitHubDeviceFlow:
    """Implementa OAuth 2.0 Device Authorization Grant para GitHub"""
    
    CLIENT_ID = "Iv1.b507a08c87ecfe98"
    DEVICE_CODE_URL = "https://github.com/login/device/code"
    ACCESS_TOKEN_URL = "https://github.com/login/oauth/access_token"
    
    def __init__(self):
        self.session = requests.Session()
        self.session.headers.update({
            "Accept": "application/json",
            "Content-Type": "application/x-www-form-urlencoded",
        })
    
    def request_device_code(self, scope: str = "read:user") -> Dict:
        """
        Passo 1: Requisitar código de dispositivo
        
        Returns:
            {
                "device_code": str,
                "user_code": str,
                "verification_uri": str,
                "expires_in": int,
                "interval": int
            }
        """
        data = {
            "client_id": self.CLIENT_ID,
            "scope": scope,
        }
        
        response = self.session.post(self.DEVICE_CODE_URL, data=data)
        response.raise_for_status()
        
        result = response.json()
        
        if not all(k in result for k in ["device_code", "user_code", "verification_uri"]):
            raise ValueError("Resposta incompleta do GitHub")
        
        return result
    
    def poll_for_access_token(
        self,
        device_code: str,
        interval: int = 5,
        expires_in: int = 900
    ) -> str:
        """
        Passo 2: Fazer polling até usuário autorizar
        
        Args:
            device_code: Código do dispositivo
            interval: Intervalo entre requisições (segundos)
            expires_in: Tempo até expiração (segundos)
        
        Returns:
            GitHub access token
        
        Raises:
            TimeoutError: Se expirar antes da autorização
            ValueError: Se usuário negar acesso
        """
        expires_at = time.time() + expires_in
        
        while time.time() < expires_at:
            data = {
                "client_id": self.CLIENT_ID,
                "device_code": device_code,
                "grant_type": "urn:ietf:params:oauth:grant-type:device_code",
            }
            
            response = self.session.post(self.ACCESS_TOKEN_URL, data=data)
            response.raise_for_status()
            
            result = response.json()
            
            # Sucesso - token obtido
            if "access_token" in result:
                return result["access_token"]
            
            # Erros de polling
            error = result.get("error")
            
            if error == "authorization_pending":
                # Usuário ainda não autorizou
                time.sleep(interval)
                continue
            
            elif error == "slow_down":
                # GitHub pediu para diminuir frequência
                time.sleep(interval + 2)
                continue
            
            elif error == "expired_token":
                raise TimeoutError("Código de dispositivo expirou")
            
            elif error == "access_denied":
                raise ValueError("Usuário negou acesso")
            
            else:
                raise ValueError(f"Erro OAuth: {error}")
        
        raise TimeoutError("Timeout aguardando autorização")
    
    def complete_device_flow(self, scope: str = "read:user") -> Dict:
        """
        Fluxo completo: requisitar código + polling
        
        Returns:
            {
                "access_token": str,
                "user_code": str,
                "verification_uri": str
            }
        """
        # Passo 1: Obter código
        device_info = self.request_device_code(scope)
        
        print(f"Visite: {device_info['verification_uri']}")
        print(f"Código: {device_info['user_code']}")
        
        # Passo 2: Polling
        access_token = self.poll_for_access_token(
            device_code=device_info["device_code"],
            interval=device_info.get("interval", 5),
            expires_in=device_info.get("expires_in", 900),
        )
        
        return {
            "access_token": access_token,
            "user_code": device_info["user_code"],
            "verification_uri": device_info["verification_uri"],
        }
```

### 6.3 Cliente API do Copilot

#### `copilot_api.py`

```python
import re
import time
import requests
from typing import Dict, List, Optional
from dataclasses import dataclass
from datetime import datetime, timedelta

@dataclass
class CopilotToken:
    """Token do Copilot com metadados"""
    token: str
    expires_at: datetime
    base_url: str
    updated_at: datetime

class CopilotAPIClient:
    """Cliente para API do GitHub Copilot"""
    
    COPILOT_TOKEN_URL = "https://api.github.com/copilot_internal/v2/token"
    COPILOT_USER_URL = "https://api.github.com/copilot_internal/user"
    DEFAULT_BASE_URL = "https://api.individual.githubcopilot.com"
    
    def __init__(self, github_token: str):
        self.github_token = github_token
        self._cached_token: Optional[CopilotToken] = None
    
    def exchange_for_copilot_token(self) -> CopilotToken:
        """
        Trocar GitHub token por Copilot token
        
        Returns:
            CopilotToken com token, expiração e base URL
        """
        # Verificar cache
        if self._cached_token and self._is_token_valid(self._cached_token):
            return self._cached_token
        
        # Requisitar novo token
        response = requests.get(
            self.COPILOT_TOKEN_URL,
            headers={
                "Accept": "application/json",
                "Authorization": f"Bearer {self.github_token}",
            }
        )
        response.raise_for_status()
        
        data = response.json()
        
        token_str = data.get("token")
        expires_at_raw = data.get("expires_at")
        
        if not token_str or not expires_at_raw:
            raise ValueError("Resposta incompleta do Copilot token endpoint")
        
        # Parsear expiração (pode ser segundos ou milissegundos)
        expires_at_timestamp = int(expires_at_raw)
        if expires_at_timestamp > 10_000_000_000:
            # Milissegundos
            expires_at = datetime.fromtimestamp(expires_at_timestamp / 1000)
        else:
            # Segundos
            expires_at = datetime.fromtimestamp(expires_at_timestamp)
        
        # Extrair base URL do token
        base_url = self._extract_base_url_from_token(token_str)
        
        # Criar e cachear token
        copilot_token = CopilotToken(
            token=token_str,
            expires_at=expires_at,
            base_url=base_url,
            updated_at=datetime.now(),
        )
        
        self._cached_token = copilot_token
        return copilot_token
    
    def _extract_base_url_from_token(self, token: str) -> str:
        """
        Extrair base URL do token Copilot
        
        Token formato: ghu_xxx;exp=123;proxy-ep=https://proxy.xxx.com;...
        """
        match = re.search(r'(?:^|;)\s*proxy-ep=([^;\s]+)', token, re.IGNORECASE)
        
        if not match:
            return self.DEFAULT_BASE_URL
        
        proxy_ep = match.group(1).strip()
        
        # Converter proxy.* -> api.*
        host = proxy_ep.replace("https://", "").replace("http://", "")
        host = re.sub(r'^proxy\.', 'api.', host, flags=re.IGNORECASE)
        
        return f"https://{host}"
    
    def _is_token_valid(self, token: CopilotToken) -> bool:
        """Verificar se token ainda é válido (margem de 5 minutos)"""
        return token.expires_at > datetime.now() + timedelta(minutes=5)
    
    def get_usage_info(self) -> Dict:
        """
        Obter informações de uso e quota
        
        Returns:
            {
                "plan": str,
                "premium_remaining": float,
                "chat_remaining": float
            }
        """
        response = requests.get(
            self.COPILOT_USER_URL,
            headers={
                "Authorization": f"token {self.github_token}",
                "Editor-Version": "vscode/1.96.2",
                "User-Agent": "GitHubCopilotChat/0.26.7",
                "X-Github-Api-Version": "2025-04-01",
            }
        )
        response.raise_for_status()
        
        data = response.json()
        
        return {
            "plan": data.get("copilot_plan", "unknown"),
            "premium_remaining": data.get("quota_snapshots", {})
                .get("premium_interactions", {})
                .get("percent_remaining", 0),
            "chat_remaining": data.get("quota_snapshots", {})
                .get("chat", {})
                .get("percent_remaining", 0),
        }
    
    def list_available_models(self) -> List[str]:
        """
        Lista de modelos conhecidos do Copilot
        
        Nota: Copilot não tem endpoint de lista. 
        Esta é uma lista estática de modelos conhecidos.
        A disponibilidade real depende do plano do usuário.
        """
        return [
            "gpt-4o",
            "gpt-4.1",
            "gpt-4.1-mini",
            "gpt-4.1-nano",
            "o1",
            "o1-mini",
            "o3-mini",
            "claude-3.7-sonnet",
            "gemini-2.0-flash",
        ]
    
    def chat_completion(
        self,
        model: str,
        messages: List[Dict[str, str]],
        temperature: float = 0.7,
        max_tokens: int = 8192,
    ) -> Dict:
        """
        Fazer requisição de chat completion
        
        Args:
            model: Nome do modelo (ex: "gpt-4o")
            messages: Lista de mensagens [{"role": "user", "content": "..."}]
            temperature: Temperatura de geração
            max_tokens: Máximo de tokens
        
        Returns:
            Resposta da API no formato OpenAI
        """
        copilot_token = self.exchange_for_copilot_token()
        
        url = f"{copilot_token.base_url}/chat/completions"
        
        payload = {
            "model": model,
            "messages": messages,
            "temperature": temperature,
            "max_tokens": max_tokens,
            "stream": False,
        }
        
        response = requests.post(
            url,
            json=payload,
            headers={
                "Authorization": f"Bearer {copilot_token.token}",
                "Content-Type": "application/json",
                "Accept": "application/json",
            }
        )
        
        response.raise_for_status()
        return response.json()
```

### 6.4 Modelos Django

#### `models.py`

```python
from django.db import models
from django.contrib.auth.models import User
from django.utils import timezone
from datetime import timedelta

class CopilotAuthProfile(models.Model):
    """Perfil de autenticação GitHub Copilot"""
    
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='copilot_profiles')
    profile_id = models.CharField(max_length=255, unique=True)
    github_access_token = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        db_table = 'copilot_auth_profiles'
        verbose_name = 'Copilot Auth Profile'
        verbose_name_plural = 'Copilot Auth Profiles'
    
    def __str__(self):
        return f"{self.user.username} - {self.profile_id}"

class CopilotTokenCache(models.Model):
    """Cache de tokens Copilot"""
    
    profile = models.OneToOneField(
        CopilotAuthProfile,
        on_delete=models.CASCADE,
        related_name='token_cache'
    )
    copilot_token = models.TextField()
    base_url = models.URLField(max_length=500)
    expires_at = models.DateTimeField()
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        db_table = 'copilot_token_cache'
        verbose_name = 'Copilot Token Cache'
    
    def is_valid(self) -> bool:
        """Verificar se token ainda é válido (margem de 5 minutos)"""
        return self.expires_at > timezone.now() + timedelta(minutes=5)
    
    def __str__(self):
        return f"Token for {self.profile.profile_id}"

class CopilotUsageLog(models.Model):
    """Log de uso dos modelos Copilot"""
    
    profile = models.ForeignKey(
        CopilotAuthProfile,
        on_delete=models.CASCADE,
        related_name='usage_logs'
    )
    model_name = models.CharField(max_length=100)
    prompt_tokens = models.IntegerField(default=0)
    completion_tokens = models.IntegerField(default=0)
    total_tokens = models.IntegerField(default=0)
    request_time = models.DateTimeField(auto_now_add=True)
    response_time_ms = models.IntegerField(null=True, blank=True)
    success = models.BooleanField(default=True)
    error_message = models.TextField(null=True, blank=True)
    
    class Meta:
        db_table = 'copilot_usage_logs'
        verbose_name = 'Copilot Usage Log'
        ordering = ['-request_time']
    
    def __str__(self):
        return f"{self.model_name} - {self.request_time}"
```

### 6.5 Views Django

#### `views.py`

```python
from django.http import JsonResponse
from django.views.decorators.http import require_http_methods
from django.views.decorators.csrf import csrf_exempt
from django.contrib.auth.decorators import login_required
from .oauth import GitHubDeviceFlow
from .copilot_api import CopilotAPIClient
from .models import CopilotAuthProfile, CopilotTokenCache, CopilotUsageLog
import json
import time

@require_http_methods(["POST"])
@login_required
def start_oauth_flow(request):
    """
    Iniciar fluxo OAuth Device Flow
    
    POST /copilot/auth/start
    
    Returns:
        {
            "user_code": "WDJB-MJHT",
            "verification_uri": "https://github.com/login/device",
            "device_code": "...",
            "expires_in": 900,
            "interval": 5
        }
    """
    try:
        flow = GitHubDeviceFlow()
        device_info = flow.request_device_code()
        
        # Armazenar device_code em sessão para polling posterior
        request.session['device_code'] = device_info['device_code']
        request.session['expires_in'] = device_info['expires_in']
        request.session['interval'] = device_info['interval']
        request.session['started_at'] = time.time()
        
        return JsonResponse({
            "user_code": device_info["user_code"],
            "verification_uri": device_info["verification_uri"],
            "expires_in": device_info["expires_in"],
            "interval": device_info["interval"],
        })
    
    except Exception as e:
        return JsonResponse({
            "error": str(e)
        }, status=500)

@require_http_methods(["POST"])
@login_required
def poll_oauth_token(request):
    """
    Fazer polling do token OAuth
    
    POST /copilot/auth/poll
    
    Returns:
        {
            "status": "pending" | "authorized" | "expired" | "denied",
            "access_token": "..." (se authorized)
        }
    """
    try:
        device_code = request.session.get('device_code')
        started_at = request.session.get('started_at')
        expires_in = request.session.get('expires_in', 900)
        interval = request.session.get('interval', 5)
        
        if not device_code or not started_at:
            return JsonResponse({
                "error": "OAuth flow não iniciado"
            }, status=400)
        
        # Verificar expiração
        if time.time() - started_at > expires_in:
            return JsonResponse({
                "status": "expired"
            })
        
        # Tentar obter token
        flow = GitHubDeviceFlow()
        
        try:
            access_token = flow.poll_for_access_token(
                device_code=device_code,
                interval=interval,
                expires_in=1  # Apenas uma tentativa
            )
            
            # Salvar perfil
            profile_id = f"github-copilot:{request.user.username}"
            profile, created = CopilotAuthProfile.objects.update_or_create(
                user=request.user,
                profile_id=profile_id,
                defaults={
                    'github_access_token': access_token,
                }
            )
            
            # Limpar sessão
            del request.session['device_code']
            del request.session['started_at']
            del request.session['expires_in']
            del request.session['interval']
            
            return JsonResponse({
                "status": "authorized",
                "access_token": access_token,
                "profile_id": profile_id,
            })
        
        except TimeoutError:
            return JsonResponse({
                "status": "pending"
            })
        
        except ValueError as e:
            if "negou acesso" in str(e):
                return JsonResponse({
                    "status": "denied"
                })
            raise
    
    except Exception as e:
        return JsonResponse({
            "error": str(e)
        }, status=500)

@require_http_methods(["GET"])
@login_required
def get_copilot_models(request):
    """
    Listar modelos disponíveis
    
    GET /copilot/models
    
    Returns:
        {
            "models": ["gpt-4o", "gpt-4.1", ...]
        }
    """
    try:
        profile = CopilotAuthProfile.objects.filter(user=request.user).first()
        
        if not profile:
            return JsonResponse({
                "error": "Usuário não autenticado com Copilot"
            }, status=401)
        
        client = CopilotAPIClient(profile.github_access_token)
        models = client.list_available_models()
        
        return JsonResponse({
            "models": models
        })
    
    except Exception as e:
        return JsonResponse({
            "error": str(e)
        }, status=500)

@require_http_methods(["GET"])
@login_required
def get_usage_info(request):
    """
    Obter informações de uso
    
    GET /copilot/usage
    
    Returns:
        {
            "plan": "individual",
            "premium_remaining": 75.5,
            "chat_remaining": 92.3
        }
    """
    try:
        profile = CopilotAuthProfile.objects.filter(user=request.user).first()
        
        if not profile:
            return JsonResponse({
                "error": "Usuário não autenticado com Copilot"
            }, status=401)
        
        client = CopilotAPIClient(profile.github_access_token)
        usage_info = client.get_usage_info()
        
        return JsonResponse(usage_info)
    
    except Exception as e:
        return JsonResponse({
            "error": str(e)
        }, status=500)

@csrf_exempt
@require_http_methods(["POST"])
@login_required
def chat_completion(request):
    """
    Fazer requisição de chat completion
    
    POST /copilot/chat/completions
    Body:
        {
            "model": "gpt-4o",
            "messages": [
                {"role": "user", "content": "Hello"}
            ],
            "temperature": 0.7,
            "max_tokens": 8192
        }
    
    Returns:
        Resposta OpenAI-compatible
    """
    try:
        profile = CopilotAuthProfile.objects.filter(user=request.user).first()
        
        if not profile:
            return JsonResponse({
                "error": "Usuário não autenticado com Copilot"
            }, status=401)
        
        data = json.loads(request.body)
        model = data.get("model", "gpt-4o")
        messages = data.get("messages", [])
        temperature = data.get("temperature", 0.7)
        max_tokens = data.get("max_tokens", 8192)
        
        if not messages:
            return JsonResponse({
                "error": "Mensagens não fornecidas"
            }, status=400)
        
        client = CopilotAPIClient(profile.github_access_token)
        
        start_time = time.time()
        
        try:
            response = client.chat_completion(
                model=model,
                messages=messages,
                temperature=temperature,
                max_tokens=max_tokens,
            )
            
            response_time = int((time.time() - start_time) * 1000)
            
            # Registrar uso
            usage = response.get("usage", {})
            CopilotUsageLog.objects.create(
                profile=profile,
                model_name=model,
                prompt_tokens=usage.get("prompt_tokens", 0),
                completion_tokens=usage.get("completion_tokens", 0),
                total_tokens=usage.get("total_tokens", 0),
                response_time_ms=response_time,
                success=True,
            )
            
            return JsonResponse(response)
        
        except Exception as e:
            response_time = int((time.time() - start_time) * 1000)
            
            # Registrar erro
            CopilotUsageLog.objects.create(
                profile=profile,
                model_name=model,
                success=False,
                error_message=str(e),
                response_time_ms=response_time,
            )
            
            raise
    
    except json.JSONDecodeError:
        return JsonResponse({
            "error": "JSON inválido"
        }, status=400)
    
    except Exception as e:
        return JsonResponse({
            "error": str(e)
        }, status=500)
```

### 6.6 URLs

#### `urls.py`

```python
from django.urls import path
from . import views

app_name = 'copilot'

urlpatterns = [
    # OAuth
    path('auth/start', views.start_oauth_flow, name='auth_start'),
    path('auth/poll', views.poll_oauth_token, name='auth_poll'),
    
    # Informações
    path('models', views.get_copilot_models, name='models'),
    path('usage', views.get_usage_info, name='usage'),
    
    # API
    path('chat/completions', views.chat_completion, name='chat_completion'),
]
```

### 6.7 Exemplo de Uso no Frontend

#### JavaScript para fluxo OAuth

```javascript
// Iniciar OAuth
async function startCopilotAuth() {
    const response = await fetch('/copilot/auth/start', {
        method: 'POST',
        headers: {
            'X-CSRFToken': getCsrfToken(),
        },
    });
    
    const data = await response.json();
    
    // Exibir código para usuário
    document.getElementById('user-code').textContent = data.user_code;
    document.getElementById('verification-link').href = data.verification_uri;
    document.getElementById('verification-link').textContent = data.verification_uri;
    
    // Iniciar polling
    pollForToken(data.interval);
}

// Polling para verificar autorização
async function pollForToken(interval) {
    const maxAttempts = 120; // 10 minutos com intervalo de 5s
    let attempts = 0;
    
    const poll = async () => {
        if (attempts >= maxAttempts) {
            alert('Timeout: autorização não concluída');
            return;
        }
        
        const response = await fetch('/copilot/auth/poll', {
            method: 'POST',
            headers: {
                'X-CSRFToken': getCsrfToken(),
            },
        });
        
        const data = await response.json();
        
        if (data.status === 'authorized') {
            alert('Autorizado com sucesso!');
            location.reload();
        } else if (data.status === 'pending') {
            attempts++;
            setTimeout(poll, interval * 1000);
        } else if (data.status === 'expired') {
            alert('Código expirado. Tente novamente.');
        } else if (data.status === 'denied') {
            alert('Acesso negado pelo usuário.');
        }
    };
    
    poll();
}

// Fazer chat completion
async function chatWithCopilot(message) {
    const response = await fetch('/copilot/chat/completions', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRFToken': getCsrfToken(),
        },
        body: JSON.stringify({
            model: 'gpt-4o',
            messages: [
                { role: 'user', content: message }
            ],
            temperature: 0.7,
            max_tokens: 2048,
        }),
    });
    
    const data = await response.json();
    
    if (data.error) {
        console.error('Erro:', data.error);
        return null;
    }
    
    return data.choices[0].message.content;
}

// Helper CSRF
function getCsrfToken() {
    return document.querySelector('[name=csrfmiddlewaretoken]').value;
}
```

### 6.8 Configuração do Django

#### `settings.py` - Adicionar ao projeto

```python
INSTALLED_APPS = [
    # ...
    'copilot_integration',
]

# Sessions (necessário para OAuth polling)
SESSION_ENGINE = 'django.contrib.sessions.backends.db'
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SECURE = True  # Em produção com HTTPS
```

### 6.9 Migrações

```bash
python manage.py makemigrations copilot_integration
python manage.py migrate
```

---

## 7. Conclusões e Recomendações

### 7.1 Pontos-Chave da Implementação

1. **OAuth Device Flow**: Permite autenticação sem navegador no mesmo dispositivo
2. **Troca de Token**: GitHub token → Copilot token com endpoint específico
3. **Cache**: Tokens Copilot são cacheados e reutilizados até expiração
4. **Descoberta de Modelos**: Lista estática; disponibilidade depende do plano
5. **Rastreamento**: API específica para verificar uso e limites

### 7.2 Vantagens da Abordagem

- ✅ Sem necessidade de API key separada
- ✅ Usa a assinatura Copilot existente do usuário
- ✅ Tokens com expiração automática (segurança)
- ✅ Compatível com API OpenAI (fácil integração)
- ✅ Rastreamento de uso em tempo real

### 7.3 Limitações

- ⚠️ Requer assinatura ativa do GitHub Copilot
- ⚠️ Modelos disponíveis dependem do plano
- ⚠️ Quotas de uso por período
- ⚠️ Necessário renovar tokens periodicamente
- ⚠️ API interna (`copilot_internal`) pode mudar sem aviso

### 7.4 Segurança

1. **Armazenamento de Tokens**: Usar criptografia (ex: `django-cryptography`)
2. **HTTPS Obrigatório**: Em produção
3. **Rate Limiting**: Implementar para evitar abuso
4. **Logs de Auditoria**: Registrar todas as requisições
5. **Rotação de Tokens**: Implementar refresh automático

### 7.5 Próximos Passos

Para uma implementação completa em produção:

1. Adicionar autenticação JWT para API
2. Implementar WebSocket para streaming de respostas
3. Criar dashboard de monitoramento de uso
4. Adicionar suporte a múltiplos perfis por usuário
5. Implementar cache Redis para tokens
6. Criar testes unitários e de integração
7. Documentar API com Swagger/OpenAPI

---

## Referências

- [RFC 8628 - OAuth 2.0 Device Authorization Grant](https://datatracker.ietf.org/doc/html/rfc8628)
- [GitHub OAuth Apps](https://docs.github.com/en/developers/apps/building-oauth-apps)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)
- [Django Documentation](https://docs.djangoproject.com/)

---

**Documento criado por**: Análise do código OpenClaw  
**Data**: 11 de Fevereiro de 2026  
**Versão**: 1.0
