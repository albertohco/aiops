# Agente AIOps para Kubernetes

Sistema automatizado de monitoramento, diagnóstico e relatório para clusters Kubernetes. Detecta warnings, realiza investigação profunda com IA (Google Gemini) e cria issues no GitHub.

## 🎯 Características Principais

- **Monitoramento de Eventos**: Captura warnings do Kubernetes das últimas 60 minutos (todos os namespaces)
- **Deduplicação Automática**: Agrupa eventos similares e elimina duplicatas por padrão
- **Diagnóstico com IA**: Google Gemini analisa os problemas e investiga recursos afetados
- **Investigação Profunda**: Coleta logs, descreve recursos e identifica causas raiz
- **Relatórios Estruturados**: Gera diagnósticos em Markdown com severidade (CRÍTICO/ALTO/MÉDIO/BAIXO)
- **Criação de Issues**: Registra problemas automaticamente no GitHub com análise detalhada
- **Segurança**: API k8s-mcp fornece acesso read-only (apenas: kubectl_get, kubectl_describe, kubectl_logs)

## 🏗️ Arquitetura

```
┌─────────────────────────────────────────────────────────────┐
│                       n8n Workflow                          │
│  (aiops_albertohco.json - 326 linhas)                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [Manual Trigger] ──→ [HTTP Request]                       │
│                          ↓                                   │
│                   [Parse SSE Response]                      │
│                          ↓                                   │
│              [Filter Warnings (60 min)]                    │
│                          ↓                                   │
│              [Aggregate & Deduplicate]                      │
│                          ↓                                   │
│            [Group by Problem Pattern]                       │
│                          ↓                                   │
│        [AI Agent + Gemini Diagnostics] ←─ [MCP Client]     │
│                          ↓                                   │
│           [Create GitHub Issue]                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                              ↑
                         ┌────┴─────────┐
                         │              │
                    k8s-mcp        Kubernetes
                    (Port 3001)      Cluster
```

### Componentes

1. **n8n** (Orquestração)
   - Workflow principal: `n8n/aiops_albertohco.json`
   - HTTP Request para k8s-mcp
   - JavaScript parsing de SSE (Server-Sent Events)
   - Agregação e deduplicação de eventos
   - Integração com AI Agent

2. **k8s-mcp** (Kubernetes Access)
   - API JSON-RPC em `http://k8s-mcp:3001/mcp`
   - Ferramentas disponíveis (read-only):
     - `kubectl_get` - status dos recursos
     - `kubectl_describe` - detalhes e eventos
     - `kubectl_logs` - logs do container (com suporte a `previous`)

3. **Google Gemini** (Análise Inteligente)
   - Modelo: `models/gemini-2.5-flash-lite`
   - Investigação automática de problemas
   - Geração de recomendações

4. **Integrações**
   - GitHub: Criação de issues com análise

## 📋 Pré-requisitos

- **Docker & Docker Compose** - Para executar n8n e k8s-mcp
- **Cluster Kubernetes** - Acessível via `kubectl` com kubeconfig configurado
- **Google Cloud** - Conta com acesso à API Gemini
- **GitHub** - Repositório para issues e Personal Access Token

## ⚙️ Configuração e Instalação

### 1. Clone o Repositório

```bash
git clone https://github.com/albertohco/aiops.git
cd aiops
```

### 2. Prepare a Configuração do Kubernetes

Crie uma cópia do kubeconfig para Docker:

```bash
# Se estiver usando WSL/Docker Desktop
sed 's/127.0.0.1/host.docker.internal/g' ~/.kube/config > ~/.kube/config-docker

# Ou simplesmente:
cp ~/.kube/config ~/.kube/config-docker
```

Verifique que o arquivo está acessível:
```bash
ls -la ~/.kube/config-docker
```

### 3. Configure Variáveis de Ambiente

Crie `n8n/.env`:

```bash
# n8n Configuration
N8N_BASIC_AUTH_USER=seu_usuario
N8N_BASIC_AUTH_PASSWORD=sua_senha_segura
GENERIC_TIMEZONE=America/Sao_Paulo
TZ=America/Sao_Paulo

# Google Gemini API
GOOGLE_API_KEY=sua_chave_gemini

# GitHub
GITHUB_TOKEN=seu_personal_access_token
GITHUB_OWNER=seu_usuario_github
GITHUB_REPO=aiops


```

### 4. Defina Credenciais no n8n

1. Acesse n8n em `http://localhost:5678`
2. Faça login com as credenciais do `.env`
3. Vá para **Credentials**
4. Crie/atualize:
   - **Google Gemini API** - Cole sua chave GOOGLE_API_KEY
   - **GitHub** - Cole seu GITHUB_TOKEN

### 5. Inicie os Serviços

```bash
cd n8n
docker-compose up -d
```

Verifique o status:
```bash
docker-compose ps
```

Acesse n8n: `http://localhost:5678`

## 🚀 Uso

### Execução Manual

1. Abra a UI do n8n (`http://localhost:5678`)
2. Abra o workflow **aiops_albertohco**
3. Clique em **Execute Workflow**
4. Monitore a execução em tempo real

### Como Funciona

