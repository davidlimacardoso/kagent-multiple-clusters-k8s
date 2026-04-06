# Cross-Cluster Agent Configuration

Este diretório contém as configurações para um agente kagent que acessa ferramentas em **múltiplos clusters Kubernetes**.

## 📋 Visão Geral

A arquitetura permite que um agente executando no **Cluster cluster-a** (onde está o UI do kagent) consuma ferramentas MCP de um **Cluster cluster-b** remoto via HTTPS.

```
┌────────────────────────────────┐          ┌────────────────────────────────┐
│   Cluster cluster-a            │          │   Cluster cluster-b              │
│   (UI + Agentes)               │          │   (MCP Tools Server)           │
│                                │          │                                │
│  ┌──────────────────────┐      │  HTTPS   │  ┌──────────────────────┐      │
│  │ cross-cluster-agent  │──────┼──────────┼─→│   kagent-tools       │      │
│  │  (UI aqui)           │      │  /mcp    │  │   (port 8084)        │      │
│  └──────────────────────┘      │          │  └──────────────────────┘      │
│           ↓                    │          │           ↑                    │
│  ┌──────────────────────┐      │          │  ┌──────────────────────┐      │
│  │  RemoteMCPServer     │      │          │  │  Ingress (ALB)       │      │
│  │  cluster-b-tools     │      │          │  │  HTTPS :443          │      │
│  └──────────────────────┘      │          │  └──────────────────────┘      │
│                                │          │                                │
│  ┌──────────────────────┐      │          │                                │
│  │ kagent-tool-server   │      │          │                                │
│  │ (ferramentas locais) │      │          │                                │
│  └──────────────────────┘      │          │                                │
└────────────────────────────────┘          └────────────────────────────────┘
```

## 📁 Arquivos

- **`remote-mcp-server.yaml`** - Define a conexão remota ao cluster cluster-b
- **`cross-cluster-agent.yaml`** - Configuração do agente com acesso aos dois clusters
- **`cluster-b-ingress.yaml`** - Ingress para expor o MCP server do cluster-b *(aplicar no cluster-b)*

## 🚀 Deploy

### ⚠️ IMPORTANTE: Ordem de Aplicação

#### 1️⃣ No Cluster cluster-b (servidor de ferramentas)

**a) Configure o values.yaml para ser apenas um MCP server:**

```yaml
# kagent/clusters/cluster-b.yaml
kagent-tools:
  enabled: true  # ✅ ESSENCIAL

agents:
  k8s-agent:
    enabled: false  # ❌ Desabilitar todos os agents
  # ... demais agents disabled
```

**b) Instale o kagent no cluster-b:**

```bash
# No cluster cluster-b
helm upgrade kagent ./kagent -f kagent/clusters/cluster-b.yaml -n kagent --create-namespace
```

**c) Exponha o MCP server via Ingress:**

```bash
# No cluster cluster-b
kubectl apply -f cross-cluster-agent/cluster-b-ingress.yaml

# Verifique
kubectl get ingress kagent-mcp-server -n kagent
```

**d) Teste a conectividade:**

```bash
curl -v https://kagent-mcp-cluster-b.stg.myagent.io/mcp
# Esperado: Resposta do MCP server
```

---

#### 2️⃣ No Cluster cluster-a (onde está o UI)

**a) Crie o RemoteMCPServer apontando para o cluster-b:**

```bash
# No cluster cluster-a (onde está o UI)
kubectl apply -f cross-cluster-agent/remote-mcp-server.yaml
```

**b) Verifique o status do RemoteMCPServer:**

```bash
kubectl get remotemcpserver cluster-b-tools -n kagent -o yaml
kubectl describe remotemcpserver cluster-b-tools -n kagent
```

**c) Crie o cross-cluster-agent:**

```bash
# No cluster cluster-a (onde está o UI)
kubectl apply -f cross-cluster-agent/cross-cluster-agent.yaml
```

**d) Verifique se o agente está rodando:**

```bash
kubectl get pods -n kagent -l app.kubernetes.io/name=cross-cluster-agent
kubectl logs -n kagent -l app.kubernetes.io/name=cross-cluster-agent
```

## 🔧 Configuração do Agente

O `cross-cluster-agent` tem acesso a ferramentas de ambos os clusters:

### Ferramentas do Cluster cluster-a (local)
- `k8s_get_resources` - Lista recursos Kubernetes
- `k8s_get_pod_logs` - Consulta logs de pods
- `k8s_describe_resource` - Descreve recursos

