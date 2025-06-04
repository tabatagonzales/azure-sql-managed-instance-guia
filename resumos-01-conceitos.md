# Conceitos Básicos - Azure SQL Managed Instance

Este documento apresenta os conceitos fundamentais sobre Azure SQL Managed Instance para estudantes de cloud computing.

## 🔍 O que é Azure SQL Managed Instance?

**Azure SQL Managed Instance** é um serviço de banco de dados em nuvem (PaaS - Platform as a Service) que oferece quase 100% de compatibilidade com o SQL Server on-premises, combinando os benefícios de uma plataforma totalmente gerenciada com a flexibilidade do SQL Server.

### Definição Técnica
É uma instância completa do SQL Server hospedada na nuvem Azure, que mantém compatibilidade total com recursos como SQL Server Agent, Database Mail, Linked Servers, CLR, Service Broker e muito mais.

## 🎯 Principais Características

### ✅ Compatibilidade Total
- **99% compatível** com SQL Server on-premises
- Suporte a recursos nativos do SQL Server:
  - SQL Server Agent para agendamento
  - Database Mail para envio de emails
  - Linked Servers para conexões externas
  - Common Language Runtime (CLR)
  - Service Broker para mensageria
  - Cross-database queries

### 🏗️ Arquitetura
- **Instância completa**: Não apenas databases individuais
- **Até 100 databases** por instância
- **Virtual Network (VNet)**: Isolamento total de rede
- **Cluster virtual dedicado**: Recursos não compartilhados

### 🔧 Gerenciamento Automático
- **Backups automáticos**: Point-in-time recovery
- **Patching automático**: Sistema operacional e SQL Server
- **Alta disponibilidade**: 99.99% SLA integrado
- **Monitoramento**: Telemetria e alertas nativos

## 📊 Quando Usar Azure SQL Managed Instance?

### ✅ Cenários Ideais

#### 1. **Migração Lift-and-Shift**
```
SQL Server On-Premises → Azure SQL Managed Instance
• Migração com mudanças mínimas de código
• Mantém funcionalidades específicas do SQL Server
• Reduz tempo e complexidade de migração
```

#### 2. **Aplicações Complexas**
- Sistemas que usam SQL Server Agent
- Aplicações com cross-database queries
- Sistemas que dependem de CLR assemblies
- Integração com outras tecnologias Microsoft

#### 3. **Requisitos de Compliance**
- Isolamento de rede através de VNet
- Controle granular de acesso
- Auditoria avançada
- Criptografia em trânsito e repouso

### ❌ Quando NÃO Usar

- **Aplicações simples**: Use Azure SQL Database
- **Cargas de trabalho pequenas**: Custo pode ser elevado
- **Acesso público direto**: Requer configuração especial
- **Latência crítica**: Considere SQL Server em VM

## 🏢 Tipos de Camadas de Serviço

### 📊 General Purpose (Uso Geral)
```
💰 Custo: Moderado
🔄 Disponibilidade: 99.99%
💾 Storage: Azure Premium Storage
🎯 Uso: Maioria das cargas de trabalho
```

**Características:**
- Storage separado da computação
- Baseado em Azure Premium Storage
- Até 16TB de armazenamento
- Throughput de I/O moderado

### 💼 Business Critical (Crítico para Negócios)
```
💰 Custo: Premium
🔄 Disponibilidade: 99.995%
💾 Storage: Local SSD
🎯 Uso: Aplicações críticas
```

**Características:**
- Storage local SSD de alta performance
- Réplica secundária para leitura
- Failover automático e transparente
- Maior throughput de I/O

## 🌐 Arquitetura de Conectividade

### 🔒 VNet-Local Endpoint
```
Formato: <instance-name>.<dns-zone>.database.windows.net
Porta: 1433
Acesso: Apenas dentro da VNet ou redes conectadas
```

### 🌍 Public Endpoint (Opcional)
```
Formato: <instance-name>.public.<dns-zone>.database.windows.net
Porta: 3342
Acesso: Internet pública (requer configuração)
```

