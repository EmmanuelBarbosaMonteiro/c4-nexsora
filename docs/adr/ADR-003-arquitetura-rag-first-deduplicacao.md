# ADR-003: Arquitetura RAG-First com Deduplicação Inteligente

**Status**: Aceito  
**Data**: 2025-09-08  
**Autor**: Equipe de Arquitetura  

## Contexto

Com a implementação completa da gestão de dados mestre (ADR-002), identificamos a necessidade de otimizar o acesso dos agentes AI às informações contextualizadas. O desafio principal era eliminar a dependência direta dos agentes aos serviços CRUD, centralizando todo o conhecimento em uma base vetorizada para melhor performance e consistência.

### Problemas Identificados:

1. **Acesso Fragmentado**: Agentes consultavam múltiplos serviços CRUD para obter contexto
2. **Duplicação de Processamento**: Manuais e especificações similares sendo vetorizados múltiplas vezes
3. **Inconsistência de Dados**: Possibilidade de informações desatualizadas entre RAG e CRUDs
4. **Performance Subótima**: Múltiplas consultas HTTP para formar contexto completo

### Casos de Uso Específicos:

- **Equipamentos Similares**: Mesmo modelo/fabricante com especificações idênticas
- **Sensores Reutilizados**: Múltiplos sensores do mesmo tipo com manual compartilhado
- **Manuais Duplicados**: Mesmo arquivo PDF para diferentes equipamentos
- **FMEA Compartilhada**: Análises de falha aplicáveis a família de equipamentos

## Decisão

Implementamos uma **Arquitetura RAG-First com Deduplicação Inteligente** onde:

### 1. **Auto-Triggers em Todos os CRUDs**
- Equipment Management Service → AUTO-TRIGGER RAG
- Sensor Management Service → AUTO-TRIGGER RAG  
- Manual & Documentation Service → AUTO-TRIGGER RAG
- Maintenance History Service → AUTO-TRIGGER RAG

### 2. **RAG Processing Orchestrator**
- **RAG Coordinator**: Coordena todo processamento automático
- **Deduplication Engine**: Evita reprocessamento de conteúdo idêntico
- **Content Hasher**: Gera fingerprints para identificação de duplicatas
- **Processing Queue**: Fila priorizada para processamento eficiente

### 3. **Agentes AI - Acesso Exclusivo via RAG**
- ✅ **Anomaly Detection Agent**: Busca contexto APENAS via RAG
- ✅ **Predictive Analysis Agent**: Busca contexto APENAS via RAG
- ✅ **Maintenance Planning Agent**: Busca conhecimento APENAS via RAG
- ✅ **Work Order Agent**: Busca contexto APENAS via RAG
- ⚠️ **Exceção Única**: Work Order Agent registra nova manutenção via CRUD

### 4. **Sistema de Deduplicação Inteligente**
- **Content Hashing**: MD5/SHA256 para arquivos + structural hash para specs
- **Smart Linking**: Mesmo conteúdo linkado a múltiplas entidades
- **Cache Strategy**: Redis para lookup rápido de duplicatas
- **Version Tracking**: Controle de versões para updates incrementais

## Alternativas Consideradas

### 1. **Sincronização Bidirecional RAG ↔ CRUD**
- **Prós**: Mantém ambos os sistemas atualizados
- **Contras**: Complexidade alta, possibilidade de inconsistências
- **Por que rejeitado**: RAG-first é mais simples e consistente

### 2. **Cache Compartilhado Entre RAG e CRUDs**
- **Prós**: Evita duplicação de dados em memória
- **Contras**: Não resolve fragmentação de consultas
- **Por que rejeitado**: Não elimina dependências múltiplas dos agentes

### 3. **Processamento Manual de Deduplicação**
- **Prós**: Controle total sobre quais conteúdos são duplicados
- **Contras**: Não escalável, propenso a erro humano
- **Por que rejeitado**: Deduplicação automática é mais eficiente

## Consequências

### Positivas ✅

