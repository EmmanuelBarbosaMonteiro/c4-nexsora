# Plataforma SaaS Nexora - Manutenção Prescritiva Inteligente

## 🎯 Propósito

A **Plataforma SaaS Nexora** é uma solução especializada em **Manutenção Prescritiva** que utiliza Inteligência Artificial para predizer falhas em equipamentos industriais antes que elas aconteçam, otimizando custos operacionais e maximizando a disponibilidade de ativos.

## 🚀 Objetivos

### Objetivo Principal
Transformar dados de sensores IoT em insights acionáveis de manutenção através de um pipeline inteligente de agentes AI, reduzindo paradas não planejadas e custos de manutenção.

### Objetivos Específicos
- **Detecção Proativa**: Identificar anomalias em tempo real usando modelos de LLM
- **Predição Inteligente**: Prever falhas futuras com alta precisão usando análise de séries temporais
- **Planejamento Otimizado**: Gerar planos de manutenção contextualizados usando RAG (Retrieval-Augmented Generation)
- **Integração Seamless**: Conectar com ERPs e sistemas corporativos existentes sem vendor lock-in
- **Controle de Custos**: Oferece triggers configuráveis para otimizar uso de LLMs e controlar custos SaaS

## 🏗️ Arquitetura

### Paradigma: Pipeline Sequencial Event-Driven
A plataforma utiliza uma arquitetura de **pipeline sequencial** coordenada por uma **Fila de Eventos central**, permitindo escalabilidade e evolução futura para subagentes especializados.

```
Sensores IoT → Entrada → TimeSeries DB + Observador → Fila de Eventos
                                                         ↓
Agent Detecção Anomalias → Agent Análise Preditiva → Agent Necessidades Manutenção → Agent Emissão OS
                                    ↓                          ↓                              ↓
                           (Trigger Configurável)      RAG + Vector DB              Tools + MCP Gateway
```

### Componentes Principais

#### 🤖 Agentes AI Especializados
- **Agent Detecção de Anomalias**: Detecta padrões anômalos usando modelos ML próprios (scikit-learn)
- **Agent Análise Preditiva**: Prediz falhas usando LSTM/Prophet (acionado sob demanda via triggers)
- **Agent Necessidades de Manutenção**: Planeja manutenções usando RAG e LangChain
- **Agent Emissão de OS**: Gera ordens de serviço priorizadas

#### ⚙️ Trigger Manager Configurável
Permite ao usuário escolher entre duas estratégias:
1. **Análise Agendada**: Intervalos fixos ou horários específicos
2. **Baseada em Indicadores**: Limites personalizados e combinações lógicas

#### 🔐 Docker MCP Gateway
Centraliza credenciais e controla acesso a ferramentas externas (ERPs, APIs de notificação) via Model Context Protocol.

#### 📊 Infraestrutura de Dados
- **PostgreSQL**: Dados operacionais e configurações
- **InfluxDB/TimescaleDB**: Séries temporais otimizadas para IoT
- **Pinecone**: Vector database para embeddings RAG
- **RabbitMQ**: Message broker para coordenação de eventos

## 🛠️ Stack Tecnológica

### Backend
- **GO**: Linguagem para regras de negócio
- **Python**: Linguagem principal para agentes AI
- **LangChain**: Framework para agentes AI e RAG
- **CrewAI**: (Preparado para orquestração futura)
- **FastAPI**: APIs REST de alta performance
- **Celery**: Task queue para triggers agendados

### AI/ML
- **scikit-learn**: Detecção de anomalias
- **TensorFlow**: Modelos preditivos LSTM
- **Prophet**: Análise de séries temporais
- **Ollama**: LLMs locais (Llama 3.1, Mistral)
- **SentenceTransformers**: Embeddings semânticos

### Frontend
- **React + TypeScript**: Interface web moderna e responsiva
- **Material-UI**: Componentes de design

### Infraestrutura
- **Kubernetes**: Orquestração de containers
- **Docker**: Containerização
- **Prometheus + Grafana**: Monitoramento e métricas
- **Jaeger**: Distributed tracing

## 🎨 Características Inovadoras

### 1. **Trigger Configurável pelo Usuário**
- Controle granular de quando executar análises preditivas
- Otimização automática de custos de LLM
- Flexibilidade total para diferentes cenários de negócio

### 2. **Pipeline Evolutivo**
- Arquitetura preparada para separação futura de agentes
- Event-driven design permite adição de novos agentes sem impacto
- Fila central facilita auditoria e monitoramento

### 3. **Security by Design**
- MCP Gateway para controle de acesso a sistemas externos
- Vault seguro para credenciais
- Network policies e RBAC no Kubernetes

### 4. **Multi-tenant SaaS Ready**
- Isolamento por tenant
- Budget controls por cliente
- Rate limiting configurável
- Cost tracking transparente

## 🔧 Casos de Uso

### Indústria
- Monitoramento de ativos diversos
- Predição de falhas em produção
- Otimização de paradas programadas
- Predição de falhas em equipamentos críticos
- Integração com sistemas ERPs diversos

## 📈 Benefícios

### Para o Negócio
- **Redução** em paradas não planejadas
- **Economia** em custos de manutenção
- **Compliance** automatizada com normas de segurança

### Para Operações
- **Visibilidade total** do estado dos equipamentos
- **Alertas proativos** multi-canal (WhatsApp, E-mail, SMS, Discord, Slack, SAP)
- **Planejamento otimizado** de recursos e estoque
- **Integração seamless** com sistemas existentes

### Para TI
- **Arquitetura cloud-native** escalável
- **APIs padronizadas** para integração
- **Observabilidade completa** com métricas e traces
- **Security** enterprise-grade

## 🚀 Roadmap

### Fase 1 (MVP) - Atual
- ✅ Pipeline básico de agentes AI
- ✅ Trigger Manager configurável
- ✅ Integração com sensores IoT
- ✅ RAG para recomendações

### Fase 2 (Q1 2026)
- 🔄 Orquestração avançada com CrewAI
- 🔄 Subagentes especializados por tipo de equipamento
- 🔄 ML AutoML para treinamento automático de modelos
- 🔄 Dashboard avançado com Digital Twins

### Fase 3 (Q2 2026)
- 📋 Computer Vision para inspeção visual
- 📋 Integração com drones para monitoramento
- 📋 Marketplace de modelos AI
- 📋 Carbon footprint tracking

## 📚 Documentação

- [Architecture Decision Records (ADR)](./docs/adr/)
- [Diagramas C4](./diagramas/novo/)
- [API Documentation](./docs/api/)
- [Deployment Guide](./docs/deployment/)

---

**Nexora SaaS Platform** - Transformando Manutenção através de Inteligência Artificial 🤖⚡