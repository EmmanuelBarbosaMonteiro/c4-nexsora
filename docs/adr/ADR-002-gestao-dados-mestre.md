# ADR-002: Gestão Completa de Dados Mestre para Efetividade dos Agentes AI

**Status**: Aceito  
**Data**: 2025-09-06  
**Autor**: Equipe de Arquitetura  

## Contexto

Os agentes AI da Plataforma SaaS Nexora precisam de dados estruturados e contextualizados para serem efetivos na manutenção prescritiva. Identificamos que sem uma gestão robusta de dados mestre, os agentes operariam com informações limitadas, reduzindo significativamente a qualidade das análises e recomendações.

### Dados Essenciais Identificados:

1. **Equipamentos**: Hierarquia, especificações técnicas, criticidade
2. **Sensores**: Tipos, limites operacionais, histórico de instalação
3. **Manuais e FMEA**: Procedimentos, análises de falha, conhecimento especializado
4. **Histórico de Manutenções**: Padrões de falha, ROI, correlações
5. **Processamento de Documentos**: Extração automática de conhecimento

## Decisão

Implementamos uma **arquitetura completa de gestão de dados mestre** com os seguintes componentes:

### 1. **Equipment Management Service**
- CRUD completo com hierarquia (Planta → Linha → Equipamento → Componente)
- Validação de regras de negócio
- Herança de configurações na hierarquia
- Gestão de criticidade e classificação

### 2. **Sensor Management Service**
- Registry global para reutilização de tipos de sensores
- Validação automática de compatibilidade e limites
- Histórico de performance por tipo de sensor
- Padronização de especificações técnicas

### 3. **Manual & Documentation Service**
- Upload seguro de PDFs e documentos técnicos
- Processamento automático com OCR e NLP
- Gestão de FMEA (Failure Mode Effect Analysis)
- Versionamento e controle de acesso

### 4. **Maintenance History Service**
- Registro completo de manutenções (preventiva, corretiva, preditiva)
- Análise de padrões de falha e correlações
- Cálculo automático de ROI e eficácia
- Trending de degradação de equipamentos

### 5. **Document Processing Pipeline**
- OCR para PDFs escaneados
- NER (Named Entity Recognition) para identificar peças/procedimentos
- Classificação automática de tipos de documento
- Geração de embeddings contextuais para RAG

## Alternativas Consideradas

### 1. **Integração Direta com ERP Existente**
- **Prós**: Aproveita dados já existentes, menos duplicação
- **Contras**: Dependência de estrutura legada, limitações de flexibilidade
- **Por que rejeitado**: ERPs não foram desenhados para AI, estrutura inadequada

### 2. **Dados Mestre Mínimos**
- **Prós**: Implementação mais rápida, menor complexidade inicial
- **Contras**: Agentes AI com contexto limitado, efetividade reduzida
- **Por que rejeitado**: Comprometeria a qualidade das análises preditivas

### 3. **Processamento Manual de Documentos**
- **Prós**: Controle total sobre qualidade dos dados
- **Contras**: Não escalável, custo operacional alto, lentidão
- **Por que rejeitado**: Inviável para SaaS com múltiplos tenants

## Consequências

### Positivas ✅

1. **Agentes AI Contextualizados**: Acesso a informações completas para análises precisas
2. **Reutilização de Conhecimento**: Sensor registry e manual processing permitem reuso
3. **Escalabilidade**: Processamento automático de documentos
4. **Qualidade de Dados**: Validação automática e regras de negócio
5. **ROI Mensurável**: Tracking completo de eficácia das manutenções
6. **Knowledge Evolution**: Base de conhecimento que melhora continuamente

### Negativas ❌

1. **Complexidade Inicial**: Mais componentes para desenvolver e manter
2. **Curva de Aprendizado**: Usuários precisam entender a importância dos dados mestre
3. **Migração de Dados**: Complexidade para importar dados existentes
4. **Storage Costs**: Armazenamento de documentos e embeddings

### Neutras 🟡

1. **Time to Market**: Maior tempo inicial, mas melhor produto final
2. **User Experience**: Mais configuração inicial, melhor experiência de uso

## Detalhes de Implementação

### Modelo de Dados de Equipamentos:

```sql
-- Equipment Hierarchy
CREATE TABLE plants (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    location VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE production_lines (
    id UUID PRIMARY KEY,
    plant_id UUID REFERENCES plants(id),
    name VARCHAR(255) NOT NULL,
    criticality ENUM('LOW', 'MEDIUM', 'HIGH', 'CRITICAL')
);

CREATE TABLE equipment (
    id UUID PRIMARY KEY,
    line_id UUID REFERENCES production_lines(id),
    name VARCHAR(255) NOT NULL,
    manufacturer VARCHAR(255),
    model VARCHAR(255),
    serial_number VARCHAR(255),
    criticality ENUM('LOW', 'MEDIUM', 'HIGH', 'CRITICAL'),
    operational_limits JSONB,
    maintenance_plan_id UUID,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Sensor Registry Schema:

```sql
CREATE TABLE sensor_types (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    measurement_type ENUM('TEMPERATURE', 'VIBRATION', 'PRESSURE', 'FLOW'),
    unit VARCHAR(50),
    min_value DECIMAL,
    max_value DECIMAL,
    accuracy DECIMAL,
    specifications JSONB
);

