# Agente AIOps para Kubernetes

Este projeto é um agente AIOps automatizado, projetado para monitorar, diagnosticar e relatar problemas em um cluster Kubernetes. Ele utiliza o n8n para automação de fluxo de trabalho, um serviço customizado `k8s-mcp` para acesso seguro ao kubectl e um modelo de IA (Google Gemini) para análise inteligente.

## Funcionalidades

*   **Detecção Automatizada de Problemas:** Monitora eventos do Kubernetes em busca de avisos e alertas.
*   **Diagnóstico Inteligente:** Utiliza um agente de IA para analisar o estado do cluster, buscar logs relevantes e descrever recursos problemáticos.
*   **Relatórios Abrangentes:** Gera relatórios de diagnóstico detalhados em formato markdown.
*   **Notificação Proativa:** Cria issues no GitHub e envia notificações para o Discord para alertar a equipe imediatamente.
*   **Acesso Seguro ao `kubectl`:** Um serviço dedicado `k8s-mcp` fornece uma API controlada para executar comandos `kubectl`.

## Arquitetura

O sistema compreende os seguintes componentes chave:

*   **Workflow n8n (`n8n/agente-aiops.json`):** O orquestrador central que define a sequência de operações, processamento de dados, interações de IA e etapas de notificação.
*   **Configuração Docker Compose (`n8n/docker-compose.yaml`):** Gerencia o ambiente de execução, incluindo a instância n8n e o serviço customizado `k8s-mcp`.
*   **Serviço `k8s-mcp`:** Um serviço customizado que expõe uma API JSON-RPC para executar um conjunto predefinido de comandos `kubectl` de forma segura.
*   **Agente de IA (Google Gemini):** Analisa eventos do Kubernetes e dados de diagnóstico para gerar insights acionáveis e relatórios, guiado por instruções detalhadas em `prompt.md`.
*   **Integrações:**
    *   **Kubernetes:** Busca eventos do cluster e detalhes de recursos.
    *   **GitHub:** Cria issues para problemas detectados.
    *   **Discord:** Envia notificações sobre novos issues do GitHub.

## Pré-requisitos

Antes de executar este projeto, certifique-se de ter o seguinte instalado e configurado:

*   **Docker:** Para executar os serviços n8n e `k8s-mcp`.
*   **Cluster Kubernetes:** Acessível via `kubectl`. Seu arquivo `kubeconfig` deve estar configurado corretamente.
*   **Conta Google Cloud:** Com acesso à API Gemini e chaves de API apropriadas.
*   **Conta GitHub:** Com um repositório para criar issues e um token de acesso pessoal.
*   **Conta Discord:** Com uma URL de webhook ou token de bot para notificações.

## Configuração e Instalação

1.  **Clone o repositório:**
    ```bash
    git clone <url-do-repositorio>
    cd aiops
    ```

2.  **Configure o n8n:**
    *   Edite o arquivo `.env` no diretório `n8n/`.
    *   Defina `N8N_BASIC_AUTH_USER` e `N8N_BASIC_AUTH_PASSWORD` para acesso ao n8n.
    *   Configure as credenciais dentro da interface do n8n para Google Gemini, GitHub e Discord.

3.  **Configure o `k8s-mcp`:**
    *   Certifique-se de que seu arquivo `kubeconfig` seja acessível pelo serviço `k8s-mcp`.

4.  **Configure o Agente de IA (`prompt.md`):**
    *   Revise e ajuste as instruções do agente de IA em `prompt.md` conforme necessário.

5.  **Inicie os serviços:**
    ```bash
    cd n8n
    docker-compose up -d
    ```

## Uso

1.  **Acesse o n8n:** Abra seu navegador na UI do n8n (ex: `http://localhost:5678` se estiver executando localmente) usando as credenciais definidas no seu arquivo `.env`.
2.  **Dispare o Workflow:** Inicie manualmente o workflow `agente-aiops` para iniciar uma análise do cluster Kubernetes.
3.  **Monitore:** Observe a execução no n8n. Relatórios serão gerados como issues no GitHub e notificações serão enviadas para o Discord.

## Estrutura do Projeto

```
/
├── .gitignore
├── .nvmrc
├── package.json
├── prompt.md           # Instruções para o agente de IA
├── relatorio.md        # Exemplo de relatório de diagnóstico (ou template)
├── .gemini/            # Arquivos específicos do Gemini CLI
├── .git/               # Dados do repositório Git
├── k8s/                # Manifestos do Kubernetes (ex: para testes)
│   └── nginx.yaml
└── n8n/                # Arquivos relacionados ao n8n
    ├── .env            # Variáveis de ambiente e credenciais do n8n
    ├── agente-aiops.json # O workflow principal do n8n
    └── docker-compose.yaml # Configuração do Docker Compose para serviços
```

## Contribuições

Contribuições são bem-vindas! Por favor, siga os procedimentos padrão de branching e pull request do Git. Certifique-se de que todas as alterações sigam os padrões de código do projeto e que novas funcionalidades sejam acompanhadas por testes apropriados.

## Licença

[Licença MIT] (ou especifique sua licença)
