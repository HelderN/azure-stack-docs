---
title: Update Azure Stack HCI clusters
description: How to apply operating system and firmware updates to Azure Stack HCI using Windows Admin Center and PowerShell.
author: khdownie
ms.author: v-kedow
ms.topic: how-to
ms.date: 07/21/2020
---

# Update Azure Stack HCI clusters

> Applies to: Azure Stack HCI, version 20H2; Windows Server 2019

When updating Azure Stack HCI clusters, the goal is to maintain availability by updating only one server in the cluster at a time. Many operating system updates require taking the server offline, for example to do a restart or to update software such as the network stack. We recommend using [Cluster-Aware Updating (CAU)](/windows-server/failover-clustering/cluster-aware-updating), a feature that makes it easier to install Windows updates to every server in your cluster while keeping your applications running by automating the software updating process. Cluster-Aware Updating can be used on all editions of Windows Server, including Server Core installations, and can be initiated either through Windows Admin Center or using PowerShell.

## Update a cluster using Windows Admin Center

Windows Admin Center makes it easy to update a cluster and apply operating system and solution updates using a simple user interface. If you've purchased an integrated system from a Microsoft hardware partner, it’s easy to get the latest drivers, firmware, and other updates directly from Windows Admin Center by installing the appropriate partner update extension(s). ​If your hardware was not purchased as an integrated system, firmware and driver updates may need to be performed separately, following the hardware vendor's recommendations.

1. When you connect to a cluster, the Windows Admin Center dashboard will alert you if one or more servers have updates ready to be installed, and provide a link to update now. Alternatively, you can select **Updates** from the **Tools** menu at the left.
1. To use the Cluster-Aware updating tool in Windows Admin Center, you must enable Credential Security Service Provider (CredSSP) and provide explicit credentials. When asked if CredSSP should be enabled, click **Yes**.
1. Specify your username and password, and click **Continue**.
1. Any available updates will be displayed; click **Check Available Updates** to refresh the list.
1. Select the updates you wish to install and click **Apply All Updates**. This will install the updates on every server in the cluster. If a restart is needed, cluster roles such as virtual machines will be moved to another server first to prevent any disruption.
1. To improve security, disable CredSSP as soon as you're finished installing the updates:
    - In Windows Admin Center, under **All connections**, select the first server in your cluster, and then select **Connect**.
    - On the **Overview** page, select **Disable CredSSP**, and then on the **Disable CredSSP** pop-up window, select **Yes**.

## Update a cluster using PowerShell

Before you can update a cluster using Cluster-Aware Updating, you first need to install the **Failover Clustering Tools**, which are part of the **Remote Server Administration Tools (RSAT)** and include the Cluster-Aware Updating software. If you're updating an existing cluster, these tools may already be installed.

To test whether a failover cluster is properly set up to apply software updates using Cluster-Aware Updating, run the **Test-CauSetup** PowerShell cmdlet, which performs a Best Practices Analyzer (BPA) scan of the failover cluster and network environment and alerts you of any warnings or errors:

```PowerShell
Test-CauSetup -ClusterName Cluster1
```