### 🔗 Tipos de Conexão

#### Proxy (Padrão)
- Todo tráfego passa pelo gateway
- Maior latência, maior compatibilidade
- Suporta drivers antigos

#### Redirect (Recomendado)
- Conexão direta ao nó do database
- Menor latência, melhor performance
- Requer drivers modernos (TDS 7.4+)

## 💰 Modelo de Preços

### 🧮 Componentes de Custo

#### 1. **Computação (vCores)**
```
Modelos disponíveis:
• 4 vCores  (mínimo)
• 8 vCores
• 16 vCores
• 24 vCores
• 32 vCores
• 40 vCores
• 64 vCores
• 80 vCores (máximo)
```

#### 2. **Armazenamento**
```
General Purpose: 32GB - 16TB
Business Critical: 32GB - 4TB
Cobrança: Por GB usado
```

#### 3. **Backup**
```
Backup local: Incluído no preço base
Backup geo-redundante: Cobrança adicional
Retenção: 7-35 dias configurável
```

### 💡 Benefício Híbrido do Azure
- **Economia até 55%** ao usar licenças existentes
- Aplicável para Software Assurance
- Pode ser ativado/desativado conforme necessário

## 🔄 Diferenças Principais vs SQL Server On-Premises

### ✅ Disponível em Managed Instance
- SQL Server Agent ✅
- Database Mail ✅
- Linked Servers ✅
- CLR Assemblies ✅
- Cross-database queries ✅
- Service Broker ✅
- Replicação transacional ✅

### ❌ Limitações vs On-Premises
- FileStream/FileTable ❌
- Backup para URL local ❌
- Database mirroring ❌
- Instalação de extensões ❌
- Acesso ao sistema operacional ❌
- Configuração de hardware ❌

## 📈 Cenários de Migração

### 🎯 Estratégias de Migração

#### 1. **Database Migration Service (DMS)**
```
Método: Online migration
Downtime: Mínimo
Complexidade: Baixa
Suporte: Automático
```

#### 2. **Backup/Restore**
```
Método: Backup completo + restore
Downtime: Moderado
Complexidade: Média
Controle: Total
```

#### 3. **Log Shipping**
```
Método: Log shipping contínuo
Downtime: Baixo
Complexidade: Alta
Flexibilidade: Máxima
```

## 🔍 Monitoramento e Diagnóstico

### 📊 Métricas Principais
- **CPU utilization**: Uso de processador
- **Storage usage**: Utilização de armazenamento
- **IO statistics**: Estatísticas de I/O
- **Connection count**: Número de conexões
- **Query performance**: Performance de queries

### 🛠️ Ferramentas de Monitoramento
- **Azure Monitor**: Métricas e alertas
- **Query Store**: Análise de performance
- **DMVs**: Dynamic Management Views
- **Extended Events**: Monitoramento avançado
- **SQL Insights**: Monitoramento inteligente

## 🚀 Próximos Passos

1. 📖 [Comparação SQL Database vs Managed Instance](./02-comparacao-services.md)
2. 🏗️ [Tipos de Camadas de Serviço](./03-camadas-servico.md)
3. 💻 [Diferenças T-SQL](./04-diferencas-tsql.md)
4. 🛠️ [Tutorial Prático](../passo-a-passo/01-criando-instancia.md)

## 🔗 Referências

- [Documentação Oficial](https://learn.microsoft.com/pt-br/azure/azure-sql/managed-instance/sql-managed-instance-paas-overview?view=azuresql)
- [Arquitetura de Conectividade](https://learn.microsoft.com/pt-br/azure/azure-sql/managed-instance/connectivity-architecture-overview?view=azuresql)
- [Preços Oficiais](https://azure.microsoft.com/pt-br/pricing/details/azure-sql-managed-instance/)

---
📅 **Última atualização**: Junho 2025  
⬅️ [Voltar ao Índice](../README.md) | ➡️ [Próximo: Comparação de Serviços](./02-comparacao-services.md)