# ADR-004: Herança Inteligente de Modelos para Simplificação Extrema do Cadastro

**Status**: Aceito  
**Data**: 2025-09-08  
**Autor**: Equipe de Arquitetura  

## Contexto

Após a implementação da arquitetura RAG-First (ADR-003), identificamos uma oportunidade de simplificação ainda maior no processo de cadastro de equipamentos. O problema principal era a duplicação massiva de informações quando múltiplos equipamentos do mesmo modelo eram cadastrados, gerando trabalho repetitivo e inconsistência de dados.

### Problemas Identificados:

1. **Cadastro Repetitivo**: Mesmo modelo de equipamento sendo cadastrado múltiplas vezes com especificações idênticas
2. **Duplicação de Manuais**: Mesmo manual PDF sendo uploaded para cada equipamento individual
3. **Inconsistência**: Possibilidade de especificações diferentes para o mesmo modelo
4. **Trabalho Manual Excessivo**: 90% do cadastro sendo informações que já existem para o modelo
5. **Processamento RAG Redundante**: Vetorização desnecessária de conteúdo idêntico

### Casos de Uso Específicos:

- **Fábrica com 50 PLCs Siemens S7-1500**: Mesmo manual, mesmas especificações, diferentes apenas no serial/localização
- **Linha com 20 sensores PT100**: Mesmo tipo, mesmo manual de calibração, diferentes apenas na instalação
- **Família de equipamentos ABB**: Modelos similares compartilhando FMEA e procedimentos

## Decisão

Implementamos uma **Arquitetura de Herança Inteligente baseada em Modelos e Tipos** onde:

### 1. **Master Data Architecture**
- **Equipment Model Service**: CRUD de modelos de equipamentos (master data)
- **Sensor Type Registry**: CRUD de tipos de sensores (master data)  
- **Manual & Documentation Service**: Manuais vinculados a modelos/tipos
- **FMEA Manager**: Análises compartilhadas por família de equipamentos

### 2. **Inheritance Orchestrator**
- **Model Inheritance Engine**: Engine de herança baseada em modelo/sensores
- **Smart Linking Service**: Linking automático baseado em herança
- **Equipment Configurator**: Interface inteligente para seleção modelo + sensores
- **Cascade Update Manager**: Atualizações em cascata quando modelo é atualizado

### 3. **Fluxo Simplificado de Cadastro**
1. **Pessoa seleciona**: Modelo de equipamento + tipos de sensores via dropdown
2. **Sistema herda automaticamente**: Especificações, manuais, FMEA, procedimentos
3. **Cadastro específico**: Apenas serial number, localização, configurações particulares
4. **RAG processing**: Apenas do histórico específico + linking inteligente

### 4. **Processing Optimization**
- **Processamento único por modelo/tipo**: Embedding gerado uma vez, linkado N vezes
- **Smart Linking**: Equipment individual → modelo base no Pinecone
- **Cascade Updates**: Atualização de modelo propaga para todos os equipamentos
- **Inheritance Cache**: Cache Redis para herança instantânea

## Alternativas Consideradas

### 1. **Template-Based Equipment Creation**
- **Prós**: Flexibilidade para modificar especificações por equipamento
- **Contras**: Ainda permite inconsistência, não elimina duplicação
- **Por que rejeitado**: Herança automática garante consistência total

### 2. **Equipment Categories com Partial Inheritance**
- **Prós**: Meio termo entre flexibilidade e consistência
- **Contras**: Complexidade adicional, ainda permite inconsistências parciais
- **Por que rejeitado**: Full inheritance é mais simples e consistente

### 3. **Manual Tagging System**
- **Prós**: Permite linking manual de manuais a múltiplos equipamentos
- **Contras**: Propenso a erro humano, não automatiza especificações
- **Por que rejeitado**: Herança automática elimina erro humano

## Consequências

### Positivas ✅

1. **Redução Dramática de Cadastro**: 90% menos campos para preencher
2. **Consistência Garantida**: Impossível ter especificações diferentes para mesmo modelo
3. **Eliminação de Duplicação**: Zero redundância de manuais e especificações
4. **RAG Processing Otimizado**: Processamento único por modelo/tipo
5. **Manutenção Simplificada**: Atualização de modelo propaga automaticamente
6. **Onboarding Acelerado**: Novos usuários cadastram equipamentos em minutos
7. **Storage Efficiency**: Redução massiva de armazenamento duplicado

### Negativas ❌

1. **Rigidez de Modelo**: Equipamentos devem seguir especificações do modelo
2. **Setup Inicial**: Necessário cadastrar modelos/tipos antes dos equipamentos
3. **Complexidade de Herança**: Lógica adicional para gerenciar inheritance
4. **Dependência de Master Data**: Qualidade dos equipamentos depende da qualidade dos modelos

### Neutras 🟡

1. **Learning Curve**: Usuários precisam entender conceito de herança
2. **Migration Complexity**: Migração de base existente para estrutura de herança