1. **Performance Otimizada**: Agentes fazem 1 consulta RAG vs múltiplas consultas CRUD
2. **Consistência Garantida**: RAG Service como fonte única de verdade
3. **Processamento Eficiente**: Deduplicação evita trabalho redundante
4. **Escalabilidade**: Sistema otimizado para crescimento de dados
5. **Custo Reduzido**: Menos processamento de embeddings duplicados
6. **Manutenção Simplificada**: Lógica centralizada no RAG Service

### Negativas ❌

1. **Complexidade Inicial**: Implementação do sistema de deduplicação
2. **Latência de Atualização**: Delay entre CRUD save e disponibilidade no RAG
3. **Dependência Crítica**: RAG Service torna-se ponto único de falha
4. **Storage Overhead**: Metadados de deduplicação no PostgreSQL

### Neutras 🟡

1. **Migração de Dados**: Necessário reprocessar base existente com deduplicação
2. **Learning Curve**: Equipe precisa entender novo fluxo RAG-first

## Detalhes de Implementação

### Auto-Triggers Webhook Pattern:

```python
# Exemplo: Equipment Service
class EquipmentService:
    async def create_equipment(self, equipment_data: EquipmentCreate):
        # 1. Validação e persistência
        equipment = await self.repository.create(equipment_data)
        
        # 2. AUTO-TRIGGER para RAG
        await self.rag_webhook.trigger_processing({
            "entity_type": "equipment",
            "entity_id": equipment.id,
            "action": "create",
            "priority": equipment.criticality
        })
        
        return equipment
```

### Deduplication Engine:

```python
class DeduplicationEngine:
    async def check_content_exists(self, content_hash: str) -> Optional[str]:
        # 1. Check Redis cache primeiro
        cached_result = await self.redis.get(f"dedup:{content_hash}")
        if cached_result:
            return cached_result
            
        # 2. Check PostgreSQL para hash existente
        existing = await self.db.query(
            "SELECT vector_id FROM content_hashes WHERE hash = %s", 
            content_hash
        )
        
        if existing:
            # 3. Cache resultado para próximas consultas
            await self.redis.setex(
                f"dedup:{content_hash}", 
                3600, 
                existing.vector_id
            )
            return existing.vector_id
            
        return None
```

### RAG Coordinator com Deduplicação:

```python
class RAGCoordinator:
    async def process_content(self, request: ProcessingRequest):
        # 1. Gerar hash do conteúdo
        content_hash = await self.hasher.generate_hash(request)
        
        # 2. Verificar se já existe
        existing_vector_id = await self.dedup_engine.check_content_exists(content_hash)
        
        if existing_vector_id:
            # 3. Apenas criar link metadata → vector existente
            await self.create_entity_vector_link(
                entity_id=request.entity_id,
                vector_id=existing_vector_id
            )
        else:
            # 4. Processar novo conteúdo
            await self.processing_queue.enqueue(request, priority=request.priority)
```

### Pinecone Metadata com Deduplicação:

```json
{
  "vector_id": "vec_123456",
  "content_hash": "sha256:abc123...",
  "linked_entities": [
    {
      "entity_type": "equipment",
      "entity_id": "eq_001",
      "relationship": "manual"
    },
    {
      "entity_type": "equipment", 
      "entity_id": "eq_002",
      "relationship": "manual"
    }
  ],
  "document_type": "maintenance_manual",
  "manufacturer": "Siemens",
  "model_family": "S7-1500",
  "created_at": "2025-09-08T10:00:00Z",
  "last_updated": "2025-09-08T10:00:00Z"
}
```

## Schema de Banco para Deduplicação

