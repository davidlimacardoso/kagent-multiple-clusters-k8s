# KAgent Helm Chart

Este diretório contém o Helm chart principal do **kagent**, uma plataforma de agentes de IA para Kubernetes que utiliza o protocolo MCP (Model Context Protocol) para fornecer ferramentas e capacidades de automação.

## 📋 Visão Geral

O kagent é composto por:

- **Controller** - Gerencia os recursos customizados (CRDs) e orquestra os agentes
- **UI** - Interface web para interação com os agentes
- **MCP Servers** - Servidores que expõem ferramentas e capacidades via protocolo MCP
- **Agents** - Agentes especializados (k8s, observability, istio, etc.)

## 📁 Estrutura

```
kagent/
├── Chart.yaml              # Definição do chart e dependências
├── values.yaml             # Valores padrão de configuração
├── charts/                 # Sub-charts de agentes e ferramentas
│   ├── k8s-agent/
│   ├── observability-agent/
│   ├── istio-agent/
│   ├── grafana-mcp/
│   └── ...
├── clusters/               # Configurações específicas por cluster
│   ├── vrdev-cluster-a/    # Cluster com UI (principal)
│   └── vrdev-cluster-b/      # Cluster com MCP server remoto
└── templates/              # Templates Kubernetes
```

## 🏗️ Arquiteturas de Deploy

### Cluster cluster-a (Principal)

O **cluster cluster-a** é o cluster principal onde:

- ✅ **UI do kagent** - Interface web acessível aos usuários
- ✅ **Controller** - Gerenciamento dos recursos e agentes
- ✅ **Agentes** - Instâncias dos agentes de IA
- ✅ **MCP Server local** - Ferramentas para o cluster local
- ✅ **RemoteMCPServer** - Conexões para clusters remotos (ex: cluster-b)

**👥 Este é o cluster que os usuários acessarão para interagir com os agentes.**

#### Arquivo de configuração
`clusters/vrdev-cluster-a/master.yaml` - Configuração completa com UI e agentes habilitados.

### Cluster cluster-b (MCP Remoto)

O **cluster cluster-b** é um cluster secundário onde:

- ✅ **MCP Server** - Expõe ferramentas via HTTPS
- ❌ **UI** - Desabilitada (não há interface)
- ❌ **Agentes** - Desabilitados (apenas fornece ferramentas)

**🔧 Este cluster serve como provedor de ferramentas para o cluster cluster-a.**

#### Arquivo de configuração
`clusters/vrdev-cluster-b/agent.yaml` - Configuração apenas com MCP server habilitado.

## 🚀 Como Usar

### Deploy no Cluster cluster-a

```bash
# Instalar kagent com UI e agentes
helm install kagent . -f clusters/vrdev-cluster-a/master.yaml -n kagent --create-namespace
```

### Deploy no Cluster cluster-b

```bash
# Instalar somente MCP server
helm install kagent . -f clusters/vrdev-cluster-b/agent.yaml -n kagent --create-namespace
```

### Customizar Configuração

Você pode sobrescrever valores específicos:

```bash
helm install kagent . \
  -f clusters/vrdev-cluster-a/master.yaml \
  --set ui.enabled=true \
  --set controller.replicas=2 \
  -n kagent
```

## 🔗 Conectando Clusters

Para que o cluster cluster-a acesse ferramentas do cluster cluster-b:

1. **No cluster-b**: Exponha o MCP server via Ingress/LoadBalancer
2. **No cluster-a**: Configure um `RemoteMCPServer` apontando para o cluster-b
3. **No Agente**: Referencie o RemoteMCPServer nas ferramentas

Exemplo em `custom-agents/cross-cluster-agent/`

## 📚 Componentes Disponíveis

### Agentes
- `k8s-agent` - Agente para gerenciamento Kubernetes
- `observability-agent` - Agente de observabilidade
- `istio-agent` - Agente para service mesh Istio
- `kgateway-agent` - Agente para Kubernetes Gateway API
- `helm-agent` - Agente para gerenciamento Helm
- `cilium-*-agent` - Agentes para CNI Cilium
- `argo-rollouts-agent` - Agente para Argo Rollouts

### Ferramentas
- `kagent-tools` - Ferramentas core do kagent
- `kmcp` - Servidor MCP base
- `grafana-mcp` - Integração com Grafana
- `querydoc` - Consulta de documentação

## ⚙️ Configuração

Principais seções do `values.yaml`:

```yaml
# Componentes core
controller:
  enabled: true

ui:
  enabled: true    # Habilitar no cluster cluster-a
  
# MCP Server
kagent-tools:
  enabled: true    # Sempre habilitado
  
# Agentes específicos
agents:
  k8s-agent:
    enabled: true
  observability-agent:
    enabled: true
```

## 📖 Documentação

Para mais informações:
- [Documentação kagent](https://kagent.dev)
- [CRDs](../templates/) - Custom Resource Definitions
- [Agentes Customizados](../custom-agents/) - Exemplos de agentes personalizados