CREATE TABLE sensor_installations (
    id UUID PRIMARY KEY,
    equipment_id UUID REFERENCES equipment(id),
    sensor_type_id UUID REFERENCES sensor_types(id),
    installation_date DATE,
    position VARCHAR(255),
    calibration_data JSONB,
    operational_limits JSONB
);
```

### Document Processing Pipeline:

```python
# Exemplo de processamento automático
class DocumentProcessor:
    def process_manual(self, pdf_path: str, equipment_id: str):
        # 1. OCR e extração de texto
        text = self.ocr_processor.extract_text(pdf_path)
        
        # 2. NER para identificar entidades
        entities = self.ner_model.extract_entities(text)
        
        # 3. Classificação do documento
        doc_type = self.classifier.classify(text)
        
        # 4. Geração de embeddings contextuais
        embeddings = self.embedding_model.generate(
            text, context={"equipment_id": equipment_id}
        )
        
        # 5. Indexação no vector database
        self.vector_db.store(embeddings, metadata={
            "equipment_id": equipment_id,
            "document_type": doc_type,
            "entities": entities
        })
```

### FMEA Integration:

```sql
CREATE TABLE fmea_analyses (
    id UUID PRIMARY KEY,
    equipment_id UUID REFERENCES equipment(id),
    failure_mode VARCHAR(255),
    failure_effects TEXT,
    severity INTEGER CHECK (severity BETWEEN 1 AND 10),
    occurrence INTEGER CHECK (occurrence BETWEEN 1 AND 10),
    detection INTEGER CHECK (detection BETWEEN 1 AND 10),
    rpn INTEGER GENERATED ALWAYS AS (severity * occurrence * detection),
    recommended_actions TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

## Integração com Agentes AI

### Agent Contextualization:

1. **Anomaly Detection Agent**:
   - Consulta especificações do equipamento para limites normais
   - Verifica histórico de falhas similares
   - Considera criticidade do equipamento na análise

2. **Predictive Analysis Agent**:
   - Utiliza histórico de manutenções para modelos preditivos
   - Correlaciona com FMEA para identificar modos de falha prováveis
   - Considera especificações técnicas na análise de degradação

3. **Maintenance Planning Agent**:
   - Busca procedimentos específicos nos manuais processados
   - Consulta FMEA para ações recomendadas
   - Considera hierarquia de equipamentos para planejamento

4. **Work Order Agent**:
   - Acessa especificações completas do equipamento
   - Registra nova manutenção no histórico
   - Calcula ROI esperado baseado em dados históricos

## Métricas de Sucesso

### Qualidade dos Dados:
- **Data Completeness**: > 95% dos equipamentos com dados completos
- **Data Accuracy**: < 2% de erro em especificações técnicas
- **Document Processing**: > 90% de precisão no OCR e NER

### Efetividade dos Agentes:
- **Prediction Accuracy**: > 85% de precisão nas predições
- **False Positives**: < 15% de falsos positivos em anomalias
- **Maintenance ROI**: > 25% de melhoria no ROI das manutenções

### Performance do Sistema:
- **Data Ingestion**: < 100ms para validação de sensor
- **Document Processing**: < 2min para processar manual de 50 páginas
- **Agent Response**: < 30s para análise contextualizada

## Migração e Onboarding

### Estratégia de Migração:
1. **Fase 1**: Importação de dados básicos de equipamentos
2. **Fase 2**: Upload e processamento de manuais críticos
3. **Fase 3**: Migração de histórico de manutenções
4. **Fase 4**: Configuração de FMEA e sensor registry

### Data Import Tools:
- Importadores automáticos para ERPs comuns (SAP, Oracle)
- Templates Excel para entrada manual de dados
- APIs REST para integração com sistemas existentes
- Validação automática durante importação

## Monitoramento e Evolução

### KPIs de Dados Mestre:
- **Equipment Coverage**: % de equipamentos com dados completos
- **Manual Processing Rate**: Documentos processados por dia
- **Knowledge Base Growth**: Crescimento da base de conhecimento
- **Data Quality Score**: Score agregado de qualidade dos dados

### Feedback Loop:
- Agentes AI reportam lacunas de informação
- Usuários podem corrigir/enriquecer dados via interface
- Análise automática de gaps de conhecimento
- Sugestões de melhoria baseadas em performance

## Revisão Futura

Este ADR deve ser revisado quando:
1. **Q1 2026**: Avaliação da efetividade dos agentes com dados completos
2. **Q2 2026**: Análise de ROI do investimento em dados mestre
3. **Sempre**: Quando novos tipos de equipamento/sensores forem suportados

## Referências

- [Equipment Management API Specification](../api/equipment.yaml)
- [Document Processing Pipeline](../docs/document-processing.md)
- [FMEA Integration Guide](../docs/fmea-integration.md)
- [Data Migration Playbook](../docs/data-migration.md)

---

**Relacionado:**
- ADR-001: Pipeline Sequencial Event-Driven
- ADR-003: Vector Database Strategy (Próximo)
- ADR-004: Multi-tenant Data Isolation (Próximo)