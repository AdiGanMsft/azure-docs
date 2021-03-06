---
title: Manage registered servers with Azure File Sync (preview) | Microsoft Docs
description: Learn how to register and unregister a Windows Server with an Azure File Sync Storage Sync Service.
services: storage
documentationcenter: ''
author: wmgries
manager: klaasl
editor: jgerend

ms.assetid: 297f3a14-6b3a-48b0-9da4-db5907827fb5
ms.service: storage
ms.workload: storage
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 12/04/2017
ms.author: wgries
---

# Manage registered servers with Azure File Sync (preview)
Azure File Sync (preview) allows you to centralize your organization's file shares in Azure Files without giving up the flexibility, performance, and compatibility of an on-premises file server. It does this by transforming your Windows Servers into a quick cache of your Azure File share. You can use any protocol available on Windows Server to access your data locally (including SMB, NFS, and FTPS) and you can have as many caches as you need across the world.

The following article illustrates how to register and manage a server with a Storage Sync Service. See [How to deploy Azure File Sync (preview)](storage-sync-files-deployment-guide.md) for information on how to deploy Azure File Sync end-to-end.

## Register/unregister a server with Storage Sync Service
Registering a server with Azure File Sync establishes a trust relationship between Windows Server and Azure. This relationship can then be used to create *server endpoints* on the server, which represent specific folders that should be synced with an Azure file share (also known as a *cloud endpoint*). 

### Prerequisites
To register a server with a Storage Sync Service, you must first prepare your server with the necessary prerequisites:

* Your server must be running a supported version of Windows Server. For more information, see [Supported versions of Windows Server](storage-sync-files-planning.md#supported-versions-of-windows-server).
* Ensure that a Storage Sync Service has been deployed. For more information on how to deploy a Storage Sync Service, see [How to deploy Azure File Sync (preview)](storage-sync-files-deployment-guide.md).
* Ensure that the server is connected to the internet and that Azure is accessible.
* Disable the IE Enhanced Security Configuration for administrators with the Server Manager UI.
    
    ![Server Manager UI with the IE Enhanced Security Configuration highlighted](media/storage-sync-files-server-registration/server-manager-ie-config.png)

* Ensure that the AzureRM PowerShell module is installed on your server. If your server is a member of a Failover Cluster, every node in the cluster will require the AzureRM module. More details on how to install the AzureRM module can be found on the [Install and configure Azure PowerShell](https://docs.microsoft.com/powershell/azure/install-azurerm-ps).

    > [!Note]  
    > We recommend using the newest version of the AzureRM PowerShell module to register/unregister a server. If the AzureRM package has been previously installed on this server (and the PowerShell version on this server is 5.* or greater), you can use the `Update-Module` cmdlet to update this package. 

### Register a server with Storage Sync Service
Before a server can be used as a *server endpoint* in an Azure File Sync *sync group*, it must be registered with a *Storage Sync Service*. A server can only be registered with one Storage Sync Service at a time.

#### Install the Azure File Sync agent
1. [Download the Azure File Sync agent](https://go.microsoft.com/fwlink/?linkid=858257).
2. Start the Azure File Sync agent installer.
    
    ![The first pane of the Azure File Sync agent installer](media/storage-sync-files-server-registration/install-afs-agent-1.png)

3. Be sure to enable updates to the Azure File Sync agent using Microsoft Update. It is important because critical security fixes and feature enhancements to the server package are shipped via Microsoft Update.

    ![Ensure Microsoft Update is enabled in the Microsoft Update pane of the Azure File Sync agent installer](media/storage-sync-files-server-registration/install-afs-agent-2.png)

4. If the server has not been previously registered, the server registration UI will pop up immediately after completing the installation.

> [!Important]  
> If the server is a member of a Failover Cluster, the Azure File Sync agent needs to be installed on every node in the cluster.

#### Register the server using the server registration UI
> [!Important]  
> Cloud Solution Provider (CSP) subscriptions cannot use the server registration UI. Instead, use PowerShell (below this section).

1. If the server registration UI did not start immediately after completing the installation of the Azure File Sync agent, it can be started manually by executing `C:\Program Files\Azure\StorageSyncAgent\ServerRegistration.exe`.
2. Click *Sign-in* to access your Azure subscription. 

    ![Opening dialog of the server registration UI](media/storage-sync-files-server-registration/server-registration-ui-1.png)

3. Pick the correct subscription, resource group, and Storage Sync Service from the dialog.

    ![Storage Sync Service information](media/storage-sync-files-server-registration/server-registration-ui-2.png)

4. In preview, one more sign-in is required to complete the process. 

    ![Sign in dialog](media/storage-sync-files-server-registration/server-registration-ui-3.png)

> [!Important]  
> If the server is a member of a Failover Cluster, each server needs to run the Server Registration. When you view the registered servers in the Azure Portal, Azure File Sync automatically recognizes each node as a member of the same Failover Cluster, and groups them together appropriately.

#### Register the server with PowerShell
You can also perform server registration via PowerShell. This is the only supported way of server registration for Cloud Solution Provider (CSP) subscriptions:

```PowerShell
Import-Module "C:\Program Files\Azure\StorageSyncAgent\StorageSync.Management.PowerShell.Cmdlets.dll"
Login-AzureRmStorageSync -SubscriptionID "<your-subscription-id>" -TenantID "<your-tenant-id>"
Register-AzureRmStorageSyncServer -SubscriptionId "<your-subscription-id>" - ResourceGroupName "<your-resource-group-name>" - StorageSyncService "<your-storage-sync-service-name>"
```

### Unregister the server with Storage Sync Service
There are several steps that are required to unregister a server with a Storage Sync Service. Let's take a look at how to properly unregister a server.

#### (Optional) Recall all tiered data
When enabled for a server endpoint, cloud tiering will *tier* files to your Azure File shares. This enables on-premises file shares to act as a cache, rather than a complete copy of the dataset, to make efficient use of the space on the file server. However, if a server endpoint is removed with tiered files still locally on the server, those files will become unaccessible. Therefore, if continued file access is desired, you must recall all tiered files from Azure Files before continuing with deregistration. 

This can be done with the PowerShell cmdlet as shown below:

```PowerShell
Import-Module "C:\Program Files\Azure\StorageSyncAgent\StorageSync.Management.ServerCmdlets.dll"
Invoke-StorageSyncFileRecall -Path <path-to-to-your-server-endpoint>
```

> [!Warning]  
> If the local volume hosting the server endpoint does not have enough free space to recall all the tiered data, the `Invoke-StorageSyncFileRecall` cmdlet will fail.  

#### Remove the server from all sync groups
Before unregistering the server on the Storage Sync Service, all server endpoints on that server must be removed. This can be done via the Azure portal:

1. Navigate to the Storage Sync Service where your server is registered.
2. Remove all server endpoints for this server in each sync group in the Storage Sync Service. This can be accomplished by right-clicking the relevant server endpoint in the sync group pane.

    ![Removing a server endpoint from a sync group](media/storage-sync-files-server-registration/sync-group-server-endpoint-remove-1.png)

This can also be accomplished with a simple PowerShell script:

```PowerShell
Import-Module "C:\Program Files\Azure\StorageSyncAgent\StorageSync.Management.PowerShell.Cmdlets.dll"

$accountInfo = Login-AzureRmAccount
Login-AzureRmStorageSync -SubscriptionId $accountInfo.Context.Subscription.Id -TenantId $accountInfo.Context.Tenant.Id -ResourceGroupName "<your-resource-group>"

$StorageSyncService = "<your-storage-sync-service>"

Get-AzureRmStorageSyncGroup -StorageSyncServiceName $StorageSyncService | ForEach-Object { 
    $SyncGroup = $_; 
    Get-AzureRmStorageSyncServerEndpoint -StorageSyncServiceName $StorageSyncService -SyncGroupName $SyncGroup.Name | Where-Object { $_.DisplayName -eq $env:ComputerName } | ForEach-Object { 
        Remove-AzureRmStorageSyncServerEndpoint -StorageSyncServiceName $StorageSyncService -SyncGroupName $SyncGroup.Name -ServerEndpointName $_.Name 
    } 
}
```

#### Unregister the server
Now that all data has been recalled and the server has been removed from all sync groups, the server can be unregistered. 

1. In the Azure portal, navigate to the *Registered servers* section of the Storage Sync Service.
2. Right-click on the server you want to unregister and click "Unregister Server".

    ![Unregister server](media/storage-sync-files-server-registration/unregister-server-1.png)

## Ensuring Azure File Sync is a good neighbor in your datacenter 
Since Azure File Sync will rarely be the only service running in your datacenter, you may want to limit the network and storage usage of Azure File Sync.

> [!Important]  
> Setting limits too low will impact the performance of Azure File Sync synchronization and recall.

### Set Azure File Sync network limits
You can constrain the network utilitization of Azure File Sync by using the 'StorageSyncNetworkLimit` cmdlets. 

For example, you can create a new network limit to ensure that Azure File Sync does not use more than 10 Mbps between 9 am and 5 pm (17:00h) during the work week: 

```PowerShell
Import-Module "C:\Program Files\Azure\StorageSyncAgent\StorageSync.Management.ServerCmdlets.dll"
New-StorageSyncNetworkLimit -Day Monday, Tuesday, Wednesday, Thursday, Friday -StartHour 9 -EndHour 17 -LimitKbps 10000
```

You can see your limit by using the following cmdlet:

```PowerShell
Get-StorageSyncNetworkLimit # assumes StorageSync.Management.ServerCmdlets.dll is imported
```

To remove network limits, use `Remove-StorageSyncNetworkLimit`. For example, the following command removes all network limits:

```PowerShell
Get-StorageSyncNetworkLimit | ForEach-Object { Remove-StorageSyncNetworkLimit -Id $_.Id } # assumes StorageSync.Management.ServerCmdlets.dll is imported
```

### Use Windows Server storage QoS 
When Azure File Sync is hosted in a virtual machine running on a Windows Server virtualization host, you can use Storage QoS (storage quality of service) to regulate storage IO consumption. The Storage QoS policy can be set either as a maximum (or limit, like how StorageSyncNetwork limit is enforced above) or as a minimum (or reservation). Setting a minimum instead of a maximum allows Azure File Sync to burst to use available storage bandwidth if other workloads are not using it. For more information, see [Storage Quality of Service](https://docs.microsoft.com/windows-server/storage/storage-qos/storage-qos-overview).

## See also
- [Planning for an Azure File Sync (preview) deployment](storage-sync-files-planning.md)
- [Deploy Azure File Sync (preview)](storage-sync-files-deployment-guide.md) 
- [Troubleshoot Azure File Sync (preview)](storage-sync-files-troubleshoot.md)