---
title: Data encryption with customer-managed key - Azure Database for PostgreSQL - Flexible server
description: Azure Database for PostgreSQL Flexible server data encryption with a customer-managed key enables you to Bring Your Own Key (BYOK) for data protection at rest. It also allows organizations to implement separation of duties in the management of keys and data.
author: gennadNY
ms.author: gennadyk
ms.reviewer: maghan
ms.date: 1/24/2023
ms.service: postgresql
ms.subservice: flexible-server
ms.custom: devx-track-azurecli
ms.topic: conceptual
---

# Azure Database for PostgreSQL - Flexible Server Data Encryption with a Customer-managed Key 

[!INCLUDE [applies-to-postgresql-flexible-server](../includes/applies-to-postgresql-flexible-server.md)]



Azure PostgreSQL uses [Azure Storage encryption](../../storage/common/storage-service-encryption.md) to encrypt data at-rest by default using Microsoft-managed keys. For Azure PostgreSQL users, it's similar to Transparent Data Encryption (TDE) in other databases such as SQL Server. Many organizations require full control of access to the data using a customer-managed key. Data encryption with customer-managed keys for Azure Database for PostgreSQL Flexible server enables you to bring your key (BYOK) for data protection at rest. It also allows organizations to implement separation of duties in the management of keys and data. With customer-managed encryption, you're responsible for, and in full control of, a key's lifecycle, key usage permissions, and auditing of operations on keys.

Data encryption with customer-managed keys for Azure Database for PostgreSQL Flexible server  is set at the server level. For a given server, a customer-managed key, called the key encryption key (KEK), is used to encrypt the service's data encryption key (DEK). The KEK is an asymmetric key stored in a customer-owned and customer-managed [Azure Key Vault](https://azure.microsoft.com/services/key-vault/)) instance. The Key Encryption Key (KEK) and Data Encryption Key (DEK) are described in more detail later in this article.

Key Vault is a cloud-based, external key management system. It's highly available and provides scalable, secure storage for RSA cryptographic keys, optionally backed by FIPS 140-2 Level 2 validated hardware security modules (HSMs). It doesn't allow direct access to a stored key but provides encryption and decryption services to authorized entities. Key Vault can generate the key, import it, or have it transferred from an on-premises HSM device.

## Benefits

Data encryption with customer-managed keys for Azure Database for PostgreSQL - Flexible Server  provides the following benefits:

- You fully control data-access by the ability to remove the key and make the database inaccessible.

- Full control over the key-lifecycle, including rotation of the key to aligning with corporate policies.

- Central management and organization of keys in Azure Key Vault.

- Enabling encryption doesn't have any additional performance impact with or without customers managed key (CMK) as PostgreSQL relies on the Azure storage layer for data encryption in both scenarios. The only difference is when CMK is used **Azure Storage Encryption Key**, which performs actual data encryption, is encrypted using CMK.

- Ability to implement separation of duties between security officers, DBA, and system administrators.

## Terminology and description

**Data encryption key (DEK)**: A symmetric AES256 key used to encrypt a partition or block of data. Encrypting each block of data with a different key makes crypto analysis attacks more difficult. Access to DEKs is needed by the resource provider or application instance that encrypts and decrypting a specific block. When you replace a DEK with a new key, only the data in its associated block must be re-encrypted with the new key.

**Key encryption key (KEK)**: An encryption key used to encrypt the DEKs. A KEK that never leaves Key Vault allows the DEKs themselves to be encrypted and controlled. The entity that has access to the KEK might be different than the entity that requires the DEK. Since the KEK is required to decrypt the DEKs, the KEK is effectively a single point by which DEKs can be effectively deleted by deleting the KEK.

The DEKs, encrypted with the KEKs, are stored separately. Only an entity with access to the KEK can decrypt these DEKs. For more information, see [Security in encryption at rest](../../security/fundamentals/encryption-atrest.md).

## How data encryption with a customer-managed key work

:::image type="content" source="./media/concepts-data-encryption/postgresql-data-encryption-overview.png" alt-text ="Diagram that shows an overview of Bring Your Own Key." :::

Azure Active Directory [user- assigned managed identity](../../active-directory/managed-identities-azure-resources/overview.md) will be used to connect and retrieve customer-managed key. Follow this [tutorial](../../active-directory/managed-identities-azure-resources/qs-configure-portal-windows-vm.md) to create identity.