```sql
-- Tabela para tracking de hashes de conteúdo
CREATE TABLE content_hashes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_hash VARCHAR(64) NOT NULL UNIQUE,
    content_type ENUM('manual', 'specification', 'fmea', 'maintenance_case'),
    vector_id VARCHAR(255) NOT NULL,
    file_path VARCHAR(500),
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_content_hash (content_hash),
    INDEX idx_vector_id (vector_id)
);

-- Tabela para linking entities → vectors
CREATE TABLE entity_vector_links (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type ENUM('equipment', 'sensor', 'manual', 'maintenance'),
    entity_id UUID NOT NULL,
    vector_id VARCHAR(255) NOT NULL,
    relationship_type ENUM('specification', 'manual', 'fmea', 'historical_case'),
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_entity (entity_type, entity_id),
    INDEX idx_vector (vector_id)
);

-- Tabela para status de processamento
CREATE TABLE rag_processing_status (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type ENUM('equipment', 'sensor', 'manual', 'maintenance'),
    entity_id UUID NOT NULL,
    content_hash VARCHAR(64),
    status ENUM('pending', 'processing', 'completed', 'failed'),
    progress INTEGER DEFAULT 0,
    error_message TEXT,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);
```

## Fluxo de Consulta RAG Otimizado

### Agent Query Example:

```python
# Agent solicita contexto para equipamento específico
async def get_equipment_context(self, equipment_id: str) -> str:
    rag_query = f"""
    Buscar informações completas sobre equipamento {equipment_id}:
    - Especificações técnicas
    - Procedimentos de manutenção  
    - Histórico de falhas similares
    - Sensores instalados e características
    - Análises FMEA relacionadas
    """
    
    # UMA única consulta RAG retorna contexto completo
    context = await self.rag_service.query(
        query=rag_query,
        filters={
            "entity_id": equipment_id,
            "entity_type": "equipment"
        },
        max_results=50
    )
    
    return context
```

## Métricas de Sucesso

### Performance:
- **Query Latency**: < 500ms para consultas RAG complexas
- **Deduplication Rate**: > 40% de conteúdo deduplicated
- **Processing Efficiency**: > 70% redução em tempo de processamento
- **Storage Savings**: > 60% redução em embeddings duplicados

### Qualidade:
- **Context Accuracy**: > 95% precisão em contexto retornado
- **Link Integrity**: < 1% broken links entre entities e vectors
- **Update Latency**: < 2min entre CRUD save e RAG availability

### Operacional:
- **System Availability**: > 99.9% uptime do RAG Service
- **Error Rate**: < 0.1% falha no processamento automático
- **Cache Hit Rate**: > 80% cache hit para deduplicação checks

## Monitoramento e Alertas

### Dashboards:
- **RAG Processing Pipeline**: Status, throughput, error rate
- **Deduplication Metrics**: Savings, cache performance, duplicate detection
- **Agent Query Performance**: Latency, cache hits, context quality
- **Storage Efficiency**: Vector count, deduplication ratio, cost savings

### Alertas Críticos:
- RAG Service down/unhealthy
- Processing queue backlog > 1000 items
- Deduplication cache hit rate < 60%
- Agent query latency > 1s (95th percentile)

## Migração e Rollback

### Migração Plan:
1. **Fase 1**: Implementar deduplication engine em parallel
2. **Fase 2**: Reprocessar base existente com deduplicação
3. **Fase 3**: Migrar agentes para RAG-only access gradualmente
4. **Fase 4**: Remover acesso direto CRUD dos agentes

### Rollback Strategy:
- Feature flags para reverter agentes para acesso CRUD
- Backup de metadados de deduplicação
- Reprocessing capability para regenerar vectors

## Revisão Futura

Este ADR deve ser revisado quando:
1. **Q1 2026**: Avaliação de performance após 6 meses em produção
2. **Q2 2026**: Análise de cost savings com deduplicação
3. **Sempre**: Quando novos tipos de entidade forem adicionados ao sistema

## Referências

- [ADR-001: Pipeline Sequencial Event-Driven](./ADR-001-pipeline-sequencial-trigger-configuravel.md)
- [ADR-002: Gestão Completa de Dados Mestre](./ADR-002-gestao-dados-mestre.md)
- [C4 Container Diagram - RAG First](../C4/Nexsora/c2-containers.puml)
- [C4 Component Diagram - Deduplication](../C4/Nexsora/c3-componentes.puml)

---

**Relacionado:**
- ADR-001: Pipeline Sequencial Event-Driven
- ADR-002: Gestão Completa de Dados Mestre  
- ADR-004: Vector Database Strategy (Próximo)
- ADR-005: Multi-tenant Data Isolation (Próximo)