## Detalhes de Implementação

### Equipment Model Schema:

```sql
-- Modelos de equipamentos (master data)
CREATE TABLE equipment_models (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    manufacturer VARCHAR(255) NOT NULL,
    model_name VARCHAR(255) NOT NULL,
    model_family VARCHAR(255),
    technical_specifications JSONB NOT NULL,
    operational_limits JSONB,
    maintenance_requirements JSONB,
    criticality_default ENUM('LOW', 'MEDIUM', 'HIGH', 'CRITICAL'),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(manufacturer, model_name)
);

-- Tipos de sensores (master data)
CREATE TABLE sensor_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    manufacturer VARCHAR(255) NOT NULL,
    type_name VARCHAR(255) NOT NULL,
    measurement_type ENUM('TEMPERATURE', 'VIBRATION', 'PRESSURE', 'FLOW'),
    technical_specifications JSONB NOT NULL,
    calibration_requirements JSONB,
    operational_limits JSONB,
    installation_guidelines JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(manufacturer, type_name)
);

-- Equipamentos individuais (dados específicos apenas)
CREATE TABLE equipment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_model_id UUID REFERENCES equipment_models(id) NOT NULL,
    serial_number VARCHAR(255) NOT NULL,
    installation_location VARCHAR(255),
    installation_date DATE,
    specific_configurations JSONB, -- Apenas configurações particulares
    notes TEXT,
    plant_id UUID REFERENCES plants(id),
    line_id UUID REFERENCES production_lines(id),
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(serial_number)
);
```

### Inheritance Engine:

```python
class ModelInheritanceEngine:
    async def create_equipment_with_inheritance(
        self, 
        model_id: UUID, 
        sensor_type_ids: List[UUID],
        specific_data: EquipmentSpecificData
    ) -> Equipment:
        
        # 1. Buscar dados do modelo
        model = await self.model_repository.get_by_id(model_id)
        
        # 2. Buscar dados dos tipos de sensores
        sensor_types = await self.sensor_repository.get_by_ids(sensor_type_ids)
        
        # 3. Criar equipamento com dados específicos apenas
        equipment = await self.equipment_repository.create({
            "equipment_model_id": model_id,
            "serial_number": specific_data.serial_number,
            "installation_location": specific_data.location,
            "specific_configurations": specific_data.configurations
        })
        
        # 4. Configurar herança no RAG
        await self.smart_linking_service.setup_inheritance_links(
            equipment_id=equipment.id,
            model_id=model_id,
            sensor_type_ids=sensor_type_ids
        )
        
        return equipment
```

### Smart Linking Service:

```python
class SmartLinkingService:
    async def setup_inheritance_links(
        self,
        equipment_id: UUID,
        model_id: UUID, 
        sensor_type_ids: List[UUID]
    ):
        
        # 1. Criar links para embeddings do modelo
        model_vectors = await self.vector_db.query(
            filters={"entity_type": "equipment_model", "entity_id": str(model_id)}
        )
        
        for vector in model_vectors:
            await self.create_inheritance_link(
                child_entity_id=equipment_id,
                child_entity_type="equipment",
                parent_vector_id=vector.id,
                relationship_type="model_inheritance"
            )
        
        # 2. Criar links para embeddings dos tipos de sensores
        for sensor_type_id in sensor_type_ids:
            sensor_vectors = await self.vector_db.query(
                filters={"entity_type": "sensor_type", "entity_id": str(sensor_type_id)}
            )
            
            for vector in sensor_vectors:
                await self.create_inheritance_link(
                    child_entity_id=equipment_id,
                    child_entity_type="equipment",
                    parent_vector_id=vector.id,
                    relationship_type="sensor_inheritance"
                )
```

### Equipment Configurator Interface:

```python
class EquipmentConfigurator:
    async def get_available_models(self, filters: Optional[ModelFilters] = None):
        """Retorna modelos disponíveis com preview de especificações"""
        models = await self.model_repository.list(filters)
        
        return [
            {
                "id": model.id,
                "display_name": f"{model.manufacturer} {model.model_name}",
                "family": model.model_family,
                "specifications_preview": model.technical_specifications,
                "has_manuals": await self.check_manuals_availability(model.id),
                "has_fmea": await self.check_fmea_availability(model.id)
            }
            for model in models
        ]
    
    async def get_compatible_sensors(self, model_id: UUID):
        """Retorna tipos de sensores compatíveis com o modelo"""
        model = await self.model_repository.get_by_id(model_id)
        
        compatible_sensors = await self.sensor_repository.get_compatible_with_model(
            model_specifications=model.technical_specifications
        )
        
        return [
            {
                "id": sensor.id,
                "display_name": f"{sensor.manufacturer} {sensor.type_name}",
                "measurement_type": sensor.measurement_type,
                "specifications": sensor.technical_specifications,
                "has_manuals": await self.check_sensor_manuals(sensor.id)
            }
            for sensor in compatible_sensors
        ]
```

