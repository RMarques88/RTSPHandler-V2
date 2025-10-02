# 🎥 RTSP Handler V2 - Sistema de Streaming Enterprise

> **✨ PROJETO CONSOLIDADO E MODERNIZADO (2024)**  
> Arquitetura MVC completamente refatorada, frontend unificado, Docker otimizado

[![Security](https://img.shields.io/badge/Security-Enterprise-green)](https://github.com)
[![Authentication](https://img.shields.io/badge/Auth-bcrypt-blue)](https://github.com)
[![Go](https://img.shields.io/badge/Go-1.23+-blue)](https://golang.org)
[![PHP](https://img.shields.io/badge/PHP-8.2+-purple)](https://php.net)
[![Docker](https://img.shields.io/badge/Docker-Ready-blue)](https://docker.com)
[![Nginx](https://img.shields.io/badge/Nginx-Optimized-green)](https://nginx.org)

## 🚀 INÍCIO RÁPIDO

### 1. Construir e Executar
```bash
# Parar containers existentes (se houver)
docker-compose down

# Construir e executar
docker-compose up --build

# Ou usar a task do VS Code (opcional)
# Ctrl+Shift+P -> "Tasks: Run Task" -> "Rebuild and Run RTSP Server"
```

### 2. Acessar a Aplicação
- **URL**: http://localhost (porta 80)
- **Login**: `admin`
- **Senha**: `******`

### 3. Verificar Sistema
O sistema está configurado para execução em Linux. Para testes locais de desenvolvimento, utilize Docker.

## 📁 ESTRUTURA DE DADOS CENTRALIZADA

O sistema utiliza uma estrutura de dados centralizada para facilitar backup e recovery:

```
/app/data/
├── logs/
│   ├── nginx/          # Logs do Nginx
│   ├── php/            # Logs do PHP-FPM
│   ├── web/            # Logs da aplicação web
│   ├── rtsp/           # Logs do servidor RTSP
│   ├── mysql/          # Logs do MySQL
│   └── supervisor/     # Logs do Supervisor
├── recordings/         # Gravações de streams
├── thumbnails/         # Thumbnails das câmeras
├── backups/           # Backups automáticos
└── version.json       # Informações da versão atual
```

## 🏗️ ARQUITETURA DO SISTEMA

### Backend Go (RTSP Server)
- **Arquivo principal**: `RTSPtoWeb.go`
- **Configuração**: Totalmente via banco de dados MySQL
- **Protocolos**: HLS, MSE, WebRTC
- **Autenticação**: Sistema duplo de tokens integrado

### Frontend PHP MVC
```
web/
├── 🎨 public/
├── 📱 app/
│   ├── Core/           # Sistema principal MVC
│   │   ├── App.php
│   │   ├── Auth.php
│   │   ├── Controller.php
│   │   ├── Model.php
│   │   ├── Router.php
│   │   ├── Utils.php
│   │   └── PermissionManager.php  # Sistema de permissões híbrido
│   ├── Controllers/    # Controladores da aplicação
│   │   ├── AuthController.php
│   │   ├── DashboardController.php
│   │   ├── StreamController.php
│   │   ├── UserController.php
│   │   ├── SecurityController.php
│   │   └── CameraHealthController.php
│   ├── Models/         # Modelos de dados
│   │   ├── Database.php
│   │   ├── User.php
│   │   ├── Stream.php
│   │   ├── Logger.php
│   │   └── CameraHealth.php
│   ├── Views/          # Templates da interface
│   │   ├── layouts/    # Layouts base
│   │   ├── dashboard/  # Dashboard principal
│   │   ├── streams/    # Gerenciamento de streams
│   │   ├── auth/       # Login/autenticação
│   │   └── users/      # Gerenciamento de usuários
│   └── Middleware/     # Middlewares
│       ├── StreamPermissions.php
│       └── StreamProxy.php
├── 📁 public/thumbnails/ # Thumbnails como arquivos
└── bootstrap_mvc.php    # Bootstrap do sistema MVC
```

## 🔐 SISTEMA DE AUTENTICAÇÃO

### Duplo Token System
1. **Session Token**: Autenticação temporária para acesso à interface
2. **Watermark Token**: Token permanente para auditoria e rastreamento

### Sistema de Permissões Híbrido
O `PermissionManager.php` implementa um sistema flexível:

```php
// Formato: DEPT:vendas|UUID:abc123|LEVEL:admin
$permissions = "DEPT:vendas,ti|UUID:stream-uuid-1,stream-uuid-2|LEVEL:operator";
```

**Tipos de Permissão**:
- **DEPT**: Por departamento (vendas, ti, admin, etc.)
- **UUID**: Por stream específico (UUID do stream)
- **LEVEL**: Por nível de acesso (admin, operator, viewer)

## 🔑 FLUXO DE AUTENTICAÇÃO E WATERMARK

### Como funciona
- **Token de sessão (temporário):**
  - Usado para autenticação do usuário e autorização de acesso a todas as rotas protegidas (streaming, gravações, etc).
  - Validado sempre via banco de dados, retornando o userID autenticado.
  - Permissões de acesso a streams são verificadas centralizadamente.
- **Token de watermark (permanente):**
  - Obtido sempre via userID autenticado (nunca do token de sessão).
  - Usado apenas para marcação/auditoria, nunca para autenticação ou autorização.
  - Enviado automaticamente no header `X-Watermark-Token` em todas as rotas de streaming (MSE, HLS, WebRTC, HLSLL, SaveMP4).
  - O backend Go e o frontend PHP garantem que o watermark correto do usuário autenticado seja sempre utilizado.

### Segurança e Centralização
- Toda lógica de permissão e obtenção de watermark está centralizada no backend Go.
- O frontend PHP busca o watermark correto do usuário no banco e encaminha no header ao backend Go.
- Não há duplicidade de funções de validação ou permissão.
- O token de watermark nunca é aceito para autenticação/autorização.

## 📚 EXEMPLOS DE USO DAS PRINCIPAIS APIS

### 1. Autenticação (Login)

**Request:**
```http
POST /api/auth/login
Content-Type: application/json
{
  "username": "admin",
  "password": "*****"
}
```
**Response:**
```json
{
  "status": 1,
  "payload": {
    "token": "SESSION_TOKEN",
    "message": "Login realizado com sucesso",
    "success": true,
    "expires_at": "2025-06-24T18:00:00Z"
  }
}
```

### 2. Streaming (HLS, MSE, WebRTC, etc.)

**Request:**
```http
GET /stream/{uuid}/channel/{channel}/hls/live/index.m3u8?token=SESSION_TOKEN
X-Watermark-Token: WM_xxxxyyyyzzzz
```
**Response:**
- Playlist HLS ou stream de vídeo, com header `X-Watermark-Token` presente na resposta.

### 3. Listar Gravações

**Request:**
```http
GET /api/recordings?uuid={uuid}&channel={channel}&token=SESSION_TOKEN
```
**Response:**
```json
{
  "success": true,
  "recordings": [ ... ],
  "total_count": 12,
  "start_date": "2025-06-01",
  "end_date": "2025-06-24"
}
```

### 4. Auditoria de Watermark

**Request:**
```http
GET /api/watermark/audit
Authorization: Bearer SESSION_TOKEN
```
**Response:**
```json
[
  {
    "user_id": 1,
    "username": "admin",
    "watermark_token": "WM_xxxxyyyyzzzz",
    "stream_uuid": "...",
    "channel_id": "0",
    "ip_address": "192.168.0.10",
    "accessed_at": "2025-06-24T17:00:00Z"
  },
  ...
]
```

## 🔎 FLUXO DE AUDITORIA E INTEGRAÇÃO FRONTEND-BACKEND

- O frontend PHP SEMPRE obtém o watermark correto do usuário autenticado via model User (`getWatermarkToken($userId)`).
- Toda requisição de streaming ou gravação encaminha o header `X-Watermark-Token` para o backend Go.
- O backend Go valida o token de sessão, verifica permissão e associa o watermark correto ao userID autenticado.
- Toda ação sensível (stream, gravação, download) é registrada para auditoria, incluindo userID, IP, watermark e timestamp.
- Logs de auditoria podem ser consultados via API (`/api/watermark/audit`).

## 🗄️ BANCO DE DADOS

### Estrutura Principal
- **streams**: Configurações dos streams RTSP
- **users**: Usuários do sistema com permissões
- **stream_access_log**: Log de acessos para auditoria
- **session_tokens**: Tokens de sessão temporários
- **watermark_tokens**: Tokens permanentes para auditoria

### Configuração
Toda configuração é armazenada no banco MySQL. Não há mais arquivos `config.json`.

## 🛠️ DESENVOLVIMENTO

### Para Desenvolvimento Windows
Use o script `deploy.ps1`:

```powershell
# Deploy local
.\deploy.ps1 local

# Homologação
.\deploy.ps1 homolog "Descrição das mudanças"

# Release para produção
.\deploy.ps1 release "Descrição da versão"

# Status do projeto
.\deploy.ps1 status

# Ver logs
.\deploy.ps1 logs
```

### Scripts Disponíveis
```
scripts/
├── init-rtsp-server.sh      # Inicialização completa do container
├── auto-update.sh           # Auto-atualização via Git (Linux)
├── version-manager.sh       # Gerenciamento de versões
├── git-health-check.sh      # Verificação de conectividade Git
├── backup-data.sh           # Backup de dados
├── restore-data.sh          # Restore de dados
├── docker-healthcheck.sh    # Health check completo
└── wait-for-mysql.sh        # Aguardar MySQL estar pronto
```

## 🔄 AUTO-ATUALIZAÇÃO

### Sistema Automático (Linux)
- **Frequência**: Diariamente às 2h da manhã
- **Processo**: Backup → Download → Build → Deploy → Health Check
- **Rollback**: Automático em caso de falha
- **Logs**: `/var/log/rtsp-update.log`

### Health Check Integrado
- ✅ Nginx funcionando
- ✅ PHP-FPM ativo
- ✅ MySQL conectável
- ✅ RTSP Server rodando
- ✅ Conectividade Git
- ✅ Arquivos críticos presentes

## 📦 DOCKER

### Containers
- **rtsp-server**: Aplicação principal (Nginx + PHP + MySQL + Go)
- **Volumes**: `/app/data` para persistência de dados
- **Portas**:
  - `80`: Interface web
  - `8083`: API RTSP
  - `9000-12000`: HTTP streaming
  - `20000-23000`: RTSP
  - `30000-33000/udp`: WebRTC

### Build
```bash
# Build completo
docker-compose build --no-cache

# Logs em tempo real
docker-compose logs -f

# Health check manual
docker exec rtsp-container /usr/local/bin/docker-healthcheck.sh
```

## 🔧 CONFIGURAÇÃO DE STREAMS

### Adicionar Novo Stream
1. Acesse o dashboard
2. Vá para "Gerenciar Streams"
3. Clique em "Adicionar Stream"
4. Configure:
   - **Nome**: Nome descritivo
   - **URL RTSP**: rtsp://ip:porta/path
   - **Usuário/Senha**: Credenciais da câmera
   - **Permissões**: Quem pode acessar

### Tipos de Stream Suportados
- **RTSP over TCP**
- **RTSP over UDP**
- **HTTP streaming (HLS)**
- **WebRTC para baixa latência**
- **MSE para navegadores modernos**

## 🔍 MONITORAMENTO

### Dashboard Principal
- Status em tempo real de todas as câmeras
- Métricas de performance
- Log de acessos e auditoria
- Gerenciamento de usuários e permissões

### Logs Centralizados
```bash
# Logs da aplicação web
tail -f /app/data/logs/web/app.log

# Logs do RTSP server
tail -f /app/data/logs/rtsp/server.log

# Logs de auto-atualização
tail -f /app/data/logs/web/git-health.log
```

## 🛡️ SEGURANÇA

### Funcionalidades de Segurança
- **Autenticação bcrypt** para senhas
- **Tokens JWT** para sessões
- **Watermark tokens** para auditoria
- **CSRF protection** em formulários
- **Rate limiting** para APIs
- **Logs de auditoria** completos

### Permissões Granulares
- Controle por departamento
- Controle por stream específico
- Níveis de acesso hierárquicos
- Log completo de acessos

## 📋 TROUBLESHOOTING

### Problemas Comuns

#### Container não inicia
```bash
# Verificar logs
docker-compose logs

# Health check manual
docker exec container-name /usr/local/bin/docker-healthcheck.sh
```

#### Stream não carrega
1. Verificar conectividade com a câmera
2. Validar credenciais RTSP
3. Verificar logs do Go server
4. Testar stream diretamente

#### Erro de autenticação
1. Verificar usuário/senha no banco
2. Limpar cookies do navegador
3. Verificar logs de PHP
4. Validar tokens de sessão

### Comandos Úteis
```bash
# Restart serviços
docker-compose restart

# Rebuild completo
docker-compose down && docker-compose up --build

# Backup manual
./scripts/backup-data.sh

# Verificar versão
cat /app/data/version.json
```

## 🔗 LINKS ÚTEIS

### Documentação Adicional
- [`CONSOLIDACAO.md`]: Documentação da arquitetura consolidada
- [`DESENVOLVIMENTO.md`]: Guia completo de desenvolvimento
- [`setup-git.sh`]: Setup inicial do repositório

### APIs e Endpoints
- **API Health**: http://localhost/api/health
- **Stream API**: http://localhost:8083/
- **WebSocket**: ws://localhost:8083/ws

---

## 📞 SUPORTE

Para questões técnicas, consulte os logs em `/app/data/logs/` e a documentação em [`CONSOLIDACAO.md`](CONSOLIDACAO.md).

**Sistema desenvolvido para ambiente enterprise com foco em segurança, auditoria e escalabilidade.**

## ❗ Observação Importante sobre Sessão Única em Playback de Gravação

O controle de **sessão única** (permitir apenas uma sessão de playback por gravação/canal/usuário) se aplica **exclusivamente** ao endpoint `/api/recordings/play` (playback de gravação Intelbras). Os demais endpoints de streaming ao vivo (HLS, MSE, WebRTC, etc.) **não** possuem essa limitação e permitem múltiplas sessões simultâneas normalmente. O objetivo é evitar sobrecarga de recursos com transcodificação.

- **Sessão única:** Apenas para `/api/recordings/play` (playback de gravação)
- **Streaming ao vivo:** Sem limitação de sessões simultâneas

## 📚 Exemplos de Uso dos Endpoints de Gravação Intelbras

### 1. Listar Gravações

**Request:**
```http
GET /api/recordings/list?uuid={stream_uuid}&channel={channel_id}&start_date={YYYY-MM-DD}&end_date={YYYY-MM-DD}&token={SESSION_TOKEN}
X-Watermark-Token: WM_xxxxyyyyzzzz
```
**Response:**
```json
{
  "success": true,
  "recordings": [
    {
      "file_name": "20241201_143022.dav",
      "channel": 1,
      "start_time": "2024-12-01T14:30:22Z",
      "end_time": "2024-12-01T14:35:18Z",
      "duration": 296,
      "file_size": 52428800,
      "file_type": "dav",
      "is_motion": false,
      "is_alarm": false
    }
  ],
  "total_count": 1,
  "start_date": "2024-12-01",
  "end_date": "2024-12-01"
}
```

### 2. Iniciar Playback de Gravação (Sessão Única)

**Request:**
```http
POST /api/recordings/play?uuid={stream_uuid}&channel={channel_id}&file={file_name}&token={SESSION_TOKEN}
X-Watermark-Token: WM_xxxxyyyyzzzz
```
**Response:**
```json
{
  "success": true,
  "session_id": "abc123def456",
  "status": "active",
  "stream_url": "/api/recordings/stream?session_id=abc123def456&token={SESSION_TOKEN}",
  "hls_url": "/api/recordings/hls?session_id=abc123def456&token={SESSION_TOKEN}",
  "mse_url": "/api/recordings/mse?session_id=abc123def456&token={SESSION_TOKEN}"
}
```

- Se já existir uma sessão active para o mesmo usuário/arquivo/canal, será retornado erro:
```json
{
  "success": false,
  "error": "Sessão já ativa para este arquivo/canal/usuário. Aguarde ou encerre a sessão existente."
}
```

### 3. Download de Gravação

**Request:**
```http
GET /api/recordings/download?uuid={stream_uuid}&channel={channel_id}&file={file_name}&token={SESSION_TOKEN}
X-Watermark-Token: WM_xxxxyyyyzzzz
```
**Response:**
```json
{
  "success": true,
  "download_url": "http://192.168.1.100/cgi-bin/RPC_Loadfile/20241201_143022.dav",
  "file_name": "20241201_143022.dav",
  "device_ip": "192.168.1.100",
  "note": "Use as credenciais do dispositivo para autenticação"
}
```

### 4. Status da Sessão de Playback

**Request:**
```http
GET /api/recordings/status?session_id={session_id}&token={SESSION_TOKEN}
```
**Response:**
```json
{
  "session_id": "abc123def456",
  "status": "active",
  "ready": true,
  "client_count": 1,
  "last_activity": "2024-12-01T15:30:22Z",
  "file_name": "20241201_143022.dav",
  "stream_uuid": "stream-uuid",
  "channel_id": "1"
}
```

### 5. Encerrar Sessão de Playback

**Request:**
```http
DELETE /api/recordings/stop?session_id={session_id}&token={SESSION_TOKEN}
```

## 📦 SISTEMA CDN/PROXY (CACHE INFINITO)

> **NOVO**: Sistema avançado de cache para snapshots, thumbnails e arquivos estáticos com persistência configurável.

### 🚀 Características Principais
- **Cache Infinito**: Snapshots nunca expiram, apenas são atualizados oportunisticamente
- **Geração Automática**: Thumbnails gerados automaticamente em múltiplos tamanhos
- **Configuração Persistente**: Configurações salvas em arquivo JSON
- **APIs Administrativas**: Controle completo via REST APIs
- **Cache Inteligente**: Atualização apenas quando stream está ativo

### 📊 Funcionalidades
1. **Snapshots**: Cache infinito com atualização oportunística
2. **Thumbnails**: Geração automática em tamanhos small/medium/large
3. **Arquivos Estáticos**: Cache de CSS, JS, imagens do frontend
4. **Estatísticas**: Monitoramento de uso de disco e performance
5. **Configuração Dinâmica**: Alterações via API persistidas em arquivo

### 🔧 Configuração
```json
{
  "cache_dir": "./cache",
  "max_disk_space_gb": 10.0,
  "compression_enabled": true,
  "serve_cache_control": "public, max-age=31536000",
  "enable_static_cache": true,
  "thumbnail_sizes": [
    {"name": "small", "width": 160, "height": 120},
    {"name": "medium", "width": 320, "height": 240},
    {"name": "large", "width": 640, "height": 480}
  ]
}
```

### 📡 APIs do CDN

#### Status e Estatísticas
```http
GET /api/cdn/status        # Status geral do CDN
GET /api/cdn/stats         # Estatísticas detalhadas
```

#### Gerenciamento de Cache
```http
POST /api/cdn/invalidate   # Invalidar cache específico
POST /api/cdn/cleanup      # Limpeza do cache
GET /api/cdn/cache/list    # Listar arquivos em cache
```

#### Configuração (Superadmin)
```http
GET /api/cdn/config        # Obter configuração atual
PUT /api/cdn/config        # Atualizar configuração
POST /api/cdn/config/reset # Resetar para padrão
```

#### Acesso aos Recursos
```http
GET /snapshot/{uuid}/{channel}              # Snapshot via CDN
GET /snapshot/{uuid}/{channel}/thumb/{size} # Thumbnail via CDN
GET /static/{path}                          # Arquivo estático via CDN
```

### 📈 Monitoramento
- **Uso de Disco**: Controle inteligente de espaço em disco
- **Performance**: Estatísticas de hits/miss de cache
- **Logs**: Rastreamento detalhado de todas as operações
- **Auditoria**: Integração com sistema de watermark

Para detalhes técnicos completos, consulte:
- [📖 Documentação Técnica CDN]
- [🛠️ Guia de Administração]

### 🎛️ Interface de Administração
O sistema CDN está preparado para integração com a interface web de administração:

1. **Dashboard CDN**: Estatísticas em tempo real
2. **Gerenciamento de Cache**: Invalidação por tipo/stream
3. **Configuração Dinâmica**: Alterações persistidas automaticamente
4. **Monitoramento**: Uso de disco, performance, logs
5. **Thumbnails**: Visualização e gerenciamento de tamanhos

**Endpoints prontos para integração:**
- Status em tempo real (`/api/cdn/status`)
- Configuração via formulário (`/api/cdn/config`)
- Invalidação seletiva (`/api/cdn/invalidate`)
- Listagem de arquivos (`/api/cdn/cache/list`)

####
