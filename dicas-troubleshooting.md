# Troubleshooting Comum - Azure SQL Managed Instance

Este guia apresenta soluções para os problemas mais frequentes encontrados ao trabalhar com Azure SQL Managed Instance.

## 🔍 Diagnóstico Inicial

### 🛠️ Verificador de Conectividade Azure SQL
A Microsoft fornece uma ferramenta oficial para diagnosticar problemas:
```
URL: https://github.com/Azure/SQL-Connectivity-Checker
Uso: Download e execute o script PowerShell
Funcionalidade: Detecta automaticamente problemas de conectividade
```

### 📋 Checklist de Problemas Comuns
- [ ] Conectividade de rede
- [ ] Configurações de firewall
- [ ] Credenciais de autenticação
- [ ] Configurações de DNS
- [ ] Problemas de performance
- [ ] Limitações de recursos

## 🌐 Problemas de Conectividade

### ❌ **Erro**: "Cannot connect to server"

#### 🔍 **Diagnóstico**
```sql
-- Verificar se a instância está online
SELECT @@SERVERNAME, @@VERSION, GETDATE()
```

#### ✅ **Soluções**

**1. Verificar Status da Instância**
```powershell
# PowerShell
Get-AzSqlInstance -ResourceGroupName "rg-name" -Name "instance-name"

# Azure CLI  
az sql mi show --resource-group rg-name --name instance-name --query "state"
```

**2. Verificar Conectividade de Rede**
```bash
# Testar conectividade básica
telnet mi-name.1234567890123.database.windows.net 1433

# Usar nslookup para verificar DNS
nslookup mi-name.1234567890123.database.windows.net
```

**3. Validar String de Conexão**
```csharp
// Formato correto para SQL Authentication
"Server=mi-name.1234567890123.database.windows.net;Database=mydb;User Id=sqladmin;Password=mypassword;Encrypt=true;TrustServerCertificate=false;"

// Formato para Azure AD Authentication  
"Server=mi-name.1234567890123.database.windows.net;Database=mydb;Authentication=Active Directory Password;User Id=user@domain.com;Password=mypassword;"
```

### ❌ **Erro**: "Login failed for user"

#### ✅ **Soluções**

**1. Verificar Credenciais**
```sql
-- Conectar como admin e verificar logins
SELECT name, type_desc, is_disabled, create_date 
FROM sys.sql_logins 
WHERE name = 'your-username'

-- Verificar usuários do database
SELECT name, type_desc, authentication_type_desc
FROM sys.database_principals
WHERE type IN ('S', 'U', 'G')
```

**2. Reset de Senha (Via Portal Azure)**
```
1. Portal Azure → SQL managed instances
2. Selecionar instância → Reset password
3. Inserir nova senha → Confirmar
```

**3. Verificar Azure AD Configuration**
```sql
-- Verificar se Azure AD está configurado
SELECT name FROM sys.sql_logins WHERE type = 'E'
```

### ❌ **Erro**: "Connection timeout expired"

#### 🔍 **Diagnóstico**
```sql
-- Verificar conexões ativas
SELECT 
    session_id,
    login_name,
    host_name,
    program_name,
    last_request_start_time,
    status
FROM sys.dm_exec_sessions 
WHERE is_user_process = 1
```

#### ✅ **Soluções**

**1. Aumentar Connection Timeout**
```csharp
// Em .NET
connectionString += "Connection Timeout=30;"

// Em applications
// Configurar timeout para pelo menos 30 segundos
```

**2. Verificar Bloqueios (Locks)**
```sql
-- Identificar processos bloqueados
SELECT 
    r.session_id,
    r.blocking_session_id,
    r.wait_type,
    r.wait_time,
    r.wait_resource,
    t.text as query_text
FROM sys.dm_exec_requests r
LEFT JOIN sys.dm_exec_sql_text(r.sql_handle) t ON 1=1
WHERE r.blocking_session_id <> 0
```

## 🔥 Problemas de Performance

### ❌ **Problema**: CPU Alto Constante

#### 🔍 **Diagnóstico**
```sql
-- Top queries por CPU
SELECT TOP 10
    qs.total_worker_time/qs.execution_count as avg_cpu_time,
    qs.execution_count,
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(qt.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2)+1) as query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY qs.total_worker_time/qs.execution_count DESC
```