### Ferramentas do Cluster cluster-b (remoto)
- `shell` - Executa comandos kubectl no cluster-b
- `prometheus_query_tool` - Consulta métricas do Prometheus
- `prometheus_targets_tool` - Lista targets do Prometheus
- `prometheus_label_names_tool` - Lista labels do Prometheus

## 💬 Como Usar no UI

### Exemplos de Perguntas

**Para consultar o Cluster cluster-a:**
```
- "Liste os pods no namespace kagent"
- "Como está a saúde do cluster?"
- "Mostre os logs do pod kagent-controller"
```

**Para consultar o Cluster cluster-b:**
```
- "Como está a saúde no cluster cluster-b?"
- "Liste os pods no namespace kagent do cluster cluster-b"
- "Mostre os nodes do cluster cluster-b"
- "Execute 'kubectl get deployments -n kube-system' no cluster-b"
```

**Para métricas (sempre do cluster-b):**
```
- "Qual o uso de CPU do cluster?"
- "Mostre as métricas do Prometheus"
```

## 🔍 Troubleshooting

### Erro: "Failed to create MCP session"

**Causa:** O agente não consegue conectar ao MCP server remoto.

**Verificações:**

1. **Teste a URL do Ingress:**
```bash
kubectl run curl-test --image=curlimages/curl -it --rm -- \
  curl -v https://kagent-mcp-cluster-b.stg.myagent.io/mcp
```

2. **Verifique se o MCP server está rodando no cluster-b:**
```bash
# No cluster cluster-b
kubectl get pods -n kagent -l app.kubernetes.io/name=tools
kubectl logs -n kagent -l app.kubernetes.io/name=tools
```

3. **Verifique o Ingress:**
```bash
# No cluster cluster-b
kubectl get ingress kagent-mcp-server -n kagent
kubectl describe ingress kagent-mcp-server -n kagent
```

4. **Verifique o RemoteMCPServer:**
```bash
# No cluster cluster-a
kubectl get remotemcpserver cluster-b-tools -n kagent -o yaml
```

### Erro: "Agent só consulta o cluster cluster-a"

**Causa:** O agente não está usando a ferramenta `shell` do cluster-b corretamente.

**Solução:** Seja explícito na pergunta, mencionando "cluster cluster-b" ou "no cluster-b".

### Timeout ao consultar o cluster-b

**Causa:** Timeout muito curto ou problemas de rede.

**Solução:** Aumente o timeout em `remote-mcp-server.yaml`:

```yaml
spec:
  timeout: 60s  # Aumentar se necessário
  sseReadTimeout: 15m0s
```

## 📊 Monitoramento

### Verificar logs do agente:
```bash
kubectl logs -n kagent -l app.kubernetes.io/name=cross-cluster-agent -f
```

### Verificar logs do MCP server no cluster-b:
```bash
# No cluster cluster-b
kubectl logs -n kagent -l app.kubernetes.io/name=tools -f
```

### Verificar status do RemoteMCPServer:
```bash
kubectl get remotemcpserver -n kagent
kubectl describe remotemcpserver cluster-b-tools -n kagent
```

## 🔐 Segurança

### Requisitos de Rede
- Clusters devem ter conectividade (VPC peering, VPN, etc.)
- Ingress deve usar HTTPS com certificado válido
- Security Groups devem permitir tráfego entre os clusters

### Autenticação
O MCP server no cluster-b usa o service account `kagent-tools` que possui RBAC adequado para operações de leitura/escrita no cluster.

### Recomendações
- Configure mutual TLS entre os clusters
- Use tokens ou certificados para autenticação adicional
- Limite os Security Groups ao mínimo necessário
- Monitore as chamadas cross-cluster via logs

## 🎯 Próximos Passos

- [ ] Adicionar mais ferramentas específicas do cluster-b
- [ ] Configurar autenticação avançada (mTLS)
- [ ] Adicionar métricas de latência cross-cluster
- [ ] Criar alertas para falhas de conectividade
- [ ] Documentar disaster recovery entre clusters

## 📚 Referências

- [kagent Documentation](https://kagent.dev)
- [MCP Protocol](https://modelcontextprotocol.io)
- [Kubernetes Multi-Cluster Patterns](https://kubernetes.io/docs/concepts/cluster-administration/federation/)
