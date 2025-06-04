# Comandos Úteis - PowerShell e Azure CLI

Este guia apresenta comandos essenciais para gerenciar Azure SQL Managed Instance através de PowerShell e Azure CLI.

## 🔧 Preparação do Ambiente

### 📦 Instalação do Azure PowerShell
```powershell
# Instalar módulo Azure PowerShell
Install-Module -Name Az -Repository PSGallery -Force

# Verificar versão instalada
Get-Module -Name Az -ListAvailable

# Login no Azure
Connect-AzAccount

# Selecionar assinatura específica
Set-AzContext -SubscriptionId "your-subscription-id"
```

### 🔷 Instalação do Azure CLI
```bash
# Windows (via Chocolatey)
choco install azure-cli

# Ubuntu/Debian
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Login no Azure
az login

# Selecionar assinatura
az account set --subscription "your-subscription-id"
```

## 🏗️ Criação e Gerenciamento de Instâncias

### ✨ Criar Nova Instância - PowerShell

```powershell
# Definir variáveis
$resourceGroupName = "rg-sqlmi-prod"
$location = "East US"
$vnetName = "vnet-sqlmi"
$subnetName = "subnet-sqlmi"
$instanceName = "mi-empresa-prod01"
$adminLogin = "sqladmin"
$adminPassword = "Sua$enhaForte123!"

# Criar Resource Group
New-AzResourceGroup -Name $resourceGroupName -Location $location

# Criar Virtual Network
$vnet = New-AzVirtualNetwork `
    -ResourceGroupName $resourceGroupName `
    -Location $location `
    -Name $vnetName `
    -AddressPrefix "10.0.0.0/16"

# Criar Subnet dedicada para Managed Instance
$subnetConfig = Add-AzVirtualNetworkSubnetConfig `
    -Name $subnetName `
    -AddressPrefix "10.0.1.0/24" `
    -VirtualNetwork $vnet `
    -Delegation "Microsoft.Sql/managedInstances"

$vnet | Set-AzVirtualNetwork

# Criar Managed Instance
$password = ConvertTo-SecureString $adminPassword -AsPlainText -Force
$creds = New-Object System.Management.Automation.PSCredential ($adminLogin, $password)

New-AzSqlInstance `
    -ResourceGroupName $resourceGroupName `
    -Name $instanceName `
    -Location $location `
    -SubnetId "/subscriptions/your-sub-id/resourceGroups/$resourceGroupName/providers/Microsoft.Network/virtualNetworks/$vnetName/subnets/$subnetName" `
    -AdministratorCredential $creds `
    -StorageSizeInGB 256 `
    -VCore 4 `
    -Edition "GeneralPurpose" `
    -ComputeGeneration "Gen5"
```

### 🔷 Criar Nova Instância - Azure CLI

```bash
# Definir variáveis
RESOURCE_GROUP="rg-sqlmi-prod"
LOCATION="eastus"
VNET_NAME="vnet-sqlmi"
SUBNET_NAME="subnet-sqlmi"
INSTANCE_NAME="mi-empresa-prod01"
ADMIN_LOGIN="sqladmin"
ADMIN_PASSWORD="Sua$enhaForte123!"

# Criar Resource Group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Criar Virtual Network
az network vnet create \
    --resource-group $RESOURCE_GROUP \
    --name $VNET_NAME \
    --address-prefix 10.0.0.0/16 \
    --location $LOCATION

# Criar Subnet para Managed Instance
az network vnet subnet create \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $VNET_NAME \
    --name $SUBNET_NAME \
    --address-prefix 10.0.1.0/24 \
    --delegations Microsoft.Sql/managedInstances

# Criar Managed Instance
az sql mi create \
    --resource-group $RESOURCE_GROUP \
    --name $INSTANCE_NAME \
    --location $LOCATION \
    --subnet $SUBNET_NAME \
    --vnet-name $VNET_NAME \
    --admin-user $ADMIN_LOGIN \
    --admin-password $ADMIN_PASSWORD \
    --capacity 4 \
    --storage 256GB \
    --edition GeneralPurpose \
    --family Gen5
```

## 📊 Comandos de Consulta e Monitoramento

### 🔍 Listar Instâncias - PowerShell

```powershell
# Listar todas as instâncias na assinatura
Get-AzSqlInstance

# Listar instâncias em um Resource Group específico
Get-AzSqlInstance -ResourceGroupName "rg-sqlmi-prod"

# Obter detalhes de uma instância específica
Get-AzSqlInstance -ResourceGroupName "rg-sqlmi-prod" -Name "mi-empresa-prod01"

# Verificar status da instância
$instance = Get-AzSqlInstance -ResourceGroupName "rg-sqlmi-prod" -Name "mi-empresa-prod01"
Write-Output "Estado: $($instance.State)"
Write-Output "FQDN: $($instance.FullyQualifiedDomainName)"
```

### 🔷 Listar Instâncias - Azure CLI

```bash
# Listar todas as instâncias
az sql mi list

