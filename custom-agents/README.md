# Custom Agents

Este diretório contém exemplos de agentes customizados para kagent, demonstrando diferentes casos de uso e configurações.

## 📁 Estrutura

### `sre-agent.yaml`

Agente de exemplo voltado para tarefas de **SRE (Site Reliability Engineering)** em ambientes Kubernetes. Este agente demonstra como criar um assistente especializado para:

- Diagnóstico de problemas em clusters Kubernetes
- Análise de logs e eventos
- Verificação de conectividade de serviços
- Observabilidade e monitoramento

O agente utiliza ferramentas do MCP server local e segue as melhores práticas de SRE para fornecer insights sobre o cluster.

### `cross-cluster-agent/`

Agente especializado para **análise de múltiplos clusters Kubernetes**. Este agente permite que você:

- Acesse ferramentas de diferentes clusters simultaneamente
- Realize diagnósticos cross-cluster
- Compare configurações e estados entre clusters

**📍 Instalação Recomendada:** Este agente deve ser instalado de preferência no cluster onde estará a **UI do kagent**, pois ele consome ferramentas MCP remotas de outros clusters via HTTPS.

Para instruções detalhadas de configuração e deploy, consulte o [README do cross-cluster-agent](cross-cluster-agent/README.md).

## 🚀 Como Usar

1. **Personalize** os arquivos YAML conforme suas necessidades
2. **Aplique** os manifestos no cluster utilizando `kubectl apply -f <arquivo>`
3. **Acesse** os agentes através da UI do kagent

## 📚 Documentação

Para mais informações sobre como criar e configurar agentes customizados, consulte:
- [Documentação kagent](https://kagent.dev)
- CRD `Agent` em `/templates/kagent.dev_agents.yaml`
