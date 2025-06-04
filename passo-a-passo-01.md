# 01. Criando Azure SQL Managed Instance

Este guia apresenta o processo completo para criar uma Azure SQL Managed Instance através do Portal Azure.

## 🎯 Pré-requisitos

- Conta ativa do Microsoft Azure
- Assinatura com permissões para criar recursos
- Conhecimento básico de redes virtuais Azure
- Papel de **SQL Managed Instance Contributor** no escopo da assinatura

## ⏰ Tempo Estimado

**Primeira instância em uma nova subnet**: 4-6 horas  
**Instâncias adicionais em subnet existente**: 90-120 minutos

> ⚠️ **Importante**: A criação de uma Managed Instance é uma operação longa. Planeje adequadamente!

## 📋 Passo a Passo

### Etapa 1: Acessando o Portal Azure

1. Acesse [portal.azure.com](https://portal.azure.com)
2. Faça login com suas credenciais
3. No menu lateral esquerdo, clique em **"Criar um recurso"**
4. Na barra de pesquisa, digite **"Azure SQL"**
5. Selecione **"Azure SQL"** nos resultados

### Etapa 2: Selecionando Managed Instance

1. Na página Azure SQL, clique em **"+ Criar"**
2. Na seção **"Instâncias gerenciadas de SQL"**, clique em **"Criar"**
3. Selecione **"Instância única"** no dropdown
4. Clique em **"Criar"** novamente

### Etapa 3: Configurações Básicas

#### Detalhes do Projeto
- **Assinatura**: Selecione sua assinatura Azure
- **Grupo de recursos**: 
  - Selecione existente **OU**
  - Clique em **"Criar novo"** e forneça um nome (ex: `rg-sqlmi-producao`)

#### Detalhes da Instância Gerenciada
- **Nome da instância gerenciada**: Nome único (ex: `mi-empresa-prod01`)
  - ✅ Deve ser único globalmente
  - ✅ Apenas letras minúsculas, números e hífens
  - ✅ Entre 3-63 caracteres
- **Região**: Escolha a região mais próxima dos usuários
- **Pool de instâncias**: Deixe **"Não"** (para primeira instância)

#### Computação + Armazenamento
- **Tipo de preço**: Clique em **"Configurar instância gerenciada"**
  - **Camada de serviço**: 
    - `General Purpose` (Para maioria das cargas de trabalho)
    - `Business Critical` (Para aplicações críticas)
  - **Série de hardware**: `Standard-series (Gen 5)` (recomendado)
  - **vCores**: Inicie com 4 ou 8 vCores
  - **Armazenamento**: 32 GB - 16 TB (conforme necessidade)

### Etapa 4: Configuração de Autenticação

#### Método de Autenticação
- **Autenticação SQL**: ✅ Recomendado para início
  - **Login de administrador**: Nome do usuário administrador
  - **Senha**: Senha forte (mínimo 16 caracteres)
  - **Confirmar senha**: Repita a senha

> 💡 **Dica**: Anote as credenciais em local seguro!

### Etapa 5: Configuração de Rede

#### Conectividade
- **Rede virtual**: 
  - **Criar nova** (primeira vez) **OU**
  - **Usar existente** (se já tiver VNet configurada)

##### Para Nova VNet:
- **Nome da rede virtual**: `vnet-sqlmi-prod`
- **Intervalo de endereços**: `10.0.0.0/16`
- **Nome da sub-rede**: `subnet-sqlmi`
- **Intervalo da sub-rede**: `10.0.1.0/24`

#### Configurações de Conexão
- **Tipo de conexão**: `Proxy (Padrão)` 
- **Ponto de extremidade público**: 
  - **Desabilitar** (mais seguro)
  - **Habilitar** (se necessário acesso público)

### Etapa 6: Configurações Adicionais (Opcional)

#### Segurança
- **Transparent Data Encryption (TDE)**: ✅ Habilitado por padrão
- **Auditoria**: Configure conforme políticas da empresa

#### Backup
- **Retenção de backup**: 7-35 dias (padrão: 7 dias)
- **Backup geograficamente redundante**: Habilite para DR

### Etapa 7: Revisão e Criação

1. Clique em **"Revisar + criar"**
2. Aguarde a validação automática
3. Revise todas as configurações
4. Verifique estimativa de custos
5. Clique em **"Criar"**

## 📊 Monitorando a Criação

### Durante o Processo
1. Acesse **"Notificações"** (ícone de sino)
2. Clique na notificação de implantação
3. Acompanhe o progresso:
   - ✅ Validação da solicitação
   - 🔄 Criação/redimensionamento do cluster virtual
   - 🔄 Implantação da instância

### Verificando Status
```bash
# Via Azure CLI (opcional)
az sql mi show --name mi-empresa-prod01 --resource-group rg-sqlmi-producao
```

## ✅ Validação da Criação

### Verificações Obrigatórias
1. **Status**: Instance deve estar **"Online"**
2. **Conectividade**: Testar conexão
3. **Rede**: VNet e subnet criadas corretamente
4. **DNS**: Resolução do nome da instância

### Próximos Passos
- ➡️ [Configuração de Rede Virtual](./02-configuracao-rede.md)
- ➡️ [Conectando com SSMS](./03-conectando-ssms.md)

## 🚨 Troubleshooting Comum

### Problema: Criação Muito Lenta
**Solução**: Normal para primeira instância. Aguarde até 6 horas.

### Problema: Erro de Validação de Subnet
**Solução**: Verificar se subnet tem pelo menos 32 endereços IP (/27 ou maior).

### Problema: Falha na Criação
**Solução**: 
1. Verificar cotas da assinatura
2. Revisar permissões de usuário
3. Tentar em região diferente

## 📋 Checklist de Criação

- [ ] Assinatura selecionada
- [ ] Grupo de recursos criado/selecionado
- [ ] Nome da instância definido
- [ ] Região escolhida
- [ ] Camada de serviço configurada
- [ ] Autenticação configurada
- [ ] Rede virtual criada/selecionada
- [ ] Configurações de segurança definidas
- [ ] Implantação iniciada
- [ ] Status "Online" confirmado

## 🔗 Referências

- [Documentação Oficial - Criar Instância](https://learn.microsoft.com/pt-br/azure/azure-sql/managed-instance/instance-create-quickstart?view=azuresql&tabs=azure-portal)
- [Requisitos de Rede](https://learn.microsoft.com/pt-br/azure/azure-sql/managed-instance/connectivity-architecture-overview?view=azuresql)
- [Preços e Cotas](https://azure.microsoft.com/pt-br/pricing/details/azure-sql-managed-instance/)

---
📅 **Última atualização**: Junho 2025  
⬅️ [Voltar ao Índice](../README.md) | ➡️ [Próximo: Configuração de Rede](./02-configuracao-rede.md)