## Pinecone Metadata com Herança:

```json
{
  "vector_id": "vec_equipment_123",
  "entity_type": "equipment",
  "entity_id": "eq_12345",
  "inheritance_links": [
    {
      "parent_vector_id": "vec_model_456",
      "relationship_type": "model_inheritance",
      "inherited_content": "technical_specifications, manuals, fmea"
    },
    {
      "parent_vector_id": "vec_sensor_789",
      "relationship_type": "sensor_inheritance", 
      "inherited_content": "calibration_procedures, troubleshooting"
    }
  ],
  "specific_content": {
    "serial_number": "SN12345",
    "installation_location": "Line 1 - Station 3",
    "maintenance_history": [...]
  },
  "created_at": "2025-09-08T10:00:00Z"
}
```

## Fluxo de Query RAG com Herança:

```python
# Agent query para equipamento específico
async def get_equipment_context(self, equipment_id: str) -> str:
    
    # Query que automaticamente inclui contexto herdado
    rag_query = f"""
    Buscar informações completas sobre equipamento {equipment_id}:
    - Especificações herdadas do modelo
    - Manuais herdados do modelo  
    - Procedimentos herdados dos tipos de sensores
    - Histórico específico deste equipamento
    - FMEA aplicável à família do modelo
    """
    
    # Uma única consulta retorna contexto completo (herdado + específico)
    context = await self.rag_service.query_with_inheritance(
        query=rag_query,
        equipment_id=equipment_id,
        include_inheritance=True,
        max_results=50
    )
    
    return context
```

## Métricas de Sucesso

### Eficiência de Cadastro:
- **Redução de Campos**: > 90% menos campos para preencher
- **Tempo de Cadastro**: < 2min para equipamento vs 20min antes
- **Erro Rate**: < 1% erro em especificações (vs 15% antes)
- **User Satisfaction**: > 95% satisfação com novo fluxo

### Consistência de Dados:
- **Specification Accuracy**: 100% consistência para mesmo modelo
- **Manual Completeness**: 100% equipamentos com manuais via herança
- **FMEA Coverage**: > 95% equipamentos com análises FMEA

### Storage Efficiency:
- **Duplicate Reduction**: > 85% redução de armazenamento duplicado
- **Vector Efficiency**: > 80% redução de embeddings redundantes
- **Processing Time**: > 70% redução no tempo de processamento RAG

## Interface de Usuário Simplificada

### Novo Fluxo de Cadastro:
1. **Selecionar Modelo**: Dropdown com preview de especificações
2. **Selecionar Sensores**: Multi-select de tipos compatíveis
3. **Preview de Herança**: Mostra o que será herdado automaticamente
4. **Dados Específicos**: Apenas serial, localização, configurações particulares
5. **Confirmação**: Submit cria equipamento + herança automática

### Dashboard de Modelos:
- Lista de modelos cadastrados com estatísticas de uso
- Equipamentos que herdam de cada modelo
- Status de processamento RAG por modelo
- Botão para atualizar modelo (propaga para todos equipamentos)

## Migração de Dados Existentes

### Estratégia de Migration:
1. **Análise de Patterns**: Identificar modelos/tipos existentes nos dados
2. **Model Extraction**: Extrair modelos únicos automaticamente
3. **Sensor Type Extraction**: Extrair tipos de sensores únicos
4. **Equipment Refactoring**: Refatorar equipamentos para usar herança
5. **RAG Reprocessing**: Reprocessar com nova estrutura de herança

### Migration Tools:
```sql
-- Script para identificar modelos únicos
SELECT 
    manufacturer, 
    model, 
    COUNT(*) as equipment_count,
    JSON_AGG(DISTINCT technical_specifications) as unique_specs
FROM equipment 
GROUP BY manufacturer, model
HAVING COUNT(*) > 1;
```

## Revisão Futura

Este ADR deve ser revisado quando:
1. **Q1 2026**: Avaliação de user satisfaction após 6 meses
2. **Q2 2026**: Análise de storage savings e performance gains
3. **Sempre**: Quando novos tipos de herança forem necessários

## Referências

- [ADR-001: Pipeline Sequencial Event-Driven](./ADR-001-pipeline-sequencial-trigger-configuravel.md)
- [ADR-002: Gestão Completa de Dados Mestre](./ADR-002-gestao-dados-mestre.md)
- [ADR-003: Arquitetura RAG-First com Deduplicação](./ADR-003-arquitetura-rag-first-deduplicacao.md)
- [C4 Container Diagram - Inheritance](../C4/Nexsora/c2-containers.puml)
- [C4 Component Diagram - Smart Linking](../C4/Nexsora/c3-componentes.puml)

---

**Relacionado:**
- ADR-001: Pipeline Sequencial Event-Driven
- ADR-002: Gestão Completa de Dados Mestre  
- ADR-003: Arquitetura RAG-First com Deduplicação
- ADR-005: Multi-tenant Isolation Strategy (Próximo)