For a PostgreSQL server to use customer-managed keys stored in Key Vault for encryption of the DEK, a Key Vault administrator gives the following **access rights** to the managed identity created above:

- **get**: For retrieving, the public part and properties of the key in the key Vault.

- **list**: For listing\iterating through keys in, the key Vault.

- **wrapKey**: To be able to encrypt the DEK. The encrypted DEK is stored in the Azure Database for PostgreSQL.

- **unwrapKey**: To be able to decrypt the DEK. Azure Database for PostgreSQL needs the decrypted DEK to encrypt/decrypt the data

The key vault administrator can also [enable logging of Key Vault audit events](../../key-vault/general/howto-logging.md?tabs=azure-cli), so they can be audited later.
> [!IMPORTANT]  
> Not providing above access rights to the Key Vault to managed identity for access to KeyVault may result in failure to fetch encryption key and subsequent failed setup of the Customer Managed Key (CMK) feature.


When the server is configured to use the customer-managed key stored in the key Vault, the server sends the DEK to the key Vault for encryptions. Key Vault returns the encrypted DEK stored in the user database. Similarly, when needed, the server sends the protected DEK to the key Vault for decryption. Auditors can use Azure Monitor to review Key Vault audit event logs, if logging is enabled.

## Requirements for configuring data encryption for Azure Database for PostgreSQL Flexible server

The following are requirements for configuring Key Vault:

- Key Vault and Azure Database for PostgreSQL Flexible server must belong to the same Azure Active Directory (Azure AD) tenant. Cross-tenant Key Vault and server interactions aren't supported. Moving the Key Vault resource afterward requires you to reconfigure the data encryption.

- The key Vault must be set with 90 days for 'Days to retain deleted vaults'. If the existing key Vault has been configured with a lower number, you'll need to create a new key vault as it can't be modified after creation.