#### ✅ **Soluções**

**1. Otimização de Queries**
```sql
-- Ativar Query Store (se não estiver ativo)
ALTER DATABASE [YourDB] SET QUERY_STORE = ON

-- Verificar estatísticas desatualizadas
SELECT 
    SCHEMA_NAME(o.schema_id) + '.' + o.name as table_name,
    s.name as stat_name,
    s.stats_id,
    sp.last_updated,
    sp.rows,
    sp.modification_counter
FROM sys.stats s
JOIN sys.objects o ON s.object_id = o.object_id
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE sp.modification_counter > sp.rows * 0.2
    AND o.type = 'U'
ORDER BY sp.modification_counter DESC
```

**2. Escalamento de Recursos**
```powershell
# Aumentar vCores temporariamente
Set-AzSqlInstance `
    -ResourceGroupName "rg-name" `
    -Name "instance-name" `
    -VCore 8  # ou o número desejado
```

### ❌ **Problema**: I/O Alto / Storage Slow

#### 🔍 **Diagnóstico**
```sql
-- Verificar I/O por database
SELECT 
    DB_NAME(database_id) as database_name,
    SUM(num_of_reads + num_of_writes) as total_io,
    SUM(io_stall) as total_io_wait_ms,
    SUM(io_stall)/SUM(num_of_reads + num_of_writes) as avg_io_wait_ms
FROM sys.dm_io_virtual_file_stats(NULL, NULL)
GROUP BY database_id
ORDER BY total_io_wait_ms DESC
```

#### ✅ **Soluções**

**1. Identificar Queries com Alto I/O**
```sql
-- Top queries por I/O lógico
SELECT TOP 10
    qs.total_logical_reads/qs.execution_count as avg_logical_reads,
    qs.total_logical_writes/qs.execution_count as avg_logical_writes,
    qs.execution_count,
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(qt.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2)+1) as query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY qs.total_logical_reads/qs.execution_count DESC
```

**2. Otimizar Índices**
```sql
-- Identificar índices ausentes
SELECT 
    mid.database_id,
    mid.object_id,
    mid.statement as table_name,
    migs.avg_total_user_cost,
    migs.avg_user_impact,
    migs.user_seeks,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns
FROM sys.dm_db_missing_index_groups mig
JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
WHERE migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans) > 10000
ORDER BY migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans) DESC
```

## 🚨 Problemas de Recurso/Limite

### ❌ **Erro**: "Resource limit exceeded"

#### 🔍 **Diagnóstico**
```sql
-- Verificar utilização de recursos
SELECT 
    end_time,
    avg_cpu_percent,
    avg_data_io_percent,
    avg_log_write_percent,
    max_worker_percent,
    max_session_percent
FROM sys.dm_db_resource_stats
ORDER BY end_time DESC
```

#### ✅ **Soluções**

**1. Monitorar Limites Atuais**
```sql
-- Verificar configuração atual
SELECT 
    @@SERVERNAME as instance_name,
    edition,
    service_objective,
    elastic_pool_name
FROM sys.database_service_objectives
```

**2. Escalamento Vertical**
```powershell
# Escalar para mais vCores
Set-AzSqlInstance `
    -ResourceGroupName "rg-name" `
    -Name "instance-name" `
    -VCore 16 `
    -StorageSizeInGB 1024
```

### ❌ **Problema**: "Out of Memory"

#### 🔍 **Diagnóstico**
```sql
-- Verificar uso de memória
SELECT 
    type,
    name,
    pages_kb,
    pages_kb/1024.0 as pages_mb
FROM sys.dm_os_memory_clerks
WHERE pages_kb > 1024
ORDER BY pages_kb DESC
```

#### ✅ **Soluções**

**1. Otimizar Planos de Execução**
```sql
-- Limpar cache de planos (use com cuidado)
DBCC FREEPROCCACHE