1. **Coleta de Eventos** (últimas 60 minutos)
   - HTTP Request chama k8s-mcp
   - Parse de resposta SSE
   - Filtra apenas `type: Warning`

2. **Deduplicação**
   - Agrupa eventos por padrão similar
   - Normaliza UIDs e nomes de pods
   - Conta ocorrências

3. **Diagnóstico com IA**
   - Gemini recebe lista de problemas agrupados
   - Usa ferramentas MCP para investigar
   - Chama `kubectl_describe` e `kubectl_logs`
   - Identifica causa raiz e recomendações

4. **Criação de Issue**
   - Gera título: `Erro no Kubernetes [timestamp]`
   - Body contém relatório completo em Markdown
   - Inclui severidade, namespace, pods afetados


## 📝 Exemplo de Relatório Gerado

```markdown
# Relatorio de Diagnostico Kubernetes

## Resumo
| Total | Criticos | Altos | Medios | Baixos |
|-------|----------|-------|--------|--------|
| 3     | 1        | 1     | 1      | 0      |

---

## Problema 1: ImagePullBackOff
- **Severidade:** CRITICO
- **Namespace:** default
- **Pods Afetados:** myapp-5d4c8f7b9-x2k8q

### Causa Raiz
Imagem Docker não existe ou nome está incorreto. A imagem `ngin:latest` deveria ser `nginx:latest`.

### Evidencias
- Pod descrito: `ImagePullBackOff` no container myapp
- Logs mostram: `pull access denied, repository does not exist`
- ConfigMap: Deployment usa imagem errada em spec.containers[0].image

### Solucao Recomendada
1. Corrigir o nome da imagem no Deployment
2. Alterar `ngin:latest` para `nginx:latest` ou `nginx:1.25`
3. Aplicar a alteração

### Comando Sugerido
```bash
kubectl set image deployment/myapp myapp=nginx:latest -n default
```
```

## 📂 Estrutura do Projeto

```
aiops/
├── n8n/
│   ├── aiops_albertohco.json     # Workflow principal
│   ├── docker-compose.yaml       # Orquestração de serviços
│   └── .env                      # Variáveis de ambiente (não versionado)
├── k8s/
│   └── nginx.yaml                # Exemplo de deployment para testes
├── prompt.md                     # Instruções detalhadas para IA
├── relatorio.md                  # Template de saída
├── package.json                  # Configuração Node.js
├── README.md                     # Este arquivo
└── .gitignore                    # Arquivos ignorados pelo Git
```

## 🔧 Troubleshooting

### Erro: "Connection refused" ao acessar n8n

```bash
# Verifique se o container está rodando
docker-compose ps

# Veja os logs
docker-compose logs n8n
```

### Erro: "Failed to pull kubernetes resources"

```bash
# Verifique a conectividade com kubeconfig
docker exec k8s-mcp kubectl get nodes

# Se usar WSL, verifique config-docker
cat ~/.kube/config-docker | grep server
```

### Erro: "Invalid Google API Key"

1. Vá para [Google Cloud Console](https://console.cloud.google.com)
2. Ative a API Gemini
3. Gere uma nova chave de API
4. Atualize em **Credentials** do n8n

### Erro: "GitHub token invalid"

1. Vá para [GitHub Settings > Tokens](https://github.com/settings/tokens)
2. Crie um novo token (Classic ou Fine-grained)
3. Permissões necessárias: `repo` (issues)
4. Atualize em **Credentials** do n8n

### Workflow não encontra eventos

1. Verifique se há warnings no cluster:
   ```bash
   kubectl get events -A --sort-by='.lastTimestamp' | grep Warning
   ```

2. Verifique o timestamp (eventos devem ter menos de 60 minutos)

3. Consulte logs do n8n:
   ```bash
   docker-compose logs -f n8n
   ```

## 🔐 Segurança

- ✅ **IA Read-Only**: Gemini não pode executar `kubectl patch`, `apply` ou `delete`
- ✅ **kubeconfig**: Isolado em arquivo separado, não versionado
- ✅ **Credenciais**: Salvas apenas no `.env` e em Credentials do n8n
- ⚠️ **N8N Acesso**: Protegido por Basic Auth (altere `N8N_BASIC_AUTH_PASSWORD`)
- ⚠️ **KUBECONFIG**: Armazene com permissões restritas (`chmod 600`)

## 📊 Métricas e Observabilidade

O workflow registra:
- Timestamp de execução
- Número de eventos capturados
- Número de problemas únicos identificados
- Status de cada investigação

Para histórico completo, acesse a aba **Execution** no n8n.

## 🤝 Contribuindo

Contribuições são bem-vindas! Por favor:

1. Crie um branch para sua feature: `git checkout -b feature/sua-feature`
2. Commit suas mudanças: `git commit -m 'Descrição clara'`
3. Push para o branch: `git push origin feature/sua-feature`
4. Abra um Pull Request

## 📄 Licença

MIT License - veja LICENSE para detalhes

## 📞 Suporte

Para problemas ou dúvidas:
1. Abra uma [issue no GitHub](https://github.com/albertohco/aiops/issues)
2. Verifique a seção Troubleshooting acima
3. Consulte os logs: `docker-compose logs n8n`

---

**Desenvolvido com ❤️ para monitoramento automatizado de Kubernetes**