# Listar instâncias em um Resource Group
az sql mi list --resource-group rg-sqlmi-prod

# Obter detalhes de uma instância específica
az sql mi show --name mi-empresa-prod01 --resource-group rg-sqlmi-prod

# Verificar status (formato JSON compacto)
az sql mi show --name mi-empresa-prod01 --resource-group rg-sqlmi-prod --query "state"
```

## ⚙️ Configuração e Modificação

### 🔧 Alterar Configurações - PowerShell

```powershell
# Alterar número de vCores
Set-AzSqlInstance `
    -ResourceGroupName "rg-sqlmi-prod" `
    -Name "mi-empresa-prod01" `
    -VCore 8

# Alterar armazenamento
Set-AzSqlInstance `
    -ResourceGroupName "rg-sqlmi-prod" `
    -Name "mi-empresa-prod01" `
    -StorageSizeInGB 512

# Alterar senha do administrador
$newPassword = ConvertTo-SecureString "NovaSenha456!" -AsPlainText -Force
Set-AzSqlInstance `
    -ResourceGroupName "rg-sqlmi-prod" `
    -Name "mi-empresa-prod01" `
    -AdministratorPassword $newPassword

# Habilitar/desabilitar public endpoint
Set-AzSqlInstance `
    -ResourceGroupName "rg-sqlmi-prod" `
    -Name "mi-empresa-prod01" `
    -PublicDataEndpointEnabled $true
```

### 🔷 Alterar Configurações - Azure CLI

```bash
# Alterar número de vCores
az sql mi update \
    --name mi-empresa-prod01 \
    --resource-group rg-sqlmi-prod \
    --capacity 8

# Alterar armazenamento
az sql mi update \
    --name mi-empresa-prod01 \
    --resource-group rg-sqlmi-prod \
    --storage 512GB

# Habilitar public endpoint
az sql mi update \
    --name mi-empresa-prod01 \
    --resource-group rg-sqlmi-prod \
    --public-data-endpoint-enabled true
```

## 🛡️ Configurações de Segurança

### 🔐 Configurar Azure AD Admin - PowerShell

```powershell
# Definir Azure AD admin
Set-AzSqlInstanceActiveDirectoryAdministrator `
    -ResourceGroupName "rg-sqlmi-prod" `
    -InstanceName "mi-empresa-prod01" `
    -DisplayName "DBA-Team" `
    -ObjectId "12345678-1234-1234-1234-123456789012"

# Verificar Azure AD admin
Get-AzSqlInstanceActiveDirectoryAdministrator `
    -ResourceGroupName "rg-sqlmi-prod" `
    -InstanceName "mi-empresa-prod01"
```

### 🔷 Configurar Azure AD Admin - Azure CLI

```bash
# Definir Azure AD admin
az sql mi ad-admin create \
    --resource-group rg-sqlmi-prod \
    --managed-instance mi-empresa-prod01 \
    --display-name "DBA-Team" \
    --object-id "12345678-1234-1234-1234-123456789012"

# Verificar Azure AD admin
az sql mi ad-admin list \
    --resource-group rg-sqlmi-prod \
    --managed-instance mi-empresa-prod01
```

## 🗄️ Gerenciamento de Databases

### 📁 Operações com Databases - PowerShell

```powershell
# Listar databases na instância
Get-AzSqlInstanceDatabase -ResourceGroupName "rg-sqlmi-prod" -InstanceName "mi-empresa-prod01"

# Criar novo database
New-AzSqlInstanceDatabase `
    -ResourceGroupName "rg-sqlmi-prod" `
    -InstanceName "mi-empresa-prod01" `
    -Name "AppDatabase"

# Fazer backup copy-only de database
Backup-AzSqlInstanceDatabase `
    -ResourceGroupName "rg-sqlmi-prod" `
    -InstanceName "mi-empresa-prod01" `
    -Name "AppDatabase" `
    -StorageUri "https://mystorage.blob.core.windows.net/backups/"
```

### 🔷 Operações com Databases - Azure CLI

```bash
# Listar databases na instância
az sql midb list \
    --resource-group rg-sqlmi-prod \
    --managed-instance mi-empresa-prod01

# Criar novo database
az sql midb create \
    --resource-group rg-sqlmi-prod \
    --managed-instance mi-empresa-prod01 \
    --name AppDatabase

# Restaurar database de backup
az sql midb restore \
    --resource-group rg-sqlmi-prod \
    --managed-instance mi-empresa-prod01 \
    --name AppDatabase-Restored \
    --dest-name AppDatabase \
    --time "2025-06-03T10:00:00"
```

## 📈 Monitoramento e Métricas

### 📊 Coletar Métricas - PowerShell

