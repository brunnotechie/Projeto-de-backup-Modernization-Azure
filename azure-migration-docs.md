# Projeto de Modernização: On-premises para Azure
## Documentação Técnica de Implementação

### Índice
1. [Pré-requisitos](#pré-requisitos)
2. [Preparação do Ambiente](#preparação-do-ambiente)
3. [Implementação da Infraestrutura](#implementação-da-infraestrutura)
4. [Migração dos Dados](#migração-dos-dados)
5. [Configuração de Backup](#configuração-de-backup)
6. [Testes e Validação](#testes-e-validação)
7. [Cutover e Go-Live](#cutover-e-go-live)

### 1. Pré-requisitos

#### 1.1. Ferramentas Necessárias
- Azure CLI instalado
- PowerShell Az Module
- AzCopy v10 ou superior
- Subscription Azure ativa
- Permissões de administrador nos servidores on-premises

#### 1.2. Informações Necessárias
```bash
# Variáveis do ambiente que serão utilizadas
SUBSCRIPTION_ID="sua-subscription"
RESOURCE_GROUP="rg-migration-prod"
LOCATION="eastus2"
VNET_NAME="vnet-prod"
SUBNET_NAME="subnet-apps"
STORAGE_ACCOUNT="stamigrationprod"
SHARE_NAME="app-shared"
VAULT_NAME="rsv-prod"
```

### 2. Preparação do Ambiente

#### 2.1. Login no Azure
```bash
# Login no Azure
az login

# Selecionar subscription
az account set --subscription $SUBSCRIPTION_ID
```

#### 2.2. Criar Resource Group
```bash
# Criar Resource Group
az group create \
    --name $RESOURCE_GROUP \
    --location $LOCATION
```

#### 2.3. Criar Virtual Network e Subnet
```bash
# Criar VNet
az network vnet create \
    --resource-group $RESOURCE_GROUP \
    --name $VNET_NAME \
    --address-prefix 10.0.0.0/16 \
    --subnet-name $SUBNET_NAME \
    --subnet-prefix 10.0.1.0/24
```

### 3. Implementação da Infraestrutura

#### 3.1. Criar Storage Account e File Share
```bash
# Criar Storage Account
az storage account create \
    --name $STORAGE_ACCOUNT \
    --resource-group $RESOURCE_GROUP \
    --location $LOCATION \
    --sku Standard_ZRS \
    --enable-large-file-share \
    --kind StorageV2

# Criar File Share
az storage share create \
    --name $SHARE_NAME \
    --account-name $STORAGE_ACCOUNT \
    --quota 1024
```

#### 3.2. Criar Virtual Machines
```bash
# Criar primeira VM
az vm create \
    --resource-group $RESOURCE_GROUP \
    --name "vm-app1" \
    --image "Win2019Datacenter" \
    --admin-username "azureuser" \
    --size "Standard_DS2_v2" \
    --vnet-name $VNET_NAME \
    --subnet $SUBNET_NAME \
    --public-ip-sku Standard

# Criar segunda VM
az vm create \
    --resource-group $RESOURCE_GROUP \
    --name "vm-app2" \
    --image "Win2019Datacenter" \
    --admin-username "azureuser" \
    --size "Standard_DS2_v2" \
    --vnet-name $VNET_NAME \
    --subnet $SUBNET_NAME \
    --public-ip-sku Standard
```

### 4. Migração dos Dados

#### 4.1. Preparar AzCopy
```powershell
# Script PowerShell para migração usando AzCopy
$sourcePath = "\\servidor-on-premises\app-shared"
$destSAS = "SAS-TOKEN-DO-AZURE-FILES"

# Executar migração com AzCopy
azcopy copy $sourcePath $destSAS --recursive
```

#### 4.2. Montar Azure Files nas VMs
```powershell
# PowerShell script para montar Azure Files
$connectTestResult = Test-NetConnection -ComputerName "$STORAGE_ACCOUNT.file.core.windows.net" -Port 445
if ($connectTestResult.TcpTestSucceeded) {
    cmd.exe /C "cmdkey /add:`"$STORAGE_ACCOUNT.file.core.windows.net`" /user:`"Azure\$STORAGE_ACCOUNT`" /pass:`"$STORAGE_KEY`""
    New-PSDrive -Name Z -PSProvider FileSystem -Root "\\$STORAGE_ACCOUNT.file.core.windows.net\$SHARE_NAME" -Persist
}
```

### 5. Configuração de Backup

#### 5.1. Criar Recovery Services Vault
```bash
# Criar Recovery Services Vault
az backup vault create \
    --resource-group $RESOURCE_GROUP \
    --name $VAULT_NAME \
    --location $LOCATION

# Configurar política de backup
az backup protection enable-for-azurefile \
    --resource-group $RESOURCE_GROUP \
    --vault-name $VAULT_NAME \
    --storage-account $STORAGE_ACCOUNT \
    --file-share $SHARE_NAME \
    --policy-name "DailyBackupPolicy"
```

### 6. Testes e Validação

#### 6.1. Checklist de Validação
```powershell
# Script de validação de acesso ao File Share
$testPath = "\\$STORAGE_ACCOUNT.file.core.windows.net\$SHARE_NAME"
if (Test-Path $testPath) {
    Write-Host "Acesso ao Azure Files OK"
} else {
    Write-Host "Falha no acesso ao Azure Files"
}

# Teste de backup
az backup protection backup-now \
    --resource-group $RESOURCE_GROUP \
    --vault-name $VAULT_NAME \
    --container-name $STORAGE_ACCOUNT \
    --item-name $SHARE_NAME \
    --backup-management-type AzureStorage
```

### 7. Cutover e Go-Live

#### 7.1. Checklist Final
- [ ] Validar sincronização de dados
- [ ] Confirmar backups
- [ ] Verificar permissões
- [ ] Testar conectividade
- [ ] Validar aplicações

#### 7.2. Script de Cutover
```powershell
# Desmontar compartilhamento antigo
Net Use X: /delete

# Remover credenciais antigas
cmdkey /delete:servidor-on-premises

# Montar novo compartilhamento Azure Files
Net Use Z: \\$STORAGE_ACCOUNT.file.core.windows.net\$SHARE_NAME /persistent:yes
```

### Monitoramento e Manutenção

#### Configuração de Alertas
```bash
# Criar regra de alerta para disponibilidade
az monitor metrics alert create \
    --name "storage-availability" \
    --resource-group $RESOURCE_GROUP \
    --scopes "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT" \
    --condition "avg Availability < 99.9" \
    --window-size 5m \
    --evaluation-frequency 1m
```

### Documentação de Rollback

Em caso de necessidade de rollback, siga estes passos:

1. Desmontar Azure Files:
```powershell
Net Use Z: /delete
```

2. Remontar compartilhamento on-premises:
```powershell
Net Use X: \\servidor-on-premises\app-shared /persistent:yes
```

3. Desabilitar backup no Azure:
```bash
az backup protection disable \
    --resource-group $RESOURCE_GROUP \
    --vault-name $VAULT_NAME \
    --container-name $STORAGE_ACCOUNT \
    --item-name $SHARE_NAME \
    --backup-management-type AzureStorage \
    --yes
```

### Notas Importantes

1. **Segurança**: 
   - Todos os scripts devem ser executados com privilégios administrativos
   - Armazenar chaves e senhas em Azure Key Vault
   - Implementar RBAC apropriado

2. **Performance**:
   - Monitorar latência durante a migração
   - Usar AzCopy com threads paralelos para otimização

3. **Compliance**:
   - Documentar todas as alterações
   - Manter logs de auditoria
   - Validar requisitos de conformidade

