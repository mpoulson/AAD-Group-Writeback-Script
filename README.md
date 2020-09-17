# AAD Group Writeback Script

This repository contains a script that can take certain groups in an Azure Active Directory, defined by a scope, writing them back to onpremises Active Directory, including group memberships.

## Setup

- Download or clone this repository into a folder on one of your servers
- Install the Active Directory PowerShell module 

```
Install-WindowsFeature RSAT-AD-PowerShell
```
- Configure authentication (See section 'Authenticating to Azure AD')




## Invocation

The script is invoked using Run.ps1, with the ConfigFile parameter. If Run.ps1 is run without parameter, "Run.config" is the default value.

### Normal invocation

Run.config will be used.

```
.\Run.ps1
```

### Normal invocation defining config file as parameter

```
.\Run.ps1 -ConfigFile Config.config
```

### Multiple config files

The following can be useful if you have multple Azure ADs or limitations in scoping. Remember that the script should then have one destionation OU per config file in order for deletion to work properly.

```
"Tenant1.config","Tenant2.config","Tenant3.config" | .\Run.ps1
```

## Configuration

| Configuration option        | Required? | Description                                                 | Valid values                        |
| --------------------------- | --------- | ----------------------------------------------------------- | ----------------------------------- |
| AuthenticationMethod        | Yes       |                                                             | MSI / ClientCredentials             |
| ClientID                    | 1         | The clientid used for the Client Credential Grant           | A guid                              |
| EncryptedSecret             | 1         | The client secret used for the Client Credential Grant      | Any string                          |
| TenantID                    | 1         | The tenantid used for the Client Credential Grant           | A guid                              |
| DestinationOU               | Yes       | The OU where groups will be created                         | OU=Groups,DC=contoso,DC=com         |
| ADGroupObjectIDAttribute    | Yes       | The AD attribute for source anchor (objectid of AAD group)  | A string attribute in AD (info, description, extensionAttribute1, etc) |
| AADGroupScopingMethod       | Yes       | The method of determining which AAD groups to sync          | Filter / GroupMemberOfGroup / PrivilegedGroups |
| AADGroupScopingConfig       | 2         | Additional info required to determine groups (filter etc.)  | id eq '<objectid>'                  |
| GroupDeprovisioningMethod   | Yes       | Determines what to do when source AAD group is deleted      | Delete / ConvertToDistributionGroup / PrintWarning / DoNothing |

1 - If AuthenticationMethod is ClientCredentials
2 - If AADGroupScopingMethod is GroupMemberOfGroup or AADGroupScopingConfig

### Example configurations

The configuration file is a JSON based, and contains a dictionary with key-value pairs. There are three example configuration files provided:

| File | Description |
| - | - |
| Example1.config | Using client credentials to authenticate to Azure AD, writing all privileged groups back to AD. If groups are deleted from Azure AD, a list of warnings are printed. |
| Example2.config | Using Managed Service Identity to authenticate to Azure AD, writing a filtered list of groups back to AD. If groups are deleted from Azure AD, the AD group will be converted to a distribution group (which does not give any access, and is an effective disable method). |
| Example3.config | Using Managed Service Identity to authenticate to Azure AD, writing all groups that are member of the group '5f7ab793-e722-435a-a8bf-ac48a3f7361e' back to AD. If groups are deleted from Azure AD, the AD group will be deleted. |

## Authenticating to Azure AD

There are two supported methods to authenticate to Azure AD - MSI and Client Credentials.

### Managed Service Identity (MSI)

If you are running the write-back script on a virtual machine in Azure, that is running in a subscription that is connected to your Azure AD tenant, you simples option is to enable MSI for your VM. 

1. On your Virtual Machine in Azure, find "Identity" and enable System Assigned Managed Identity

!(media/MSI.png)

2. Copy the ObjectID, and run the following to grant Group.Read.All access to the MSI:

```
$ObjectID = "<guid>"
Install-Module AzureAD
$graph = Get-AzureADServicePrincipal -Filter "AppId eq '00000003-0000-0000-c000-000000000000'" 
$groupReadAll = $graph.AppRoles | ? Value -eq "Group.Read.All"
New-AzureADServiceAppRoleAssignment -Id $groupReadAll.Id -PrincipalId $ObjectId -ResourceId $graph.ObjectId
```

Now, in your config file, set AuthenticationMethod to "MSI". No need to provide ClientID, TenantID or EncryptedSecret.

## Client Credentials

If you cannot use MSI, this is the way to go.

