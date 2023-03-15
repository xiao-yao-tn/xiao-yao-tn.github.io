---
layout: post
title:  "Use Bicep to deploy resources and update setting"
date:   2023-03-17 09:00:00 +0900
categories: HandsOn
tags: MySQL Bicep
comments: 1
---

#### Introduction
##### what is Bicep
* Bicep is a domain-specific language (DSL) that uses declarative syntax to deploy Azure resources.
* Bicep has simpler syntax compared to JSON.
* Official [wiki](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview?tabs=bicep).

##### Bicep and ARM
* Both Bicep and ARM are using REST API.
* Almost anything that can be done in an ARM Template can also be done in Bicep. 
* Bicep is simply an easier and less error-prone process to generate ARM templates.
  > Attention: When using bicep to deploy or update, the opertaion will be recorded by activity log.

#### Get started with Bicep
##### Prerequirement
* Visual Studio Code [install]([https://code.visualstudio.com/]).
* Bicep extension for Visual Studio Code.
* Azure Tool Extension for Visual Studio Code.

##### Deploy a resource
In this part, we are trying to deploy a private access MySQL flexible server.

* Create a <em>main.bicep</em> file on Visual Studio Code 
* Paste below sample code into it.
  
  >Attention: In this code, we will deploy 5 resources : Vnet, Subnet, PrivateDNS zone, Vnetlink, MySQL flexible server. 

  >Tips: You can check the resource dependency by click "Open bicep visualizer" on the upper right corner.


{% highlight shell %}
//Pass the value to each parameter at the beginning. 

//@description can be used to briefly describe the parameter.
@description('Server Name for Azure database for MySQL')
param serverName string = 'bicepmyfs'

@description('Name for DNS Private Zone')
param dnsZoneName string = 'bicepmyfsdnszone'

@description('Fully Qualified DNS Private Zone')
param dnsZoneFqdn string = '${dnsZoneName}.private.mysql.database.azure.com'

@description('Database administrator login name')
@minLength(1)
param administratorLogin string = 'admin'

@description('Database administrator password')
@minLength(8)
@secure()
param administratorLoginPassword string 

@description('Azure database for MySQL sku name ')
param skuName string = 'Standard_B1s'

@description('Azure database for MySQL storage Size ')
param StorageSizeGB int = 20

@description('Azure database for MySQL storage Iops')
param StorageIops int = 360

@description('Azure database for MySQL pricing tier')
@allowed([
'GeneralPurpose'
'MemoryOptimized'
'Burstable'
])
param SkuTier string = 'Burstable'

@description('MySQL version')
@allowed([
'5.7'
'8.0.21'
])
param mysqlVersion string = '8.0.21'

@description('Location for all resources.')
param location string = resourceGroup().location

@description('MySQL Server backup retention days')
param backupRetentionDays int = 7

@description('Geo-Redundant Backup setting')
param geoRedundantBackup string = 'Disabled'

@description('Virtual Network Name')
param virtualNetworkName string = 'bicepvnet'

@description('Subnet Name')
param subnetName string = 'bicepsub'

@description('Virtual Network Address Prefix')
param vnetAddressPrefix string = '10.0.0.0/24'

@description('Subnet Address Prefix')
param mySqlSubnetPrefix string = '10.0.0.0/28'

@description('Composing the subnetId')

//Create Vnet
resource vnet 'Microsoft.Network/virtualNetworks@2021-05-01' = {
    name: virtualNetworkName
    location: location
    properties: {
        addressSpace: {
        addressPrefixes: [
            vnetAddressPrefix
        ]
        }
    }
}

//Create subnet
resource subnet 'Microsoft.Network/virtualNetworks/subnets@2022-07-01' = {
    name: subnetName
    parent: vnet
    properties: {
        addressPrefix: mySqlSubnetPrefix
        delegations: [
            {
                name: 'dlg-Microsoft.DBforMySQL-flexibleServers'
                properties: {
                    serviceName: 'Microsoft.DBforMySQL/flexibleServers'
                }
            }
        ]
        privateEndpointNetworkPolicies: 'Enabled'
        privateLinkServiceNetworkPolicies: 'Enabled'
    }
}


//Create DNS zone
resource dnszone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
    name: dnsZoneFqdn
    location: 'global'
}

//Create Vnetlink int dns zone
resource vnetLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
    name: vnet.name
    parent: dnszone
    location: 'global'
    properties: {
        registrationEnabled: false
        virtualNetwork: {
        id: vnet.id
        }
    }
}

//Create mysql flexible server
resource server 'Microsoft.DBforMySQL/flexibleServers@2021-05-01' = {
    name: serverName
    dependsOn: [
        vnetLink
    ]
    location: location
    sku: {
        name: skuName
        tier: SkuTier
    }
    properties: {
        administratorLogin: administratorLogin
        administratorLoginPassword: administratorLoginPassword
        storage: {
            autoGrow: 'Enabled'
            iops: StorageIops
            storageSizeGB: StorageSizeGB
        }
        createMode: 'Default'
        version: mysqlVersion
        backup: {
            backupRetentionDays: backupRetentionDays
            geoRedundantBackup: geoRedundantBackup
        }
        highAvailability: {
            mode: 'Disabled'
        }
        network: {
            delegatedSubnetResourceId: subnet.id
            privateDnsZoneResourceId: dnszone.id
        }
    }
}

{% endhighlight %}

* Right click the <em>main.bicep</em> file choose "Deploy Bicep File".
* Enter the parameter value you want to change.


##### Update a resource
In this part, we will update some setting of this MySQL flexible server.

* Create a <em>update.bicep</em> file on Visual Studio Code 
  In this file, input the properties that you want to update. You can check the details about allowed parameter/property [here]([https://learn.microsoft.com/en-us/azure/templates/microsoft.dbformysql/flexibleservers?pivots=deployment-language-bicep])
* For example, if we want to change the backup retention days to 15, we can paste below sample code into the file (change the server name accordingly).

{% highlight shell %}
//localtion and server name are needed.
param location string = resourceGroup().location
resource mysql 'Microsoft.DBforMySQL/flexibleServers@2021-05-01' = {
    name: 'bicepmyfs'
    location: location
    properties: {
        backup: {
            backupRetentionDays: 15
        }
    }
}
{% endhighlight %}

* Right click the <em>update.bicep</em> file choose "Deploy Bicep File".
* Enter the required information.