```powershell
# Obter métricas de CPU
Get-AzMetric `
    -ResourceId "/subscriptions/sub-id/resourceGroups/rg-sqlmi-prod/providers/Microsoft.Sql/managedInstances/mi-empresa-prod01" `
    -MetricName "avg_cpu_percent" `
    -StartTime (Get-Date).AddHours(-1) `
    -EndTime (Get-Date) `
    -TimeGrain "00:05:00"

# Obter métricas de armazenamento
Get-AzMetric `
    -ResourceId "/subscriptions/sub-id/resourceGroups/rg-sqlmi-prod/providers/Microsoft.Sql/managedInstances/mi-empresa-prod01" `
    -MetricName "storage_space_used_mb" `
    -StartTime (Get-Date).AddDays(-1) `
    -EndTime (Get-Date) `
    -TimeGrain "01:00:00"
```

### 🔷 Coletar Métricas - Azure CLI

```bash
# Obter métricas de CPU (última hora)
az monitor metrics list \
    --resource "/subscriptions/sub-id/resourceGroups/rg-sqlmi-prod/providers/Microsoft.Sql/managedInstances/mi-empresa-prod01" \
    --metric "avg_cpu_percent" \
    --start-time "2025-06-03T09:00:00Z" \
    --end-time "2025-06-03T10:00:00Z" \
    --interval PT5M

# Listar métricas disponíveis
az monitor metrics list-definitions \
    --resource "/subscriptions/sub-id/resourceGroups/rg-sqlmi-prod/providers/Microsoft.Sql/managedInstances/mi-empresa-prod01"
```

## 🚨 Operações de Emergência

### ⚡ Comandos Críticos

```powershell
# EMERGÊNCIA: Parar todas as conexões ativas
# ⚠️ Use com extremo cuidado!
Invoke-AzSqlInstanceScript `
    -ResourceGroupName "rg-sqlmi-prod" `
    -InstanceName "mi-empresa-prod01" `
    -ScriptContent "KILL [session_id]"  # Substitua por session_id específico

# Verificar conexões ativas
Invoke-AzSqlInstanceScript `
    -ResourceGroupName "rg-sqlmi-prod" `
    -InstanceName "mi-empresa-prod01" `
    -ScriptContent "SELECT * FROM sys.dm_exec_sessions WHERE is_user_process = 1"
```

## 🧹 Limpeza e Exclusão

### 🗑️ Remover Recursos - PowerShell

```powershell
# Remover database específico
Remove-AzSqlInstanceDatabase `
    -ResourceGroupName "rg-sqlmi-prod" `
    -InstanceName "mi-empresa-prod01" `
    -Name "AppDatabase" `
    -Force

# Remover instância completa (⚠️ AÇÃO IRREVERSÍVEL!)
Remove-AzSqlInstance `
    -ResourceGroupName "rg-sqlmi-prod" `
    -Name "mi-empresa-prod01" `
    -Force
```

### 🔷 Remover Recursos - Azure CLI

```bash
# Remover database
az sql midb delete \
    --resource-group rg-sqlmi-prod \
    --managed-instance mi-empresa-prod01 \
    --name AppDatabase \
    --yes

# Remover instância (⚠️ AÇÃO IRREVERSÍVEL!)
az sql mi delete \
    --resource-group rg-sqlmi-prod \
    --name mi-empresa-prod01 \
    --yes
```

## 📋 Scripts de Automação

### 🔄 Script de Monitoramento Automático

```powershell
# monitor-sqlmi.ps1
param(
    [string]$ResourceGroupName,
    [string]$InstanceName,
    [int]$CpuThreshold = 80
)

$instance = Get-AzSqlInstance -ResourceGroupName $ResourceGroupName -Name $InstanceName
$metrics = Get-AzMetric -ResourceId $instance.Id -MetricName "avg_cpu_percent" -StartTime (Get-Date).AddMinutes(-10)

$latestCpu = $metrics.Data | Sort-Object TimeStamp -Descending | Select-Object -First 1

if ($latestCpu.Average -gt $CpuThreshold) {
    Write-Warning "CPU usage is high: $($latestCpu.Average)%"
    # Implementar ações de alerta aqui
} else {
    Write-Output "CPU usage is normal: $($latestCpu.Average)%"
}
```

## 🔗 Referências e Próximos Passos

### 📚 Documentação Oficial
- [PowerShell para SQL MI](https://docs.microsoft.com/powershell/module/az.sql/)
- [Azure CLI para SQL MI](https://docs.microsoft.com/cli/azure/sql/mi)
- [Exemplos de Scripts](https://github.com/Azure/azure-powershell/tree/main/src/Sql)

### 🚀 Próximos Tópicos
- [Monitoramento e Performance](./02-monitoramento.md)
- [Troubleshooting Comum](./03-troubleshooting.md)
- [Melhores Práticas](./04-melhores-praticas.md)

---
📅 **Última atualização**: Junho 2025  
⬅️ [Voltar ao Índice](../README.md) | ➡️ [Próximo: Monitoramento](./02-monitoramento.md)