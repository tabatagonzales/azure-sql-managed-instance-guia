# Conectando ao Azure SQL Managed Instance com SSMS

Este guia apresenta o passo a passo para conectar ao Azure SQL Managed Instance utilizando SQL Server Management Studio (SSMS).

## 🎯 Pré-requisitos

- SQL Server Management Studio (SSMS) - [Download da versão mais recente](https://docs.microsoft.com/sql/ssms/download-sql-server-management-studio-ssms)
- Versão recomendada: 18.6 ou superior (suporte a Microsoft Entra ID com MFA)
- Azure SQL Managed Instance previamente criado e online
- Credenciais de acesso (SQL ou Microsoft Entra ID)
- Conectividade de rede configurada (VPN, Peering ou Endpoint público)

## 🔄 Tipos de Conexão

### 1. Conexão Via VNet Local (Padrão)
```
Host: <nome-instancia>.<id-numérico>.database.windows.net
Porta: 1433
Acesso: Somente dentro da VNet ou redes conectadas
```

### 2. Conexão Via Endpoint Público (Opcional)
```
Host: <nome-instancia>.public.<id-numérico>.database.windows.net
Porta: 3342
Acesso: Permitido pela internet pública (se configurado)
```

## 📋 Métodos de Autenticação Suportados

| Método | Descrição | Pré-requisitos |
|--------|-----------|----------------|
| **SQL Authentication** | Autenticação tradicional com login/senha | Credenciais SQL configuradas |
| **Microsoft Entra ID - Senha** | Autenticação com usuário AD/email e senha | Integração Azure AD configurada |
| **Microsoft Entra ID - Integrada** | Autenticação com credenciais do Windows | Computador integrado ao AD |
| **Microsoft Entra ID - MFA** | Autenticação com múltiplos fatores | SSMS 18.6+ e Integração AD |

## 🔐 Configuração de Acesso de Rede

### Pré-requisito: Confirme a Conectividade de Rede

#### 1. Para Conexão Via VNet
- Estar conectado à VNet do Azure onde a instância está (via VPN, ExpressRoute, etc.)
- Ou conectado a uma rede com peering configurado
- NSGs/Firewalls permitindo tráfego TCP/1433

#### 2. Para Conexão Via Endpoint Público
- Endpoint público habilitado na instância
- IP do cliente adicionado às regras de firewall
- Porta 3342 desbloqueada em firewalls locais

## 📝 Passo a Passo para Conexão

### Etapa 1: Preparação
1. Abra o SQL Server Management Studio (SSMS)
2. Clique em **Arquivo** → **Conectar Objeto de Servidor**
3. Ou pressione **Ctrl+Shift+C**

### Etapa 2: Configurar Conexão

#### Dados do Servidor
1. **Tipo de Servidor**: `Motor de Banco de Dados`
2. **Nome do Servidor**: 
   - **VNet Local**: `mi-empresa.1a2b3c4d5e6f.database.windows.net`
   - **Endpoint Público**: `mi-empresa.public.1a2b3c4d5e6f.database.windows.net,3342`
   
   > 💡 **Dica**: Note que para endpoint público, a porta (3342) deve ser especificada após uma vírgula.

3. **Autenticação**:
   - **SQL Server Authentication**: Para login SQL tradicional
   - **Microsoft Entra ID - Universal com MFA**: Para AAD com Multi-fator
   - **Microsoft Entra ID - Integrada**: Para Windows Authentication
   - **Microsoft Entra ID - Senha**: Para AAD com apenas senha

4. **Usuário**: Seu login de SQL ou email do Microsoft Entra ID
5. **Senha**: Sua senha (não necessária para Integrada/MFA)

### Etapa 3: Opções Adicionais (Opcional)

1. Clique em **Opções >>** para expandir configurações adicionais
2. Na aba **Conexão**:
   - **Tempo limite de conexão**: `30` segundos (recomendado)
   - **Banco de dados**: Deixe em branco para conexão inicial

3. Na aba **Avançado**:
   - **Criptografia de rede**: `Obrigatório`
   - **Confiar no certificado do servidor**: `Não` (recomendado)
   
4. Na aba **Propriedades adicionais**:
   - **Application Name**: Opcional, para monitoramento
   - **Workstation ID**: Útil para rastreamento de conexões

### Etapa 4: Concluir Conexão

1. Clique em **Conectar**
2. Se usando MFA, siga as instruções para autenticação
3. Após conectado, você verá a instância no Object Explorer

## 🖼️ Exemplos Visuais

### Exemplo 1: Conexão SQL Authentication
```
Tipo de Servidor: Motor de Banco de Dados
Nome do Servidor: mi-empresa.1a2b3c4d5e6f.database.windows.net
Autenticação: SQL Server Authentication
Login: sqladmin
Senha: ********
```

### Exemplo 2: Conexão Microsoft Entra ID com MFA
```
Tipo de Servidor: Motor de Banco de Dados
Nome do Servidor: mi-empresa.1a2b3c4d5e6f.database.windows.net
Autenticação: Microsoft Entra ID - Universal com MFA
Login: usuario@dominio.com
```

### Exemplo 3: Conexão via Endpoint Público
```
Tipo de Servidor: Motor de Banco de Dados
Nome do Servidor: mi-empresa.public.1a2b3c4d5e6f.database.windows.net,3342
Autenticação: SQL Server Authentication
Login: sqladmin
Senha: ********
```

## 🔍 Verificação da Conexão

### Confirmando Conexão Bem-sucedida
Após conectar, execute a seguinte consulta para verificar detalhes:

```sql
-- Informações da instância conectada
SELECT @@SERVERNAME AS [Servidor],
       @@VERSION AS [Versão],
       DB_NAME() AS [Banco Atual],
       SUSER_NAME() AS [Login],
       CURRENT_USER AS [Usuário],
       CONNECTIONPROPERTY('client_net_address') AS [Endereço IP Cliente];

-- Verificar bancos disponíveis
SELECT name, database_id, create_date
FROM sys.databases
WHERE database_id > 4
ORDER BY name;
```

## ⚠️ Problemas Comuns e Soluções

### Problema 1: Timeout de Conexão

**Erro:** `Timeout expirado. O tempo limite decorrido antes da conclusão da operação ou o servidor não está respondendo.`

**Soluções:**
1. Verificar conectividade de rede (VPN/Peering)
2. Confirmar regras de NSG permitindo tráfego
3. Aumentar timeout de conexão nas opções
4. Verificar status da instância no Portal Azure

### Problema 2: Falha de Autenticação

**Erro:** `Login falhou para o usuário 'usuario'.`

**Soluções:**
1. Verificar credenciais (login/senha)
2. Confirmar se o login existe na instância
3. Verificar se Azure AD admin está configurado (para logins AAD)
4. Tentar conexão com admin SQL original da instância

### Problema 3: Problemas de DNS

**Erro:** `Não foi possível encontrar um servidor ou uma instância do SQL Server.`

**Soluções:**
1. Verificar nome do servidor digitado corretamente
2. Executar `nslookup nome-instancia.1a2b3c4d5e6f.database.windows.net`
3. Verificar configurações de DNS na rede
4. Tentar conectar via IP (caso problema seja de DNS)

### Problema 4: Problemas de Porta/Firewall

**Erro:** `Ocorreu um erro relacionado à rede ou específico da instância ao estabelecer conexão com o SQL Server.`

**Soluções:**
1. Verificar se a porta correta está sendo usada (1433 ou 3342)
2. Confirmar regras de firewall permitindo conexão
3. Testar conectividade com `telnet nome-instancia.1a2b3c4d5e6f.database.windows.net 1433`
4. Verificar se endpoint público está habilitado (para conexões externas)

## 🛠️ Ferramentas Alternativas de Conexão

### 1. Azure Data Studio
- Interface moderna multiplataforma (Windows, Linux, macOS)
- Suporte a Microsoft Entra ID e extensões
- [Download do Azure Data Studio](https://docs.microsoft.com/sql/azure-data-studio/download-azure-data-studio)

### 2. Comandos de Conexão via SQL CLI
```bash
# Usando sqlcmd
sqlcmd -S mi-empresa.1a2b3c4d5e6f.database.windows.net -U sqladmin -P "senha" -d master

# Usando az cli
az sql mi connect -n mi-empresa -g grupo-recursos -u sqladmin
```

### 3. Conexão via Aplicação (.NET)
```csharp
// String de conexão para SQL Authentication
string connectionString = "Server=mi-empresa.1a2b3c4d5e6f.database.windows.net;Database=master;User Id=sqladmin;Password=senha;Encrypt=true;TrustServerCertificate=false;";

// String de conexão para AAD Authentication
string connectionStringAAD = "Server=mi-empresa.1a2b3c4d5e6f.database.windows.net;Database=master;Authentication=Active Directory Password;User Id=usuario@dominio.com;Password=senha;Encrypt=true;TrustServerCertificate=false;";
```

## 📋 Próximos Passos

- ➡️ [Configurações Pós-Implementação](./04-pos-implementacao.md)
- ➡️ [Monitoramento e Performance](../dicas/02-monitoramento.md)
- ➡️ [Troubleshooting Comum](../dicas/03-troubleshooting.md)

## 🔗 Referências

- [Documentação Oficial: Conectar ao SQL MI](https://learn.microsoft.com/pt-br/azure/azure-sql/managed-instance/connect-application-instance)
- [Conectividade e Tipos de Endpoints](https://learn.microsoft.com/pt-br/azure/azure-sql/managed-instance/connection-types-overview)
- [Autenticação Microsoft Entra ID](https://learn.microsoft.com/pt-br/azure/azure-sql/database/authentication-aad-overview)

---
📅 **Última atualização**: Junho 2025  
⬅️ [Voltar ao Índice](../README.md) | ⬅️ [Anterior: Configuração de Rede](./02-configuracao-rede.md) | ➡️ [Próximo: Configurações Pós-Implementação](./04-pos-implementacao.md)