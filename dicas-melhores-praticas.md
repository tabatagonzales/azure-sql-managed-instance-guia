# Melhores Práticas - Azure SQL Managed Instance

Este documento apresenta as melhores práticas para implementação, configuração e operação do Azure SQL Managed Instance.

## 🏗️ Planejamento e Arquitetura

### 📐 Dimensionamento Inicial

#### **Análise de Requisitos**
```
1. Mapear Cargas de Trabalho:
   • Número de databases
   • Pico de transações (TPS)
   • Necessidades de CPU e memória
   • Requisitos de I/O e storage

2. Estimativa de Crescimento:
   • Taxa de crescimento anual
   • Sazonalidade do negócio
   • Expansão geográfica planejada
```

#### **Estratégia de Sizing**
```sql
-- Baseline para Dimensionamento
-- Execute no SQL Server on-premises atual:

-- 1. Verificar uso de CPU
SELECT 
    record_id,
    DATEADD(ms, -1 * (ts_now - [timestamp]), GETDATE()) AS EventTime,
    100-SystemIdle AS CPUUsage,
    SQLProcessUtilization
FROM (
    SELECT 
        record.value('(./Record/@id)[1]', 'int') AS record_id,
        record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') AS SystemIdle,
        record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int') AS SQLProcessUtilization,
        [timestamp], ts_now
    FROM (
        SELECT [timestamp], CONVERT(xml, record) AS record, cpu_ticks/(cpu_ticks/ms_ticks) AS ts_now
        FROM sys.dm_os_ring_buffers 
        CROSS JOIN sys.dm_os_sys_info
        WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
        AND record LIKE '%<SystemHealth>%'
    ) AS x
) AS y 
ORDER BY record_id DESC;

-- 2. Verificar uso de memória
SELECT 
    total_physical_memory_kb/1024 AS [Physical Memory (MB)],
    available_physical_memory_kb/1024 AS [Available Memory (MB)],
    total_page_file_kb/1024 AS [Total Page File (MB)],
    available_page_file_kb/1024 AS [Available Page File (MB)],
    system_memory_state_desc AS [System Memory State]
FROM sys.dm_os_sys_memory;
```

### 🌐 Arquitetura de Rede

#### **Configuração de VNet Recomendada**
```yaml
Virtual Network:
  Address Space: 10.0.0.0/16
  
Subnets:
  - Name: subnet-sqlmi
    Address: 10.0.1.0/24
    Delegation: Microsoft.Sql/managedInstances
    Min IPs: 32 (/27 ou maior)
    
  - Name: subnet-apps  
    Address: 10.0.2.0/24
    Purpose: Application servers
    
  - Name: subnet-mgmt
    Address: 10.0.3.0/24
    Purpose: Management and monitoring
```

#### **Route Table Essencial**
```bash
# Configurar Route Table para Managed Instance
az network route-table create \
    --resource-group rg-sqlmi \
    --name rt-sqlmi

# Associar à subnet
az network vnet subnet update \
    --resource-group rg-sqlmi \
    --vnet-name vnet-sqlmi \
    --name subnet-sqlmi \
    --route-table rt-sqlmi
```

### 🔒 Segurança por Design

#### **Princípios de Segurança**
```
1. Zero Trust Network:
   • Isolamento total via VNet
   • NSGs restritivos por padrão
   • Endpoint público desabilitado

2. Identidade e Acesso:
   • Azure AD como padrão
   • SQL Auth apenas para serviços
   • Princípio do menor privilégio

3. Proteção de Dados:
   • TDE habilitado sempre
   • Backup encryption
   • Always Encrypted para dados sensíveis
```

## 🚀 Implementação e Configuração

### ⚙️ Configurações Essenciais Pós-Criação

#### **1. Configurar Administrador Azure AD**
```powershell
# PowerShell - Configurar Azure AD admin
Set-AzSqlInstanceActiveDirectoryAdministrator `
    -ResourceGroupName "rg-sqlmi" `
    -InstanceName "mi-empresa" `
    -DisplayName "DBA-Team" `
    -ObjectId "12345678-1234-1234-1234-123456789012"
```

