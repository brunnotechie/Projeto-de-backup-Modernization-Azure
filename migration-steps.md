# Processo de Migração: On-premises para Azure
## Guia de Implementação Passo a Passo

### Fase 1: Planejamento e Avaliação
#### 1.1. Inventário do Ambiente Atual
```powershell
# Script para coletar informações do ambiente on-premises
$serverInfo = @{
    "ServidoresApp" = Get-WmiObject -Class Win32_ComputerSystem
    "Compartilhamentos" = Get-SmbShare
    "EspacoUtilizado" = Get-PSDrive | Where-Object {$_.Free -gt 0}
}
```

**Ações necessárias:**
- Documentar configurações dos servidores
- Mapear dependências das aplicações
- Avaliar volume de dados a serem migrados
- Identificar janelas de manutenção disponíveis

### Fase 2: Preparação do Ambiente Azure
#### 2.1. Configuração Inicial

```bash
# Variáveis de ambiente
SUBSCRIPTION="sua-subscription"
RESOURCE_GROUP="rg-migration-prod"
LOCATION="eastus2"
VNET_NAME="vnet-prod"
SUBNET_NAME="subnet-apps"

# Login no Azure
az login
az account set --subscription $SUBSCRIPTION

# Criar Resource Group
az group create \
    --name $RESOURCE_GROUP \
    --location $LOCATION
```

#### 2.2. Configuração de Rede

```bash
# Criar Virtual Network e Subnet
az network vnet create \
    --resource-group $RESOURCE_GROUP \
    --name $VNET_NAME \
    --address-prefix 10.0.0.0/16 \
    --subnet-name $SUBNET_NAME \
    --subnet-prefix 10.0.1.0/24

# Configurar NSG (Network Security Group)
az network nsg create \
    --resource-group $RESOURCE_GROUP \
    --name "nsg-apps"

# Regra para permitir SMB
az network nsg rule create \
    --resource-group $RESOURCE_GROUP \
    --nsg-name "nsg-apps" \
    --name "Allow-SMB" \
    --priority 100 \
    --direction Inbound \
    --source-address-prefixes "*" \
    --source-port-ranges "*" \
    --destination-address-prefixes "*" \
    --destination-port-ranges 445 \
    --access Allow \
    --protocol Tcp
```

### Fase 3: Implementação do Storage
#### 3.1. Criar Storage Account e File Share

```bash
# Criar Storage Account
az storage account create \
    --name "stamigrationprod" \
    --resource-group $RESOURCE_GROUP \
    --location $LOCATION \
    --sku Standard_ZRS \
    --enable-large-file-share \
    --kind StorageV2

# Criar File Share
az storage share create \
    --name "app-shared" \
    --account-name "stamigrationprod" \
    --quota 1024
```

### Fase 4: Migração dos Dados
#### 4.1. Preparação para Migração

```powershell
# Script para obter tamanho total dos dados
$sourcePath = "\\servidor-on-premises\app-shared"
$folderSize = Get-ChildItem $sourcePath -Recurse | Measure-Object -Property Length -Sum

Write-Host "Tamanho total a ser migrado: $($folderSize.Sum / 1GB) GB"
```

#### 4.2. Processo de Migração

```powershell
# Configurar AzCopy
$env:AZCOPY_CRED_TYPE = "Anonymous"
$destSAS = "seu-token-sas"

# Migrar dados usando AzCopy
azcopy copy $sourcePath "$destSAS" --recursive --preserve-smb-permissions --preserve-smb-info

# Verificar integridade da migração
azcopy jobs show --with-status "Completed"
```

### Fase 5: Configuração das VMs Azure
#### 5.1. Criar VMs

```bash
# Criar VMs de aplicação
az vm create \
    --resource-group $RESOURCE_GROUP \
    --name "vm-app1" \
    --image "Win2019Datacenter" \
    --size "Standard_DS2_v2" \
    --admin-username "azureadmin" \
    --vnet-name $VNET_NAME \
    --subnet $SUBNET_NAME

az vm create \
    --resource-group $RESOURCE_GROUP \
    --name "vm-app2" \
    --image "Win2019Datacenter" \
    --size "Standard_DS2_v2" \
    --admin-username "azureadmin" \
    --vnet-name $VNET_NAME \
    --subnet $SUBNET_NAME
```