- **Enable the soft-delete feature on the key Vault**, to protect from data loss if an accidental key (or Key Vault) deletion happens. Soft-deleted resources are retained for 90 days unless the user recovers or purges them in the meantime. The recover and purge actions have their own permissions associated with a Key Vault access policy. The soft-delete feature is off by default, but you can enable it through PowerShell or the Azure CLI (note that you can't enable it through the Azure portal).

- Enable Purge protection to enforce a mandatory retention period for deleted vaults and vault objects

- Grant the Azure Database for PostgreSQL Flexible server access to the key Vault with the get, list, wrapKey, and unwrapKey permissions using its unique managed identity.

The following are requirements for configuring the customer-managed key in Flexible Server:

- The customer-managed key to be used for encrypting the DEK can be only asymmetric, RSA 2048.

- The key activation date (if set) must be a date and time in the past. The expiration date (if set) must be a future date and time.

- The key must be in the *Enabled- state.

- If you're importing an existing key into the Key Vault, provide it in the supported file formats (`.pfx`, `.byok`, `.backup`).

### Recommendations

When you're using data encryption by using a customer-managed key, here are recommendations for configuring Key Vault:

- Set a resource lock on Key Vault to control who can delete this critical resource and prevent accidental or unauthorized deletion.

- Enable auditing and reporting on all encryption keys. Key Vault provides logs that are easy to inject into other security information and event management tools. Azure Monitor Log Analytics is one example of a service that's already integrated.

- Ensure that Key Vault and Azure Database for PostgreSQL = Flexible server reside in the same region to ensure a faster access for DEK wrap, and unwrap operations.

- Lock down the Azure KeyVault to only **disable public access** and allow only *trusted Microsoft* services to secure the resources.

    :::image type="content" source="media/concepts-data-encryption/key-vault-trusted-service.png" alt-text="Screenshot of an image of networking screen with trusted-service-with-AKV setting." lightbox="media/concepts-data-encryption/key-vault-trusted-service.png":::

> [!NOTE]
>Important to note, that after choosing **disable public access** option in Azure Key Vault networking and allowing only *trusted Microsoft* services you may see error similar to following : *You have enabled the network access control. Only allowed networks will have access to this key vault* while attempting to administer Azure Key Vault via portal through public access. This doesn't preclude ability to provide key during CMK setup or fetch keys from Azure Key Vault during server operations. 

Here are recommendations for configuring a customer-managed key:

- Keep a copy of the customer-managed key in a secure place, or escrow it to the escrow service.

- If Key Vault generates the key, create a key backup before using the key for the first time. You can only restore the backup to Key Vault.

### Accidental key access revocation from Key Vault

It might happen that someone with sufficient access rights to Key Vault accidentally disables server access to the key by:

- Revoking the Key Vault's **list**, **get**, **wrapKey**, and **unwrapKey** permissions from the identity used to retrieve key in KeyVault.

- Deleting the key.

- Deleting the Key Vault.

- Changing the Key Vault's firewall rules.

- Deleting the managed identity of the server in Azure AD.

## Monitor the customer-managed key in Key Vault

To monitor the database state, and to enable alerting for the loss of transparent data encryption protector access, configure the following Azure features:

- [Azure Resource Health](../../service-health/resource-health-overview.md): An inaccessible database that has lost access to the Customer Key shows as "Inaccessible" after the first connection to the database has been denied.
- [Activity log](../../service-health/alerts-activity-log-service-notifications-portal.md): When access to the Customer Key in the customer-managed Key Vault fails, entries are added to the activity log. You can reinstate access if you create alerts for these events as soon as possible.
- [Action groups](../../azure-monitor/alerts/action-groups.md): Define these groups to send you notifications and alerts based on your preferences.

## Restore and replicate with a customer's managed key in Key Vault

After Azure Database for PostgreSQL - Flexible Server is encrypted with a customer's managed key stored in Key Vault, any newly created server copy is also encrypted. You can make this new copy through a [PITR restore](concepts-backup-restore.md) operation or read replicas.

Avoid issues while setting up customer-managed data encryption during restore or read replica creation by following these steps on the primary and restored/replica servers:

- Initiate the restore or read replica creation process from the primary Azure Database for PostgreSQL - Flexible server.

- On the restored/replica server, you can change the customer-managed key and\or Azure Active Directory (Azure AD) identity used to access Azure Key Vault in the data encryption settings. Ensure that the newly created server is given list, wrap and unwrap permissions to the key stored in Key Vault.

-  Don't revoke the original key after restoring, as at this time we don't support key revocation after restoring CMK enabled server to another server

## Inaccessible customer-managed key condition

When you configure data encryption with a customer-managed key in Key Vault, continuous access to this key is required for the server to stay online. If the server loses access to the customer-managed key in Key Vault, the server begins denying all connections within 10 minutes. The server issues a corresponding error message, and changes the server state to *Inaccessible*.
Some of the reasons why server state can become *Inaccessible* are:

-  If you delete the KeyVault, the Azure Database for PostgreSQL - Flexible Server will be unable to access the key and will move to *Inaccessible*  state. [Recover the Key Vault](../../key-vault/general/key-vault-recovery.md) and revalidate the data encryption to make the server *Available*.
- If you delete the key from the KeyVault, the Azure Database for PostgreSQL- Flexible Server will be unable to access the key and will move to *Inaccessible* state. [Recover the Key](../../key-vault/general/key-vault-recovery.md) and revalidate the data encryption to make the server *Available*.
- If you delete [managed identity](../../active-directory/managed-identities-azure-resources/how-manage-user-assigned-managed-identities.md) from Azure AD that is used to retrieve a key from KeyVault, the Azure Database for PostgreSQL- Flexible Server will be unable to access the key and will move to *Inaccessible* state.[Recover the identity](../../active-directory/fundamentals/recover-from-deletions.md) and revalidate data encryption to make server *Available*. 
- If you revoke  the Key Vault's list, get, wrapKey, and unwrapKey access policies from the [managed identity](../../active-directory/managed-identities-azure-resources/how-manage-user-assigned-managed-identities.md) that is used to retrieve a key from KeyVault, the Azure Database for PostgreSQL- Flexible Server will be unable to access the key and will move to *Inaccessible* state. [Add required access policies](../../key-vault/general/assign-access-policy.md) to the identity in KeyVault. 
- If you set up overly restrictive Azure KeyVault firewall rules that cause Azure Database for PostgreSQL- Flexible Server inability to communicate with Azure KeyVault to retrieve keys. If you enable [KeyVault firewall](../../key-vault/general/overview-vnet-service-endpoints.md#trusted-services), make sure you check an option to *'Allow Trusted Microsoft Services to bypass this firewall.'*


## Using Data Encryption with Customer Managed Key (CMK) and Geo-redundant Business Continuity features, such as Replicas and Geo-redundant backup

Azure Database for PostgreSQL - Flexible Server supports advanced [Data Recovery (DR)](../flexible-server/concepts-business-continuity.md) features, such as [Replicas](../../postgresql/flexible-server/concepts-read-replicas.md) and [geo-redundant backup](../flexible-server/concepts-backup-restore.md).  Following are requirements for setting up data encryption with CMK and these features, additional to [basic requirements for data encryption with CMK](#requirements-for-configuring-data-encryption-for-azure-database-for-postgresql-flexible-server):

* The Geo-redundant backup encryption key needs to be the created in an Azure Key Vault (AKV) in the region where the Geo-redundant backup is stored 
* The [Azure Resource Manager (ARM) REST API](../../azure-resource-manager/management/overview.md) version for supporting Geo-redundant backup enabled CMK servers is '2022-11-01-preview'. Therefore, using [ARM templates](../../azure-resource-manager/templates/overview.md) for automation of creation of servers utilizing both encryption with CMK and geo-redundant backup features,  please use this ARM API version. 
*  Same [user managed identity](../../active-directory/managed-identities-azure-resources/how-manage-user-assigned-managed-identities.md)can't be used to authenticate for primary database Azure Key Vault (AKV) and Azure Key Vault (AKV) holding encryption key for Geo-redundant backup. To make sure that we maintain regional resiliency we recommend creating user managed identity in the same region as the geo-backups. 
* As support for Geo-redundant backup with data encryption using CMK is currently in preview, there is currently no Azure CLI support for server creation with both of these features enabled. 
* If [Read replica database](../flexible-server/concepts-read-replicas.md) is setup to be encrypted with CMK during creation, its encryption key needs to be resident in an Azure Key Vault (AKV) in the region where Read replica database resides. [User assigned identity](../../active-directory/managed-identities-azure-resources/how-manage-user-assigned-managed-identities.md) to authenticate against this Azure Key Vault (AKV) needs to be created in the same region. 


> [!NOTE]  
> CLI examples below are based on 2.45.0 version of Azure Database for PostgreSQL - Flexible Server CLI libraries

## Setup Customer Managed Key during Server Creation

### Portal

Prerequisites:

- Azure Active Directory (Azure AD) user managed identity in region where Postgres Flex Server will be created. Follow this [tutorial](../../active-directory/managed-identities-azure-resources/qs-configure-portal-windows-vm.md) to create identity.

- Key Vault with key in region where Postgres Flex Server will be created. Follow this [tutorial](../../key-vault/general/quick-create-portal.md) to create Key Vault and generate key. Follow [requirements section above](#requirements-for-configuring-data-encryption-for-azure-database-for-postgresql-flexible-server) for required Azure Key Vault settings

Follow the steps below to enable CMK while creating Postgres Flexible Server using Azure portal.

1. Navigate to Azure Database for PostgreSQL - Flexible Server create pane via Azure portal

1. Provide required information on Basics and Networking tabs

1. Navigate to Security tab. On the screen, provide Azure Active Directory (Azure AD)  identity that has access to the Key Vault and Key in Key Vault in the same region where you're creating this server

1. On Review Summary tab, make sure that you provided correct information in Security section and press Create button

1. Once it's finished, you should be able to navigate to Data Encryption  screen for the server and update identity or key if necessary


### CLI:

The Azure command-line interface (Azure CLI) is a set of commands used to create and manage Azure resources. The Azure CLI is available across Azure services and is designed to get you working quickly with Azure, with an emphasis on automation.

Prerequisites:

- You must have an Azure subscription and be an administrator on that subscription.

Follow the steps below to enable CMK while creating Postgres Flexible Server using Azure CLI.

1.  Create a key vault and a key to use for a customer-managed key. Also enable purge protection and soft delete on the key vault.

```azurecli-interactive
     az keyvault create -g <resource_group> -n <vault_name> --location <azure_region> --enable-purge-protection true
```

2.  In the created Azure Key Vault, create the key that will be used for the data encryption of the Azure Database for PostgreSQL - Flexible server.

```azurecli-interactive
     keyIdentifier=$(az keyvault key create --name <key_name> -p software --vault-name <vault_name> --query key.kid -o tsv)
```
3. Create Managed Identity which will be used to retrieve key from Azure Key Vault
```azurecli-interactive
 identityPrincipalId=$(az identity create -g <resource_group> --name <identity_name> --location <azure_region> --query principalId -o tsv)
```

4. Add access policy with key permissions of *wrapKey*, *unwrapKey*, *get*, *list* in Azure KeyVault to the managed identity we created above
```azurecli-interactive
az keyvault set-policy -g <resource_group> -n <vault_name>  --object-id $identityPrincipalId --key-permissions wrapKey unwrapKey get list
```
5.  Finally, lets create Azure Database for PostgreSQL - Flexible Server with CMK based encryption enabled
```azurecli-interactive
az postgres flexible-server create -g <resource_group> -n <postgres_server_name> --location <azure_region>  --key $keyIdentifier --identity <identity_name>
```


### Azure Resource Manager (ARM)
ARM templates are a form of infrastructure as code, a concept where you define the infrastructure you need to be deployed.
Using ARM templates in managing your Azure environment has many benefits, as declarative syntax removes the requirement of writing complicated deployment scripts to handle multiple deployment scenarios. For more on ARM templates see this [doc](../../azure-resource-manager/templates/overview.md)

Prerequisites:
- You must have an Azure subscription and be an administrator on that subscription.
- Key Vault with key in region where Postgres Flex Server will be created. Follow this [tutorial](../../key-vault/general/quick-create-portal.md) to create Key Vault and generate key. 

Following is an example Azure ARM template that creates server with Customer Managed Key (CMK) based encryption as defined in *dataEncryptionData* section of ARM template
```json
{
    "$schema": "http://schema.management.azure.com/schemas/2014-04-01-preview/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters":
    {
        "administratorLogin":
        {
            "type": "string"
        },
        "administratorLoginPassword":
        {
            "type": "securestring"
        },
        "target":
        {
            "type": "string"
        },
        "name":
        {
            "type": "string"
        },
        "serverEdition":
        {
            "type": "string",
            "defaultValue": "GeneralPurpose"
        },
        "storageSizeGB":
        {
            "type": "int",
            "defaultValue": 128
        },
        "haEnabled":
        {
            "type": "string",
            "defaultValue": "Disabled"
        },
        "availabilityZone":
        {
            "type": "string",
            "defaultValue": "1"
        },
        "standbyAvailabilityZone":
        {
            "type": "string",
            "defaultValue": "2"
        },
        "version":
        {
            "type": "string"
        },
        "tags":
        {
            "type": "object",
            "defaultValue":
            {}
        },
        "firewallRules":
        {
            "type": "object",
            "defaultValue":
            {}
        },
        "backupRetentionDays":
        {
            "type": "int"
        },
        "geoRedundantBackup":
        {
            "type": "string"
        },
        "vmName":
        {
            "type": "string",
            "defaultValue": "Standard_D4s_v3"
        },
        "vnetData":
        {
            "type": "object",
            "metadata":
            {
                "description": "Vnet data is an object which contains all parameters pertaining to vnet and subnet"
            },
            "defaultValue":
            {
                "virtualNetworkName": "",
                "subnetName": "",
                "privateDnsZoneName": "",
                "Network":
                {}
            }
        },
        "userAssignedIdentitity":
        {
            "type": "string",
            "defaultValue": "postgresflexi"
        },
        "managedIdentityType":
        {
            "type": "string",
            "defaultValue": "UserAssigned"
        },
        "identityData":
        {
            "type": "object",
            "defaultValue":
            {}
        },
        "dataEncryptionData":
        {
            "type": "object",
            "defaultValue":
            {}
        },
        "apiVersion":
        {
            "type": "string",
            "defaultValue": "2021-06-01"
        },
        "aadEnabled":
        {
            "type": "bool",
            "defaultValue": false
        },
        "aadData":
        {
            "type": "object",
            "defaultValue":
            {}
        },
        "authConfig":
        {
            "type": "object",
            "defaultValue":
            {}
        },
        "azSubscriptionId":
        {
            "type": "string",
            "metadata":
            {
                "description": "User subscription id"
            }
        },
        "resource_group":
        {
            "type": "string"
        },
        "cmkKeyvault":
        {
            "type": "string"
        },
        "cmkUri":
        {
            "type": "string"
        },
        "dataEncryptionType":
        {
            "type": "string"
        }
    },
    "variables":
    {
        "firewallRules": "[parameters('firewallRules').rules]",
        "identityData":
        {
            "type": "UserAssigned",
            "UserAssignedIdentities":
            {
                "[resourceId('Microsoft.ManagedIdentity/userAssignedIdentities', parameters('userAssignedIdentitity'))]":
                {}
            }
        },
        "Network":
        {
            "DelegatedSubnetResourceId": "[concat('/subscriptions/', parameters('azSubscriptionId'), '/resourceGroups/', parameters('resource_group'), '/providers/Microsoft.Network/virtualNetworks/', parameters('vnetData').virtualNetworkName, '/subnets/', parameters('vnetData').subnetName)]",
            "PrivateDnsZoneResourceId": "[concat('/subscriptions/', parameters('azSubscriptionId'), '/resourceGroups/', parameters('resource_group'), '/providers/Microsoft.Network/privateDnsZones/', parameters('vnetData').privateDnsZoneName)]",
            "PrivateDnsZoneArmResourceId": "[concat('/subscriptions/', parameters('azSubscriptionId'), '/resourceGroups/', parameters('resource_group'), '/providers/Microsoft.Network/privateDnsZones/', parameters('vnetData').privateDnsZoneName)]"
        },
        "dataEncryptionData":
        {
            "type": "[parameters('dataEncryptionType')]",
            "primaryUserAssignedIdentityId": "[resourceId('Microsoft.ManagedIdentity/userAssignedIdentities', parameters('userAssignedIdentitity'))]",
            "primaryKeyUri": "[parameters('cmkUri')]"
        }
    },
    "resources":
    [
        {
            "apiVersion": "[parameters('apiVersion')]",
            "location": "[parameters('target')]",
            "name": "[parameters('name')]",
            "identity": "[if(empty(variables('identityData')), json('null'), variables('identityData'))]",
            "properties":
            {
                "version": "[parameters('version')]",
                "administratorLogin": "[parameters('administratorLogin')]",
                "administratorLoginPassword": "[parameters('administratorLoginPassword')]",
                "Network": "[variables('Network')]",
                "availabilityZone": "[parameters('availabilityZone')]",
                "Storage":
                {
                    "StorageSizeGB": "[parameters('storageSizeGB')]"
                },
                "Backup":
                {
                    "backupRetentionDays": "[parameters('backupRetentionDays')]",
                    "geoRedundantBackup": "[parameters('geoRedundantBackup')]"
                },
                "highAvailability":
                {
                    "mode": "[parameters('haEnabled')]",
                    "standbyAvailabilityZone": "[parameters('standbyAvailabilityZone')]"
                },
                "dataencryption": "[if(empty(variables('dataEncryptionData')), json('null'), variables('dataEncryptionData'))]",
                "authConfig": "[if(empty(parameters('authConfig')), json('null'), parameters('authConfig'))]"
            },
            "sku":
            {
                "name": "[parameters('vmName')]",
                "tier": "[parameters('serverEdition')]"
            },
            "tags": "[parameters('tags')]",
            "type": "Microsoft.DBforPostgreSQL/flexibleServers"
        },
        {
            "condition": "[parameters('aadEnabled')]",
            "type": "Microsoft.Resources/deployments",
            "apiVersion": "2018-05-01",
            "name": "addAdmins",
            "dependsOn":
            [
                "[concat('Microsoft.DBforPostgreSQL/flexibleServers/', parameters('name'))]"
            ],
            "properties":
            {
                "mode": "Incremental",
                "template":
                {
                    "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
                    "contentVersion": "1.0.0.0",
                    "resources":
                    [
                        {
                            "type": "Microsoft.DBforPostgreSQL/flexibleServers/administrators",
                            "name": "[concat(parameters('name'),'/', parameters('aadData').adminSid)]",
                            "apiVersion": "[parameters('apiVersion')]",
                            "properties":
                            {
                                "tenantId": "[parameters('aadData').tenantId]",
                                "principalName": "[parameters('aadData').principalName]",
                                "principalType": "[parameters('aadData').principalType]"
                            }
                        }
                    ]
                }
            }
        },
        {
            "condition": "[greater(length(variables('firewallRules')), 0)]",
            "type": "Microsoft.Resources/deployments",
            "apiVersion": "2019-08-01",
            "name": "[concat('firewallRules-', copyIndex())]",
            "copy":
            {
                "count": "[if(greater(length(variables('firewallRules')), 0), length(variables('firewallRules')), 1)]",
                "mode": "Serial",
                "name": "firewallRulesIterator"
            },
            "dependsOn":
            [
                "[concat('Microsoft.DBforPostgreSQL/flexibleServers/', parameters('name'))]",
                "Microsoft.Resources/deployments/addAdmins"
            ],
            "properties":
            {
                "mode": "Incremental",
                "template":
                {
                    "$schema": "http://schema.management.azure.com/schemas/2014-04-01-preview/deploymentTemplate.json#",
                    "contentVersion": "1.0.0.0",
                    "resources":
                    [
                        {
                            "type": "Microsoft.DBforPostgreSQL/flexibleServers/firewallRules",
                            "name": "[concat(parameters('name'),'/',variables('firewallRules')[copyIndex()].name)]",
                            "apiVersion": "[parameters('apiVersion')]",
                            "properties":
                            {
                                "StartIpAddress": "[variables('firewallRules')[copyIndex()].startIPAddress]",
                                "EndIpAddress": "[variables('firewallRules')[copyIndex()].endIPAddress]"
                            }
                        }
                    ]
                }
            }
        },
        {
            "type": "Microsoft.DBforPostgreSQL/flexibleServers/configurations",
            "apiVersion": "2022-12-01",
            "name": "[concat(parameters('name'), '/shared_preload_libraries')]",
            "dependsOn":
            [
                "[resourceId('Microsoft.DBforPostgreSQL/flexibleServers', parameters('name'))]"
            ],
            "properties":
            {
                "value": "pgaudit",
                "source": "user-override"
            }
        },
        {
            "type": "Microsoft.DBforPostgreSQL/flexibleServers/configurations",
            "apiVersion": "2022-12-01",
            "name": "[concat(parameters('name'), '/pgaudit.log')]",
            "dependsOn":
            [
                "[resourceId('Microsoft.DBforPostgreSQL/flexibleServers', parameters('name'))]"
            ],
            "properties":
            {
                "value": "all",
                "source": "user-override"
            }
        }
    ]
}
```
## Update Customer Managed Key on the CMK enabled Flexible Server

### Portal

Prerequisites:

- Azure Active Directory (Azure AD) user-managed identity in region where Postgres Flex Server will be created. Follow this [tutorial](../../active-directory/managed-identities-azure-resources/qs-configure-portal-windows-vm.md) to create identity.

- Key Vault with key in region where Postgres Flex Server will be created. Follow this [tutorial](../../key-vault/general/quick-create-portal.md) to create Key Vault and generate key.

Follow the steps below to update CMK on CMK enabled Flexible Server using Azure portal:

1. Navigate to Azure Database for PostgreSQL - Flexible Server create a page via the Azure portal.

1. Navigate to Data Encryption screen under Security tab

1. Select different identity to connect to Azure Key Vault, remembering that this identity needs to have proper access rights to the Key Vault

1. Select different key by choosing subscription, Key Vault and key from dropdowns provided.


### CLI

The Azure command-line interface (Azure CLI) is a set of commands used to create and manage Azure resources. The Azure CLI is available across Azure services and is designed to get you working quickly with Azure, with an emphasis on automation.


Prerequisites:
- You must have an Azure subscription and be an administrator on that subscription.
- Key Vault with key in region where Postgres Flex Server will be created. Follow this [tutorial](../../key-vault/general/quick-create-portal.md) to create Key Vault and generate key. 

Follow the steps below to change\rotate key or identity after creation of server with data encryption. 
1. Change key/identity  for data encryption for existing server, first lets get new key identifier
```azurecli-interactive
 newKeyIdentifier=$(az keyvault key show --vault-name <vault_name> --name <key_name>  --query key.kid -o tsv)
```
2. Update server with new key and\or identity
```azurecli-interactive
  az postgres flexible-server update --resource-group <resource_group> --name <server_name> --key $newKeyIdentifier --identity <identity_name>
```





## Limitations

The following are current limitations for configuring the customer-managed key in Flexible Server:

- CMK encryption can only be configured during creation of a new server, not as an update to the existing Flexible Server. You can [restore PITR backup to new server with CMK encryption](./concepts-backup-restore.md#point-in-time-recovery) instead. 

- Once enabled, CMK encryption can't be removed. If customer desires to remove this feature, it can only be done via [restore of the server to non-CMK server](./concepts-backup-restore.md#point-in-time-recovery).

- No support for Azure HSM Key Vault 

## Next steps

- [Azure Active Directory](../../active-directory-domain-services/overview.md)