#### **2. Configurar Auditoria**
```sql
-- Habilitar auditoria na instância
ALTER SERVER AUDIT SPECIFICATION [ServerAuditSpecification-Managed]
WITH (STATE = ON);

-- Configurar auditoria de database
CREATE DATABASE AUDIT SPECIFICATION [DatabaseAuditSpecification]
FOR SERVER AUDIT [Managed_Audit]
ADD (INSERT, UPDATE, DELETE ON [dbo].[TabelaSensivel] BY [dbo]),
ADD (SELECT ON [dbo].[TabelaSensivel] BY [dbo])
WITH (STATE = ON);
```

#### **3. Configurar Transparent Data Encryption**
```sql
-- Verificar status TDE (já habilitado por padrão)
SELECT 
    name,
    is_encrypted,
    encryption_state_desc,
    percent_complete
FROM sys.dm_database_encryption_keys dek
JOIN sys.databases db ON dek.database_id = db.database_id;

-- Para usar chave gerenciada pelo cliente (opcional)
-- Configure Key Vault e Customer Managed Key no Portal Azure
```

### 📊 Monitoramento e Alertas

#### **Métricas Críticas para Monitorar**
```yaml
CPU Utilization:
  Threshold: > 80%
  Action: Alert + Auto-scale

Memory Usage:
  Threshold: > 85%
  Action: Alert + Investigation

Storage Usage:
  Threshold: > 90%
  Action: Alert + Storage expansion

Connection Count:
  Threshold: > 80% of limit
  Action: Alert + Connection analysis

Blocking Processes:
  Threshold: > 30 seconds
  Action: Alert + Kill process

Failed Logins:
  Threshold: > 10 per minute
  Action: Alert + Security review
```

#### **Script de Monitoramento Automatizado**
```sql
-- Criar stored procedure para monitoramento
CREATE PROCEDURE [dbo].[sp_HealthCheck]
AS
BEGIN
    -- 1. Verificar CPU
    DECLARE @cpu_percent FLOAT;
    SELECT TOP 1 @cpu_percent = avg_cpu_percent 
    FROM sys.dm_db_resource_stats 
    ORDER BY end_time DESC;
    
    IF @cpu_percent > 80
        PRINT 'WARNING: High CPU usage detected: ' + CAST(@cpu_percent AS VARCHAR(10)) + '%';
    
    -- 2. Verificar conexões
    DECLARE @connection_count INT;
    SELECT @connection_count = COUNT(*) 
    FROM sys.dm_exec_sessions 
    WHERE is_user_process = 1;
    
    IF @connection_count > 100
        PRINT 'WARNING: High connection count: ' + CAST(@connection_count AS VARCHAR(10));
    
    -- 3. Verificar locks
    SELECT 
        r.session_id,
        r.blocking_session_id,
        r.wait_type,
        r.wait_time,
        t.text
    FROM sys.dm_exec_requests r
    LEFT JOIN sys.dm_exec_sql_text(r.sql_handle) t ON 1=1
    WHERE r.blocking_session_id <> 0;
    
    -- 4. Verificar storage
    SELECT 
        DB_NAME(database_id) as database_name,
        SUM(size * 8.0 / 1024) as size_mb,
        SUM(CASE WHEN max_size = -1 THEN 0 ELSE max_size * 8.0 / 1024 END) as max_size_mb
    FROM sys.master_files
    WHERE database_id > 4
    GROUP BY database_id;
END;
```

## 🎯 Performance e Otimização

### 📈 Tuning de Performance

#### **Query Store - Configuração Recomendada**
```sql
-- Habilitar Query Store em todos os databases
ALTER DATABASE [YourDatabase] SET QUERY_STORE = ON;

-- Configurações otimizadas
ALTER DATABASE [YourDatabase] SET QUERY_STORE (
    OPERATION_MODE = READ_WRITE,
    CLEANUP_POLICY = (STALE_QUERY_THRESHOLD_DAYS = 30),
    DATA_FLUSH_INTERVAL_SECONDS = 900,
    INTERVAL_LENGTH_MINUTES = 60,
    MAX_STORAGE_SIZE_MB = 1000,
    QUERY_CAPTURE_MODE = AUTO,
    SIZE_BASED_CLEANUP_MODE = AUTO
);
```