#### 5.2. Configurar Azure Files nas VMs

```powershell
# Script para montar Azure Files
$storageAccount = "stamigrationprod"
$shareName = "app-shared"
$storageKey = "sua-storage-key"

# Testar conectividade
Test-NetConnection -ComputerName "$storageAccount.file.core.windows.net" -Port 445

# Persistir credenciais
cmdkey /add:"$storageAccount.file.core.windows.net" /user:"Azure\$storageAccount" /pass:$storageKey

# Montar compartilhamento
New-PSDrive -Name "Z" -PSProvider FileSystem -Root "\\$storageAccount.file.core.windows.net\$shareName" -Persist
```

### Fase 6: Configuração de Backup
#### 6.1. Recovery Services Vault

```bash
# Criar Recovery Services Vault
az backup vault create \
    --resource-group $RESOURCE_GROUP \
    --name "rsv-prod" \
    --location $LOCATION

# Configurar política de backup
az backup protection enable-for-azurefile \
    --resource-group $RESOURCE_GROUP \
    --vault-name "rsv-prod" \
    --storage-account "stamigrationprod" \
    --file-share "app-shared" \
    --policy-name "DailyBackupPolicy"
```

### Fase 7: Validação e Testes
#### 7.1. Checklist de Validação

```powershell
# Script de validação
$validationResults = @{
    "ConectividadeAzureFiles" = Test-Path "\\stamigrationprod.file.core.windows.net\app-shared"
    "PermissoesAcesso" = Get-Acl "Z:\"
    "EspacoDisponivel" = Get-PSDrive Z | Select-Object Free
}

# Testar backup
$backupTest = az backup protection backup-now \
    --resource-group $RESOURCE_GROUP \
    --vault-name "rsv-prod" \
    --container-name "stamigrationprod" \
    --item-name "app-shared" \
    --backup-management-type AzureStorage
```

### Fase 8: Cutover e Go-Live
#### 8.1. Procedimento de Cutover

```powershell
# Script de cutover
# 1. Parar serviços que usam o compartilhamento
Stop-Service -Name "NomeDoServico"

# 2. Desmontar compartilhamento antigo
Net Use X: /delete

# 3. Montar novo compartilhamento
Net Use Z: \\stamigrationprod.file.core.windows.net\app-shared /persistent:yes

# 4. Reiniciar serviços
Start-Service -Name "NomeDoServico"
```

### Fase 9: Monitoramento Pós-Migração
#### 9.1. Configuração de Alertas

```bash
# Criar regras de monitoramento
az monitor metrics alert create \
    --name "storage-performance" \
    --resource-group $RESOURCE_GROUP \
    --condition "avg Latency > 100" \
    --window-size 5m \
    --evaluation-frequency 1m

# Configurar Log Analytics
az monitor log-analytics workspace create \
    --resource-group $RESOURCE_GROUP \
    --workspace-name "law-migration-prod"
```

### Fase 10: Documentação e Handover
#### 10.1. Documentação Final

Criar documentação incluindo:
- Topologia da nova infraestrutura
- Procedimentos operacionais
- Plano de backup e recuperação
- Contatos de suporte
- Procedimentos de troubleshooting

#### 10.2. Plano de Rollback

```powershell
# Script de rollback
function Invoke-MigrationRollback {
    # 1. Desmontar Azure Files
    Net Use Z: /delete
    
    # 2. Remover credenciais Azure
    cmdkey /delete:stamigrationprod.file.core.windows.net
    
    # 3. Remontar compartilhamento original
    Net Use X: \\servidor-on-premises\app-shared /persistent:yes
    
    # 4. Reiniciar serviços
    Start-Service -Name "NomeDoServico"
}
```

### Notas Importantes:

1. **Antes da Migração:**
   - Realizar backup completo do ambiente
   - Documentar todas as configurações atuais
   - Validar requisitos de rede e latência

2. **Durante a Migração:**
   - Monitorar progresso da cópia de dados
   - Validar integridade dos dados
   - Manter logs de todas as ações

3. **Pós-Migração:**
   - Monitorar performance
   - Validar backups
   - Coletar feedback dos usuários

4. **Considerações de Segurança:**
   - Implementar encriptação em trânsito
   - Configurar RBAC apropriado
   - Seguir princípio do menor privilégio

