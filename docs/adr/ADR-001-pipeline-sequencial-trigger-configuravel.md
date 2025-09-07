# ADR-001: Pipeline Sequencial Event-Driven com Trigger Configurável

**Status**: Aceito  
**Data**: 2025-09-04  
**Autor**: Equipe de Arquitetura  

## Contexto

A Plataforma SaaS Nexora precisa processar dados de sensores IoT em tempo real para fornecer manutenção prescritiva inteligente. Os principais desafios incluem:

1. **Escalabilidade**: Centenas de sensores gerando dados continuamente
2. **Controle de Custos**: Uso eficiente de LLMs para análise preditiva
3. **Flexibilidade**: Diferentes clientes têm necessidades distintas de frequência de análise
4. **Evolução**: Arquitetura deve permitir separação futura em subagentes especializados

## Decisão

Adotamos uma **arquitetura de pipeline sequencial coordenada por fila de eventos** com um **Trigger Manager configurável** que permite ao usuário escolher quando executar análises preditivas.

### Componentes Principais:

1. **Pipeline Sequencial de Agentes AI**:
   - Agent Detecção de Anomalias (sempre ativo)
   - Agent Análise Preditiva (acionado sob demanda)
   - Agent Necessidades de Manutenção
   - Agent Emissão de Ordens de Serviço

2. **Fila de Eventos Central (RabbitMQ)**:
   - Hub de coordenação entre todos os agentes
   - Diferentes filas para diferentes tipos de eventos
   - Facilita auditoria e monitoramento

3. **Trigger Manager Configurável**:
   - **Opção 1**: Análise Agendada (intervalos fixos/horários específicos)
   - **Opção 2**: Baseada em Indicadores (thresholds personalizados)

4. **Docker MCP Gateway**:
   - Centraliza credenciais para sistemas externos
   - Controla acesso a ferramentas externas (ERPs, APIs)

## Alternativas Consideradas

### 1. CrewAI Orquestração Colaborativa
- **Prós**: Agentes trabalham colaborativamente, mais "inteligente"
- **Contras**: Mais complexo, dificulta separação futura, overhead desnecessário
- **Por que rejeitado**: O pipeline sequencial é mais adequado para o caso de uso

### 2. Análise Preditiva Contínua
- **Prós**: Resposta imediata a qualquer mudança
- **Contras**: Custo prohibitivo de LLMs, processamento desnecessário
- **Por que rejeitado**: Não escalável para SaaS com múltiplos tenants

### 3. Event Listener Direto (sem Observer Pattern)
- **Prós**: Menos componentes, mais simples
- **Contras**: Acoplamento maior, menos flexível para evolução
- **Por que rejeitado**: Observer Pattern oferece melhor desacoplamento

## Consequências

### Positivas ✅

1. **Controle de Custos**: Agent Preditiva só roda quando necessário
2. **Flexibilidade**: Usuários configuram triggers conforme necessidade
3. **Escalabilidade**: Cada agente pode escalar independentemente
4. **Evolução**: Fácil separação em subagentes futuros
5. **Auditoria**: Fila central facilita monitoramento e debugging
6. **Multi-tenant**: Isolamento e budget control por cliente

### Negativas ❌

1. **Complexidade**: Mais componentes para gerenciar
2. **Latência**: Pipeline sequencial pode introduzir delay
3. **Configuração**: Usuários precisam entender triggers para otimizar

### Neutras 🟡

1. **Performance**: Trade-off entre custo e velocidade de resposta
2. **Manutenção**: Mais lógica de trigger, mas melhor observabilidade

## Detalhes de Implementação

### Stack Tecnológica Escolhida:

**Backend & AI:**
- Python + LangChain para agentes
- FastAPI para APIs REST
- Celery + Redis para triggers agendados
- scikit-learn para detecção de anomalias
- TensorFlow/Prophet para análise preditiva

**Infraestrutura de Dados:**
- PostgreSQL: Dados operacionais e configurações de triggers
- InfluxDB/TimescaleDB: Séries temporais otimizadas para IoT
- Pinecone: Vector database para RAG
- RabbitMQ: Message broker para eventos

**Deployment:**
- Kubernetes para orquestração
- Docker MCP Gateway para segurança
- Prometheus/Grafana para monitoramento

### Triggers Configuráveis:

#### Análise Agendada:
```yaml
trigger_type: "scheduled"
schedule:
  interval: "6h"  # ou "daily", "weekly"
  timezone: "America/Sao_Paulo"
  specific_times: ["08:00", "14:00", "20:00"]
budget:
  max_analyses_per_day: 10
  cost_limit_usd: 50.00
```

#### Baseada em Indicadores:
```yaml
trigger_type: "threshold"
conditions:
  - sensor: "temperature"
    operator: ">"
    value: 80
    unit: "celsius"
  - sensor: "vibration"
    operator: "deviation"
    value: 2  # 2 desvios padrão
logic: "OR"  # ou "AND"
cooldown_minutes: 30
```

## Monitoramento e Métricas

### KPIs Principais:
- **Cost per Analysis**: Custo médio por execução do Agent Preditiva
- **Trigger Accuracy**: % de triggers que resultaram em ações necessárias
- **MTTR (Mean Time to Repair)**: Tempo médio de resolução
- **MTBF (Mean Time Between Failures)**: Tempo médio entre falhas

### Alertas:
- Budget limits próximos do limite
- Triggers com alta frequência de falsos positivos
- Agents com alta latência de processamento

## Validação

### Testes de Performance:
- [ ] Pipeline suporta 1000+ sensores simultâneos
- [ ] Latência < 5s para detecção de anomalias
- [ ] Análise preditiva completa < 2min

### Testes de Custo:
- [ ] Budget limits funcionam corretamente
- [ ] Triggers reduzem custos em pelo menos 70%
- [ ] Cost tracking preciso por tenant

### Testes de Evolução:
- [ ] Adição de novo agente sem impacto
- [ ] Separação de agente em subagentes
- [ ] Migração de triggers sem downtime

## Revisão Futura

Este ADR deve ser revisado quando:
1. **Q1 2026**: Avaliação para migração para CrewAI full orchestration
2. **Q2 2026**: Análise de performance com 10.000+ sensores
3. **Sempre**: Quando custos de LLM mudarem significativamente

## Referências

- [C4 Architecture Diagrams](../diagramas/novo/)
- [Sequence Diagrams](../diagramas/novo/sequence-fluxo-prescritivo-crewai.puml)
- [Docker MCP Gateway Documentation](https://docs.docker.com/mcp/)
- [LangChain Agent Documentation](https://langchain.readthedocs.io/)

---

**Next ADRs:**
- ADR-002: Escolha de Vector Database (Pinecone vs Alternatives)
- ADR-003: LLM Strategy (Ollama Local vs Cloud APIs)
- ADR-004: Multi-tenant Isolation Strategy