If you need to install features, tools, or roles, see the next sections. Otherwise, skip ahead to [Check for updates with PowerShell](#check-for-updates-with-powershell).

### Install Failover Clustering and Failover Clustering Tools using PowerShell

To check if a cluster or server has the Failover Clustering feature and Failover Clustering Tools already installed, issue the **`Get-WindowsFeature`** PowerShell cmdlet from your management PC (or run it directly on the cluster or server, omitting the -ComputerName parameter):

```PowerShell
Get-WindowsFeature -Name Failover*, RSAT-Clustering* -ComputerName Server1
```

Make sure "Install State" says Installed and that an X appears before both Failover Clustering and Failover Cluster Module for Windows PowerShell:

```
Display Name                                            Name                       Install State
------------                                            ----                       -------------
[X] Failover Clustering                                 Failover-Clustering            Installed
        [X] Failover Clustering Tools                   RSAT-Clustering                Installed
            [X] Failover Cluster Module for Windows ... RSAT-Clustering-Powe...        Installed
            [ ] Failover Cluster Automation Server      RSAT-Clustering-Auto...        Available
            [ ] Failover Cluster Command Interface      RSAT-Clustering-CmdI...        Available
```

If the Failover Clustering feature is not installed, install it on each server in the cluster with the **`Install-WindowsFeature`** cmdlet, using the -IncludeAllSubFeature and -IncludeManagementTools parameters:

```PowerShell
Install-WindowsFeature –Name Failover-Clustering -IncludeAllSubFeature –IncludeManagementTools -ComputerName Server1
```

This command will also install the Failover Cluster Module for PowerShell, which includes PowerShell cmdlets for managing failover clusters, and the Cluster-Aware Updating module for PowerShell, for installing software updates on failover clusters.

If the Failover Clustering feature is already installed but the Failover Cluster Module for Windows PowerShell is not, simply install it on each server in the cluster with the **Install-WindowsFeature** cmdlet:

```PowerShell
Install-WindowsFeature –Name RSAT-Clustering-PowerShell -ComputerName Server1
```

### Choose an updating mode

Cluster-Aware Updating can coordinate the complete cluster updating operation in two modes:  
  
-   **Self-updating mode** For this mode, the Cluster-Aware Updating clustered role is configured as a workload on the failover cluster that is to be updated, and an associated update schedule is defined. The cluster updates itself at scheduled times by using a default or custom updating run profile. During the updating run, the Cluster-Aware Updating Update Coordinator process starts on the node that currently owns the Cluster-Aware Updating clustered role, and the process sequentially performs updates on each cluster node. To update the current cluster node, the Cluster-Aware Updating clustered role fails over to another cluster node, and a new Update Coordinator process on that node assumes control of the updating run. In self-updating mode, Cluster-Aware Updating can update the failover cluster by using a fully automated, end-to-end updating process. An administrator can also trigger updates on-demand in this mode, or simply use the remote-updating approach if desired.
  
-   **Remote updating mode** For this mode, a remote management computer (usually a Windows 10 PC) that has network connectivity to the failover cluster but is not a member of the failover cluster is configured with the Failover Clustering Tools. From the remote management computer, called the Update Coordinator, the administrator triggers an on-demand updating run by using a default or custom updating run profile. Remote updating mode is useful for monitoring real-time progress during the updating run, and for clusters that are running on Server Core installations.  


   > [!NOTE]
   > Starting with Windows 10 October 2018 Update, RSAT is included as a set of "Features on Demand" right from Windows 10. Simply go to **Settings > Apps > Apps & features > Optional features > Add a feature > RSAT: Failover Clustering Tools**, and select **Install**. To see installation progress, click the Back button to view status on the "Manage optional features" page. The installed feature will persist across Windows 10 version upgrades. To install RSAT for Windows 10 prior to the October 2018 Update, [download an RSAT package](https://www.microsoft.com/en-us/download/details.aspx?id=45520).

### Add CAU cluster role to the cluster

The Cluster-Aware Updating cluster role is required for self-updating mode. If you're using Windows Admin Center to perform the updates, the cluster role will automatically be added.

The **`Get-CauClusterRole`** cmdlet displays the configuration properties of the Cluster-Aware Updating cluster role on the specified cluster.

```PowerShell
Get-CauClusterRole -ClusterName Cluster1
```

If the role is not yet configured on the cluster, you will see the following error message:

```Get-CauClusterRole : The current cluster is not configured with a Cluster-Aware Updating clustered role.```

To add the Cluster-Aware Updating cluster role for self-updating mode using PowerShell, use the **`Add-CauClusterRole`** cmdlet and supply the appropriate [parameters](/powershell/module/clusterawareupdating/add-cauclusterrole?view=win10-ps#parameters), as in the following example:

```PowerShell
Add-CauClusterRole -ClusterName Cluster1 -MaxFailedNodes 0 -RequireAllNodesOnline -EnableFirewallRules -VirtualComputerObjectName Cluster1-CAU -Force -CauPluginName Microsoft.WindowsUpdatePlugin -MaxRetriesPerNode 3 -CauPluginArguments @{ 'IncludeRecommendedUpdates' = 'False' } -StartDate "3/2/2020 3:00:00 AM" -DaysOfWeek 4 -WeeksOfMonth @(3) -verbose
```

   > [!NOTE]
   > The above command must be run from a management PC or domain controller.

### Enable firewall rules to allow remote restarts

You'll need to allow the servers to restart remotely during the update process. If you're using Windows Admin Center to perform the updates, Windows Firewall rules will automatically be updated on each server to allow remote restarts. If you're updating with PowerShell, either enable the Remote Shutdown firewall rule group in Windows Firewall, or pass the -EnableFirewallRules parameter to the cmdlet such as in the example above.

## Check for updates with PowerShell

You can use the **`Invoke-CAUScan`** cmdlet to scan servers for applicable updates and get a list of the initial set of updates that are applied to each server in a specified cluster:

```PowerShell
Invoke-CauScan -ClusterName Cluster1 -CauPluginName Microsoft.WindowsUpdatePlugin -Verbose
```

Generation of the list can take a few minutes to complete. The preview list includes only an initial set of updates; it does not include updates that might become applicable after the initial updates are installed.

## Install updates with PowerShell

To scan servers for applicable updates and perform a full updating run on the specified cluster, use the **`Invoke-CAURun`** cmdlet:

```PowerShell
Invoke-CauRun -ClusterName Cluster1 -CauPluginName Microsoft.WindowsUpdatePlugin -MaxFailedNodes 1 -MaxRetriesPerNode 3 -RequireAllNodesOnline -EnableFirewallRules -Force
```

This command performs a scan and a full updating run on the cluster named Cluster1. This cmdlet uses the Microsoft.WindowsUpdatePlugin plug-in and requires that all cluster nodes be online before running this cmdlet. In addition, this cmdlet allows no more than three retries per node before marking the node as failed, and allows no more than one node to fail before marking the entire updating run as failed. It also enables firewall rules to allow the servers to restart remotely. Because the command specifies the Force parameter, the cmdlet runs without displaying confirmation prompts.

The updating run process includes the following: 
- Scanning for and downloading applicable updates on each server in the cluster
- Moving currently running clustered roles off each server
- Installing the updates on each server
- Restarting the server if required by the installed updates
- Moving the clustered roles back to the original server

The updating run process also includes ensuring that quorum is maintained, checking for additional updates that can only be installed after the initial set of updates are installed, and saving a report of the actions taken.

## Check on the status of an updating run

An administrator can get summary information about an updating run in progress by running the **`Get-CauRun`** cmdlet:

```PowerShell
Get-CauRun -ClusterName Cluster1
```

Here's some sample output:

```
RunId                   : 834dd11e-584b-41f2-8d22-4c9c0471dbad 
RunStartTime            : 10/13/2019 1:35:39 PM 
CurrentOrchestrator     : NODE1 
NodeStatusNotifications : { 
Node      : NODE1 
Status    : Waiting 
Timestamp : 10/13/2019 1:35:49 PM 
} 
NodeResults             : { 
Node                     : NODE2 
Status                   : Succeeded 
ErrorRecordData          : 
NumberOfSucceededUpdates : 0 
NumberOfFailedUpdates    : 0 
InstallResults           : Microsoft.ClusterAwareUpdating.UpdateInstallResult[] 
}
```

## Next steps

For related information, see also:

- [Updating drive firmware in Storage Spaces Direct](/windows-server/storage/update-firmware)
- [Validate an Azure Stack HCI cluster](../deploy/validate.md)