-- Atualizar estatísticas
EXEC sp_updatestats
```

**2. Configurar Memory Settings**
```sql
-- Verificar configuração de memória (somente leitura no MI)
SELECT name, value, value_in_use, description
FROM sys.configurations
WHERE name LIKE '%memory%'
```

## 🔧 Problemas de Configuração

### ❌ **Erro**: "Subnet not properly configured"

#### ✅ **Soluções**

**1. Verificar Delegação de Subnet**
```powershell
# Verificar configuração da subnet
Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name $subnetName
```

**2. Validar Route Table**
```bash
# Azure CLI - verificar route table
az network route-table route list --resource-group rg-name --route-table-name rt-sqlmi
```

### ❌ **Problema**: DNS Resolution Failing

#### ✅ **Soluções**

**1. Configurar DNS Personalizado**
```powershell
# Configurar DNS na VNet
$vnet = Get-AzVirtualNetwork -ResourceGroupName "rg-name" -Name "vnet-name"
$vnet.DhcpOptions.DnsServers = @("168.63.129.16", "8.8.8.8")
Set-AzVirtualNetwork -VirtualNetwork $vnet
```

**2. Verificar Private DNS Zone**
```bash
# Listar private DNS zones
az network private-dns zone list --resource-group rg-name
```

## 📊 Comandos de Diagnóstico Avançado

### 🔍 Script de Diagnóstico Completo

```sql
-- Script de diagnóstico geral
SET NOCOUNT ON

PRINT '=== AZURE SQL MANAGED INSTANCE DIAGNOSTIC ==='
PRINT 'Instance: ' + @@SERVERNAME
PRINT 'Version: ' + @@VERSION
PRINT 'Current Time: ' + CONVERT(varchar, GETDATE(), 120)
PRINT ''

-- Verificar databases
PRINT '=== DATABASES ==='
SELECT name, database_id, state_desc, compatibility_level
FROM sys.databases
WHERE database_id > 4
PRINT ''

-- Verificar conexões ativas
PRINT '=== ACTIVE CONNECTIONS ==='
SELECT 
    COUNT(*) as total_connections,
    COUNT(CASE WHEN status = 'running' THEN 1 END) as running_sessions,
    COUNT(CASE WHEN status = 'sleeping' THEN 1 END) as sleeping_sessions
FROM sys.dm_exec_sessions
WHERE is_user_process = 1
PRINT ''

-- Verificar recursos
PRINT '=== RESOURCE USAGE (Last 30 minutes) ==='
SELECT TOP 6
    end_time,
    avg_cpu_percent,
    avg_data_io_percent,
    avg_log_write_percent,
    max_worker_percent
FROM sys.dm_db_resource_stats
ORDER BY end_time DESC
PRINT ''

-- Verificar waits
PRINT '=== TOP WAIT TYPES ==='
SELECT TOP 10
    wait_type,
    waiting_tasks_count,
    wait_time_ms,
    max_wait_time_ms,
    signal_wait_time_ms,
    wait_time_ms - signal_wait_time_ms as resource_wait_time_ms
FROM sys.dm_os_wait_stats
WHERE wait_time_ms > 0
    AND wait_type NOT IN ('CLR_SEMAPHORE', 'LAZYWRITER_SLEEP', 'RESOURCE_QUEUE', 
                         'SLEEP_TASK', 'SLEEP_SYSTEMTASK', 'WAITFOR', 'HADR_FILESTREAM_IOMGR_IOCOMPLETION')
ORDER BY wait_time_ms DESC
```

## 🆘 Contatos de Suporte

### 📞 Canais Oficiais Microsoft
- **Suporte Azure**: Portal Azure → "Ajuda + suporte"
- **Documentação**: [docs.microsoft.com/azure](https://docs.microsoft.com/azure)
- **Microsoft Q&A**: [docs.microsoft.com/answers](https://docs.microsoft.com/answers)
- **Stack Overflow**: Tag `azure-sql-managed-instance`

### 🔗 Ferramentas Úteis
- [SQL Connectivity Checker](https://github.com/Azure/SQL-Connectivity-Checker)
- [Azure Resource Explorer](https://resources.azure.com)
- [Azure Service Health](https://status.azure.com)

## 📚 Próximos Passos

- [Melhores Práticas](./04-melhores-praticas.md)
- [Monitoramento e Performance](./02-monitoramento.md)
- [Comandos Úteis](./01-comandos-uteis.md)

---
📅 **Última atualização**: Junho 2025  
⬅️ [Voltar ao Índice](../README.md) | ➡️ [Próximo: Melhores Práticas](./04-melhores-praticas.md)