#### **Índices - Melhores Práticas**
```sql
-- Script para identificar índices ausentes críticos
SELECT 
    migs.avg_total_user_cost * (migs.avg_user_impact / 100.0) * (migs.user_seeks + migs.user_scans) AS improvement_measure,
    'CREATE INDEX [IX_' + OBJECT_NAME(mid.object_id) + '_' + 
    REPLACE(REPLACE(REPLACE(ISNULL(mid.equality_columns,''), ', ', '_'), '[', ''), ']', '') +
    CASE WHEN mid.inequality_columns IS NOT NULL THEN '_' + 
    REPLACE(REPLACE(REPLACE(mid.inequality_columns, ', ', '_'), '[', ''), ']', '') ELSE '' END + ']' +
    ' ON ' + mid.statement + 
    ' (' + ISNULL(mid.equality_columns,'') + 
    CASE WHEN mid.inequality_columns IS NOT NULL AND mid.equality_columns IS NOT NULL THEN ',' ELSE '' END +
    CASE WHEN mid.inequality_columns IS NOT NULL THEN mid.inequality_columns ELSE '' END + ')' +
    CASE WHEN mid.included_columns IS NOT NULL THEN ' INCLUDE (' + mid.included_columns + ')' ELSE '' END AS create_index_statement,
    migs.*,
    mid.database_id,
    mid.[object_id]
FROM sys.dm_db_missing_index_groups mig
INNER JOIN sys.dm_db_missing_index_group_stats migs ON migs.group_handle = mig.index_group_handle
INNER JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
WHERE migs.avg_total_user_cost * (migs.avg_user_impact / 100.0) * (migs.user_seeks + migs.user_scans) > 10
ORDER BY migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans) DESC;
```

#### **Manutenção Automática**
```sql
-- Job de manutenção para executar via SQL Agent
-- Criar no master database

-- 1. Update Statistics
EXEC sp_add_job 
    @job_name = 'Update Statistics Weekly',
    @enabled = 1,
    @description = 'Update statistics for all user databases';

EXEC sp_add_jobstep 
    @job_name = 'Update Statistics Weekly',
    @step_name = 'Update Stats All DBs',
    @command = 'EXEC sp_MSforeachdb "IF ''?'' NOT IN (''master'', ''model'', ''msdb'', ''tempdb'') BEGIN USE [?]; EXEC sp_updatestats; END"';

-- 2. Rebuild Indexes
EXEC sp_add_job 
    @job_name = 'Rebuild Indexes Monthly',
    @enabled = 1,
    @description = 'Rebuild fragmented indexes';

EXEC sp_add_jobstep 
    @job_name = 'Rebuild Indexes Monthly',
    @step_name = 'Rebuild Fragmented Indexes',
    @command = '
DECLARE @sql NVARCHAR(MAX);
SELECT @sql = STRING_AGG(
    ''ALTER INDEX '' + QUOTENAME(i.name) + '' ON '' + QUOTENAME(SCHEMA_NAME(t.schema_id)) + ''.'' + QUOTENAME(t.name) + '' REBUILD;'',
    CHAR(13)
)
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, ''LIMITED'') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
JOIN sys.tables t ON i.object_id = t.object_id
WHERE ips.avg_fragmentation_in_percent > 30
AND i.index_id > 0;

EXEC sp_executesql @sql;';
```

## 💰 Gestão de Custos

### 💡 Estratégias de Otimização de Custos

#### **1. Benefício Híbrido do Azure**
```powershell
# Habilitar Azure Hybrid Benefit (até 55% de economia)
Set-AzSqlInstance `
    -ResourceGroupName "rg-sqlmi" `
    -Name "mi-empresa" `
    -LicenseType "BasePrice"  # Ou "LicenseIncluded" para desabilitar
```

#### **2. Escalamento Inteligente**
```yaml
Estratégias de Scaling:
  
  Horário Comercial:
    vCores: 16
    Schedule: 8h-18h (Mon-Fri)
    
  Fora do Horário:
    vCores: 8  
    Schedule: 18h-8h + Weekends
    
  Desenvolvimento/Teste:
    vCores: 4
    Schedule: Business hours only
    Auto-shutdown: After hours
```

#### **3. Storage Optimization**
```sql
-- Identificar databases subutilizados
SELECT 
    name as database_name,
    size * 8.0 / 1024 as size_mb,
    FILEPROPERTY(name, 'SpaceUsed') * 8.0 / 1024 as used_mb,
    (size - FILEPROPERTY(name, 'SpaceUsed')) * 8.0 / 1024 as free_mb,
    CAST((FILEPROPERTY(name, 'SpaceUsed') * 100.0 / size) AS DECIMAL(5,2)) as used_percent
FROM sys.master_files
WHERE database_id > 4 AND type = 0
ORDER BY free_mb DESC;

-- Shrink databases com muito espaço livre (use com cuidado)
-- DBCC SHRINKDATABASE('DatabaseName', 10); -- Deixar 10% de free space
```

## 🔄 Backup e Disaster Recovery

### 📋 Estratégia de Backup

#### **Configurações Recomendadas**
```sql
-- Verificar configurações de backup atual
SELECT 
    database_name,
    backup_start_date,
    backup_finish_date,
    backup_size_mb = backup_size/1024/1024,
    compressed_backup_size_mb = compressed_backup_size/1024/1024,
    compression_ratio = compressed_backup_size/backup_size
FROM msdb.dbo.backupset
WHERE database_name NOT IN ('master', 'model', 'msdb')
ORDER BY backup_start_date DESC;

-- Configurar retenção de backup (via Portal ou PowerShell)
```

#### **Copy-Only Backups para Compliance**
```sql
-- Backup copy-only para atender compliance
BACKUP DATABASE [ProductionDB] 
TO URL = 'https://storageaccount.blob.core.windows.net/backups/ProductionDB_COPY_ONLY.bak'
WITH COPY_ONLY, 
     COMPRESSION,
     INIT,
     CREDENTIAL = 'AzureStorageCredential';
```

### 🌍 Geo-Redundancy

#### **Failover Groups - Configuração**
```powershell
# Criar failover group
$failoverGroup = New-AzSqlDatabaseFailoverGroup `
    -ResourceGroupName "rg-sqlmi-primary" `
    -ServerName "mi-empresa-primary" `
    -PartnerResourceGroupName "rg-sqlmi-secondary" `
    -PartnerServerName "mi-empresa-secondary" `
    -FailoverGroupName "fg-empresa-dr" `
    -FailoverPolicy Automatic `
    -GracePeriodWithDataLossHours 1

# Adicionar databases ao failover group
Add-AzSqlDatabaseToFailoverGroup `
    -ResourceGroupName "rg-sqlmi-primary" `
    -ServerName "mi-empresa-primary" `
    -FailoverGroupName "fg-empresa-dr" `
    -Database "ProductionDB"
```

## 🚨 Troubleshooting Proativo

### 🔍 Monitoramento Contínuo

#### **Alertas Críticos**
```yaml
Critical Alerts:
  - CPU > 90% for 15 minutes
  - Memory > 95% for 10 minutes  
  - Storage > 95%
  - Failed logins > 20 per minute
  - Blocking > 60 seconds
  - Deadlocks > 5 per hour

Warning Alerts:
  - CPU > 75% for 30 minutes
  - Storage > 85%
  - Connection count > 80% of limit
  - Long running queries > 30 minutes
```

#### **Script de Health Check Diário**
```sql
-- Executar diariamente via SQL Agent
CREATE PROCEDURE [dbo].[sp_DailyHealthCheck]
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @body NVARCHAR(MAX) = '';
    
    -- Verificar espaço em disco
    SET @body += '<h3>Storage Usage</h3><table border="1">';
    SET @body += '<tr><th>Database</th><th>Size (MB)</th><th>Used (%)</th></tr>';
    
    -- Adicionar dados de storage...
    
    -- Verificar performance
    SET @body += '<h3>Performance Metrics</h3>';
    
    -- Verificar backups
    SET @body += '<h3>Backup Status</h3>';
    
    -- Enviar email se houver problemas
    IF @body LIKE '%WARNING%' OR @body LIKE '%ERROR%'
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'Default',
            @recipients = 'dba@empresa.com',
            @subject = 'SQL MI Daily Health Check',
            @body = @body,
            @body_format = 'HTML';
    END
END;
```

## 📚 Recursos e Próximos Passos

### 🔗 Links Úteis
- [Azure SQL MI Best Practices](https://docs.microsoft.com/azure/azure-sql/managed-instance/management-best-practices)
- [Performance Tuning Guide](https://docs.microsoft.com/azure/azure-sql/managed-instance/performance-guidance)
- [Security Best Practices](https://docs.microsoft.com/azure/azure-sql/database/security-best-practice)

### 📖 Material Complementar
- [Troubleshooting Comum](./03-troubleshooting.md)
- [Comandos Úteis](./01-comandos-uteis.md)
- [Monitoramento](./02-monitoramento.md)

---
📅 **Última atualização**: Junho 2025  
⬅️ [Voltar ao Índice](../README.md)