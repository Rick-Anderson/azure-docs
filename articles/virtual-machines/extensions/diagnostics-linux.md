---
title: Azure Compute - Linux Diagnostic Extension 4.0
description: How to configure the Azure Linux Diagnostic Extension (LAD) 4.0 to collect metrics and log events from Linux VMs running in Azure.
ms.topic: article
ms.service: virtual-machines
ms.subservice: extensions
author: amjads1
ms.author: amjads
ms.collection: linux
ms.date: 02/05/2021

---
# Use Linux Diagnostic Extension 4.0 to monitor metrics and logs

This document describes version 4.0 and newer of the Linux Diagnostic Extension.

> [!IMPORTANT]
> For information about version 3.*, see  [this document](./diagnostics-linux-v3.md). 
> For information about version 2.3 and older, see [this document](/previous-versions/azure/virtual-machines/linux/classic/diagnostic-extension-v2).

## Introduction

The Linux Diagnostic Extension helps a user monitor the health of a Linux VM running on Microsoft Azure. It has the following capabilities:

* Collects system performance metrics from the VM and stores them in a specific table in a designated storage account.
* Retrieves log events from syslog and stores them in a specific table in the designated storage account.
* Enables users to customize the data metrics that are collected and uploaded.
* Enables users to customize the syslog facilities and severity levels of events that are collected and uploaded.
* Enables users to upload specified log files to a designated storage table.
* Supports sending metrics and log events to arbitrary EventHub endpoints and JSON-formatted blobs in the designated storage account.

This extension works with both Azure deployment models.

## Installing the extension in your VM

You can enable this extension by using the Azure PowerShell cmdlets, Azure CLI scripts, ARM templates, or the Azure portal. For more information, see [Extensions Features](features-linux.md).

>[!NOTE]
>Certain components of the Diagnostics VM extension are also shipped in the [Log Analytics VM extension](./oms-linux.md). Due to this architecture, conflicts can arise if both extensions are instantiated in the same ARM template. To avoid these install-time conflicts, use the [`dependsOn` directive](../../azure-resource-manager/templates/define-resource-dependency.md#dependson) to ensure the extensions are installed sequentially. The extensions can be installed in either order.

These installation instructions and a [downloadable sample configuration](https://raw.githubusercontent.com/Azure/azure-linux-extensions/master/Diagnostic/tests/lad_2_3_compatible_portal_pub_settings.json) configure LAD 4.0 to:

* capture and store the same metrics as were provided by LAD 2.3, 3*;
* send metrics to Azure Monitor Sink along with the usual sink to Azure Storage, new in Lad 4.0
* capture a useful set of file system metrics, as were provided by LAD 3.0;
* capture the default syslog collection enabled by LAD 2.3;
* enable the Azure portal experience for charting and alerting on VM metrics.

The downloadable configuration is just an example; modify it to suit your own needs.

### Supported Linux distributions

The Linux Diagnostic Extension supports the following distributions and versions. The list of distributions and versions applies only to Azure-endorsed Linux vendor images. Third-party BYOL and BYOS images, like appliances, are generally not supported for the Linux Diagnostic Extension.

A distribution that lists only major versions, like Debian 7, is also supported for all minor versions. If a specific minor version is specified, only that specific version is supported; if "+" is appended, minor versions equal to or greater than the specified version are supported.

Supported distributions and versions:

- Ubuntu 18.04, 16.04, 14.04
- CentOS 7, 6.5+
- Oracle Linux 7, 6.4+
- OpenSUSE 13.1+
- SUSE Linux Enterprise Server 12
- Debian 9, 8, 7
- RHEL 7, 6.7+

### Prerequisites

* **Azure Linux Agent version 2.2.0 or later**. Most Azure VM Linux gallery images include version 2.2.7 or later. Run `/usr/sbin/waagent -version` to confirm the version installed on the VM. If the VM is running an older version of the guest agent, follow [these instructions](./update-linux-agent.md) to update it.
* **Azure CLI**. [Set up the Azure CLI](/cli/azure/install-azure-cli) environment on your machine.
* The wget command, if you don't already have it: Run `sudo apt-get install wget`.
* An existing Azure subscription and an existing general purpose storage account to store the data in.  General purpose storage accounts support Table storage which is required.  A Blob storage account will not work.
* Python 2

### Python requirement

The Linux Diagnostic Extension requires Python 2. If your virtual machine is using a distro that doesn't include Python 2 by default then you must install it. The following sample commands will install Python 2 on different distros.	

 - Red Hat, CentOS, Oracle: `yum install -y python2`
 - Ubuntu, Debian: `apt-get install -y python2`
 - SUSE: `zypper install -y python2`

The python2 executable must be aliased to *python*. Following is one method that you can use to set this alias:

1. Run the following command to remove any existing aliases.
 
    ```
    sudo update-alternatives --remove-all python
    ```

2. Run the following command to create the alias.

    ```
    sudo update-alternatives --install /usr/bin/python python /usr/bin/python2 1
    ```

### Sample installation

> [!NOTE]
> For either of the samples, fill in the correct values for the variables in the first section before running. 

The sample configuration downloaded in these examples collects a set of standard data and sends them to table storage. The URL for the sample configuration and its contents are subject to change. In most cases, you should download a copy of the portal settings JSON file and customize it for your needs, then have any templates or automation you construct use your own version of the configuration file rather than downloading that URL each time.

> [!NOTE]
> For enabling the new Azure Monitor Sink, the VMs need to have System Assigned Identity enabled for MSI Auth token generation. This can be done during VM creation or after the VM has been created. Steps for enabling System Assigned Identity through portal, CLI, PowerShell, and resource manager.  are listed in detail [here](../../active-directory/managed-identities-azure-resources/qs-configure-portal-windows-vm.md). 

#### Azure CLI sample

```azurecli
# Set your Azure VM diagnostic variables correctly below
my_resource_group=<your_azure_resource_group_name_containing_your_azure_linux_vm>
my_linux_vm=<your_azure_linux_vm_name>
my_diagnostic_storage_account=<your_azure_storage_account_for_storing_vm_diagnostic_data>

# Should login to Azure first before anything else
az login

# Select the subscription containing the storage account
az account set --subscription <your_azure_subscription_id>

# Enable System Assigned Identity to the existing VM
az vm identity assign -g $my_resource_group -n $my_linux_vm

# Download the sample Public settings. (You could also use curl or any web browser)
wget https://raw.githubusercontent.com/Azure/azure-linux-extensions/master/Diagnostic/tests/lad_2_3_compatible_portal_pub_settings.json -O portal_public_settings.json

# Build the VM resource ID. Replace storage account name and resource ID in the public settings.
my_vm_resource_id=$(az vm show -g $my_resource_group -n $my_linux_vm --query "id" -o tsv)
sed -i "s#__DIAGNOSTIC_STORAGE_ACCOUNT__#$my_diagnostic_storage_account#g" portal_public_settings.json
sed -i "s#__VM_RESOURCE_ID__#$my_vm_resource_id#g" portal_public_settings.json

# Build the protected settings (storage account SAS token)
my_diagnostic_storage_account_sastoken=$(az storage account generate-sas --account-name $my_diagnostic_storage_account --expiry 2037-12-31T23:59:00Z --permissions wlacu --resource-types co --services bt -o tsv)
my_lad_protected_settings="{'storageAccountName': '$my_diagnostic_storage_account', 'storageAccountSasToken': '$my_diagnostic_storage_account_sastoken'}"

# Finally tell Azure to install and enable the extension
az vm extension set --publisher Microsoft.Azure.Diagnostics --name LinuxDiagnostic --version 4.0 --resource-group $my_resource_group --vm-name $my_linux_vm --protected-settings "${my_lad_protected_settings}" --settings portal_public_settings.json
```
#### Azure CLI sample for Installing LAD 4.0 extension on the virtual machine scale set instance

```azurecli
#Set your Azure VMSS diagnostic variables correctly below
$my_resource_group=<your_azure_resource_group_name_containing_your_azure_linux_vm>
$my_linux_vmss=<your_azure_linux_vmss_name>
$my_diagnostic_storage_account=<your_azure_storage_account_for_storing_vm_diagnostic_data>

# Should login to Azure first before anything else
az login

# Select the subscription containing the storage account
az account set --subscription <your_azure_subscription_id>

# Enable System Assigned Identity to the existing VMSS
az vmss identity assign -g $my_resource_group -n $my_linux_vmss

# Download the sample Public settings. (You could also use curl or any web browser)
wget https://raw.githubusercontent.com/Azure/azure-linux-extensions/master/Diagnostic/tests/lad_2_3_compatible_portal_pub_settings.json -O portal_public_settings.json

# Build the VMSS resource ID. Replace storage account name and resource ID in the public settings.
$my_vmss_resource_id=$(az vmss show -g $my_resource_group -n $my_linux_vmss --query "id" -o tsv)
sed -i "s#__DIAGNOSTIC_STORAGE_ACCOUNT__#$my_diagnostic_storage_account#g" portal_public_settings.json
sed -i "s#__VM_RESOURCE_ID__#$my_vmss_resource_id#g" portal_public_settings.json

# Build the protected settings (storage account SAS token)
$my_diagnostic_storage_account_sastoken=$(az storage account generate-sas --account-name $my_diagnostic_storage_account --expiry 2037-12-31T23:59:00Z --permissions wlacu --resource-types co --services bt -o tsv)
$my_lad_protected_settings="{'storageAccountName': '$my_diagnostic_storage_account', 'storageAccountSasToken': '$my_diagnostic_storage_account_sastoken'}"

# Finally tell Azure to install and enable the extension
az vmss extension set --publisher Microsoft.Azure.Diagnostics --name LinuxDiagnostic --version 4.0 --resource-group $my_resource_group --vmss-name $my_linux_vmss --protected-settings "${my_lad_protected_settings}" --settings portal_public_settings.json
```

#### PowerShell sample

```powershell
$storageAccountName = "yourStorageAccountName"
$storageAccountResourceGroup = "yourStorageAccountResourceGroupName"
$vmName = "yourVMName"
$VMresourceGroup = "yourVMResourceGroupName"

# Get the VM object
$vm = Get-AzVM -Name $vmName -ResourceGroupName $VMresourceGroup

# Enable System Assigned Identity on an existing VM
Update-AzVM -ResourceGroupName $VMresourceGroup -VM $vm -IdentityType SystemAssigned

# Get the public settings template from GitHub and update the templated values for storage account and resource ID
$publicSettings = (Invoke-WebRequest -Uri https://raw.githubusercontent.com/Azure/azure-linux-extensions/master/Diagnostic/tests/lad_2_3_compatible_portal_pub_settings.json).Content
$publicSettings = $publicSettings.Replace('__DIAGNOSTIC_STORAGE_ACCOUNT__', $storageAccountName)
$publicSettings = $publicSettings.Replace('__VM_RESOURCE_ID__', $vm.Id)

# If you have your own customized public settings, you can inline those rather than using the template above: $publicSettings = '{"ladCfg":  { ... },}'

# Generate a SAS token for the agent to use to authenticate with the storage account
$sasToken = New-AzStorageAccountSASToken -Service Blob,Table -ResourceType Service,Container,Object -Permission "racwdlup" -Context (Get-AzStorageAccount -ResourceGroupName $storageAccountResourceGroup -AccountName $storageAccountName).Context -ExpiryTime $([System.DateTime]::Now.AddYears(10))

# Build the protected settings (storage account SAS token)
$protectedSettings="{'storageAccountName': '$storageAccountName', 'storageAccountSasToken': '$sasToken'}"

# Finally install the extension with the settings built above
Set-AzVMExtension -ResourceGroupName $VMresourceGroup -VMName $vmName -Location $vm.Location -ExtensionType LinuxDiagnostic -Publisher Microsoft.Azure.Diagnostics -Name LinuxDiagnostic -SettingString $publicSettings -ProtectedSettingString $protectedSettings -TypeHandlerVersion 4.0 
```

### Updating the extension settings

After you've changed your Protected or Public settings, deploy them to the VM by running the same command. If anything changed in the settings, the updated settings are sent to the extension. LAD reloads the configuration and restarts itself.

### Migration from previous versions of the extension

The latest version of the extension is **4.0 which is currently in Public Preview**. **Older versions of 3.x are still being supported while versions of 2.x are deprecated since July 31, 2018**.

> [!IMPORTANT]
> To migrate from 3.x to this new version of the extension, you must uninstall the old extension, then install version 4 of the extension (with the updated configuration for system assigned identity and sinks for sending metrics to Azure Monitor Sink.)

Recommendations:

* Install the extension with automatic minor version upgrade enabled.
  * On classic deployment model VMs, specify '4.*' as the version if you are installing the extension through Azure XPLAT CLI or PowerShell.
  * On Azure Resource Manager deployment model VMs, include '"autoUpgradeMinorVersion": true' in the VM deployment template.
* Can use the same storage account for LAD 4.0 as with LAD 3.*. 

## Protected settings

This set of configuration information contains sensitive information that should be protected from public view, for example, storage credentials. These settings are transmitted to and stored by the extension in encrypted form.

```json
{
    "storageAccountName" : "the storage account to receive data",
    "storageAccountEndPoint": "the hostname suffix for the cloud for this account",
    "storageAccountSasToken": "SAS access token",
    "mdsdHttpProxy": "HTTP proxy settings",
    "sinksConfig": { ... }
}
```

Name | Value
---- | -----
storageAccountName | The name of the storage account in which data is written by the extension.
storageAccountEndPoint | (optional) The endpoint identifying the cloud in which the storage account exists. If this setting is absent, LAD defaults to the Azure public cloud, `https://core.windows.net`. To use a storage account in Azure Germany, Azure Government, or Azure China, set this value accordingly.
storageAccountSasToken | An [Account SAS token](https://azure.microsoft.com/blog/sas-update-account-sas-now-supports-all-storage-services/) for Blob and Table services (`ss='bt'`), applicable to containers and objects (`srt='co'`), which grants add, create, list, update, and write permissions (`sp='acluw'`). Do *not* include the leading question-mark (?).
mdsdHttpProxy | (optional) HTTP proxy information needed to enable the extension to connect to the specified storage account and endpoint.
sinksConfig | (optional) Details of alternative destinations to which metrics and events can be delivered. The specific details of each data sink supported by the extension are covered in the sections that follow.

To get a SAS token within a Resource Manager template, use the **listAccountSas** function. For an example template, see [List function example](../../azure-resource-manager/templates/template-functions-resource.md#list-example).

You can easily construct the required SAS token through the Azure portal.

1. Select the general-purpose storage account to which you want the extension to write
1. Select "Shared access signature" from the Settings part of the left menu
1. Make the appropriate sections as previously described
1. Click the "Generate SAS" button.

:::image type="content" source="./media/diagnostics-linux/make_sas.png" alt-text="Screenshot shows the Shared access signature page with Generate S A S.":::

Copy the generated SAS into the storageAccountSasToken field; remove the leading question-mark ("?").

### sinksConfig

```json
"sinksConfig": {
    "sink": [
        {
            "name": "sinkname",
            "type": "sinktype",
            ...
        },
        ...
    ]
},
```

This optional section defines additional destinations to which the extension sends the information it collects. The "sink" array contains an object for each additional data sink. The "type" attribute determines the other attributes in the object.

Element | Value
------- | -----
name | A string used to refer to this sink elsewhere in the extension configuration.
type | The type of sink being defined. Determines the other values (if any) in instances of this type.

Version 4.0 of the Linux Diagnostic Extension supports two sink types: EventHub, and JsonBlob.

#### The EventHub sink

```json
"sink": [
    {
        "name": "sinkname",
        "type": "EventHub",
        "sasURL": "https SAS URL"
    },
    ...
]
```

The "sasURL" entry contains the full URL, including SAS token, for the Event Hub to which data should be published. LAD requires a SAS naming a policy that enables the Send claim. An example:

* Create an Event Hubs namespace called `contosohub`
* Create an Event Hub in the namespace called `syslogmsgs`
* Create a Shared access policy on the Event Hub named `writer` that enables the Send claim

If you created a SAS good until midnight UTC on January 1, 2018, the sasURL value might be:

```https
https://contosohub.servicebus.windows.net/syslogmsgs?sr=contosohub.servicebus.windows.net%2fsyslogmsgs&sig=xxxxxxxxxxxxxxxxxxxxxxxxx&se=1514764800&skn=writer
```

For more information about generating and retrieving information on SAS tokens for Event Hubs, see [this web page](/rest/api/eventhub/generate-sas-token#powershell).

#### The JsonBlob sink

```json
"sink": [
    {
        "name": "sinkname",
        "type": "JsonBlob"
    },
    ...
]
```

Data directed to a JsonBlob sink is stored in blobs in Azure storage. Each instance of LAD creates a blob every hour for each sink name. Each blob always contains a syntactically valid JSON array of object. New entries are atomically added to the array. Blobs are stored in a container with the same name as the sink. The Azure storage rules for blob container names apply to the names of JsonBlob sinks: between 3 and 63 lower-case alphanumeric ASCII characters or dashes.

## Public settings

This structure contains various blocks of settings that control the information collected by the extension. Each setting (except ladCfg) is optional. If you specify metric or syslog collection in `ladCfg`, you must also specify `StorageAccount`. sinksConfig element needs to be specified in order to enable Azure Monitor Sink for metrics from LAD 4.0

```json
{
    "ladCfg":  { ... },
    "fileLogs": { ... },
    "StorageAccount": "the storage account to receive data",
    "sinksConfig": { ... },
    "mdsdHttpProxy" : ""
}
```

Element | Value
------- | -----
StorageAccount | The name of the storage account in which data is written by the extension. Must be the same name as is specified in the [Protected settings](#protected-settings).
mdsdHttpProxy | (optional) Same as in the [Protected settings](#protected-settings). The public value is overridden by the private value, if set. Place proxy settings that contain a secret, such as a password, in the [Protected settings](#protected-settings).

The remaining elements are described in detail in the following sections.

### ladCfg

```json
"ladCfg": {
    "diagnosticMonitorConfiguration": {
        "eventVolume": "Medium",
        "metrics": { ... },
        "performanceCounters": { ... },
        "syslogEvents": { ... }
    },
    "sampleRateInSeconds": 15
}
```

This structure controls the gathering of metrics and logs for delivery to the Azure Metrics service and to other data sinks. You must specify either `performanceCounters` or `syslogEvents` or both. You must specify the `metrics` structure.

If you don't want to enable syslog or metrics collection, then you can simply specify an empty structure for ladCfg element as shown below - 

```json
"ladCfg": {
    "diagnosticMonitorConfiguration": {}
    }
```

Element | Value
------- | -----
eventVolume | (optional) Controls the number of partitions created within the storage table. Must be one of `"Large"`, `"Medium"`, or `"Small"`. If not specified, the default value is `"Medium"`.
sampleRateInSeconds | (optional) The default interval between collection of raw (unaggregated) metrics. The smallest supported sample rate is 15 seconds. If not specified, the default value is `15`.

#### metrics

```json
"metrics": {
    "resourceId": "/subscriptions/...",
    "metricAggregation" : [
        { "scheduledTransferPeriod" : "PT1H" },
        { "scheduledTransferPeriod" : "PT5M" }
    ]
}
```

Element | Value
------- | -----
resourceId | The Azure Resource Manager resource ID of the VM or of the virtual machine scale set to which the VM belongs. This setting must be also specified if any JsonBlob sink is used in the configuration.
scheduledTransferPeriod | The frequency at which aggregate metrics are to be computed and transferred to Azure Metrics, expressed as an IS 8601 time interval. The smallest transfer period is 60 seconds, that is, PT1M. You must specify at least one scheduledTransferPeriod.

Samples of the metrics specified in the performanceCounters section are collected every 15 seconds or at the sample rate explicitly defined for the counter. If multiple scheduledTransferPeriod frequencies appear (as in the example), each aggregation is computed independently.

#### performanceCounters

```json
"performanceCounters": {
    "sinks": "",
    "performanceCounterConfiguration": [
        {
            "type": "builtin",
            "class": "Processor",
            "counter": "PercentIdleTime",
            "counterSpecifier": "/builtin/Processor/PercentIdleTime",
            "condition": "IsAggregate=TRUE",
            "sampleRate": "PT15S",
            "unit": "Percent",
            "annotation": [
                {
                    "displayName" : "Aggregate CPU %idle time",
                    "locale" : "en-us"
                }
            ]
        }
    ]
}
```

This optional section controls the collection of metrics. Raw samples are aggregated for each [scheduledTransferPeriod](#metrics) to produce these values:

* mean
* minimum
* maximum
* last-collected value
* count of raw samples used to compute the aggregate

Element | Value
------- | -----
sinks | (optional) A comma-separated list of names of sinks to which LAD sends aggregated metric results. All aggregated metrics are published to each listed sink. See [sinksConfig](#sinksconfig). Example: `"EHsink1, myjsonsink"`.
type | Identifies the actual provider of the metric.
class | Together with "counter", identifies the specific metric within the provider's namespace.
counter | Together with "class", identifies the specific metric within the provider's namespace.
counterSpecifier | Identifies the specific metric within the Azure Metrics namespace.
condition | (optional) Selects a specific instance of the object to which the metric applies or selects the aggregation across all instances of that object. For more information, see the `builtin` metric definitions.
sampleRate | IS 8601 interval that sets the rate at which raw samples for this metric are collected. If not set, the collection interval is set by the value of [sampleRateInSeconds](#ladcfg). The shortest supported sample rate is 15 seconds (PT15S).
unit | Should be one of these strings: "Count", "Bytes", "Seconds", "Percent", "CountPerSecond", "BytesPerSecond", "Millisecond". Defines the unit for the metric. Consumers of the collected data expect the collected data values to match this unit. LAD ignores this field.
displayName | The label (in the language specified by the associated locale setting) to be attached to this data in Azure Metrics. LAD ignores this field.

The counterSpecifier is an arbitrary identifier. Consumers of metrics, like the Azure portal charting and alerting feature, use counterSpecifier as the "key" that identifies a metric or an instance of a metric. For `builtin` metrics, we recommend you use counterSpecifier values that begin with `/builtin/`. If you are collecting a specific instance of a metric, we recommend you attach the identifier of the instance to the counterSpecifier value. Some examples:

* `/builtin/Processor/PercentIdleTime` - Idle time averaged across all vCPUs
* `/builtin/Disk/FreeSpace(/mnt)` - Free space for the /mnt filesystem
* `/builtin/Disk/FreeSpace` - Free space averaged across all mounted filesystems

Neither LAD nor the Azure portal expects the counterSpecifier value to match any pattern. Be consistent in how you construct counterSpecifier values.

When you specify `performanceCounters`, LAD always writes data to a table in Azure storage. You can have the same data written to JSON blobs and/or Event Hubs, but you cannot disable storing data to a table. All instances of the diagnostic extension configured to use the same storage account name and endpoint add their metrics and logs to the same table. If too many VMs are writing to the same table partition, Azure can throttle writes to that partition. The eventVolume setting causes entries to be spread across 1 (Small), 10 (Medium), or 100 (Large) different partitions. Usually, "Medium" is sufficient to ensure traffic is not throttled. The Azure Metrics feature of the Azure portal uses the data in this table to produce graphs or to trigger alerts. The table name is the concatenation of these strings:

* `WADMetrics`
* The "scheduledTransferPeriod" for the aggregated values stored in the table
* `P10DV2S`
* A date, in the form "YYYYMMDD", which changes every 10 days

Examples include `WADMetricsPT1HP10DV2S20170410` and `WADMetricsPT1MP10DV2S20170609`.

#### syslogEvents

```json
"syslogEvents": {
    "sinks": "",
    "syslogEventConfiguration": {
        "facilityName1": "minSeverity",
        "facilityName2": "minSeverity",
        ...
    }
}
```

This optional section controls the collection of log events from syslog. If the section is omitted, syslog events are not captured at all.

The syslogEventConfiguration collection has one entry for each syslog facility of interest. If minSeverity is "NONE" for a particular facility, or if that facility does not appear in the element at all, no events from that facility are captured.

Element | Value
------- | -----
sinks | A comma-separated list of names of sinks to which individual log events are published. All log events matching the restrictions in syslogEventConfiguration are published to each listed sink. Example: "EHforsyslog"
facilityName | A syslog facility name (such as "LOG\_USER" or "LOG\_LOCAL0"). See the "facility" section of the [syslog man page](http://man7.org/linux/man-pages/man3/syslog.3.html) for the full list.
minSeverity | A syslog severity level (such as "LOG\_ERR" or "LOG\_INFO"). See the "level" section of the [syslog man page](http://man7.org/linux/man-pages/man3/syslog.3.html) for the full list. The extension captures events sent to the facility at or above the specified level.

When you specify `syslogEvents`, LAD always writes data to a table in Azure storage. You can have the same data written to JSON blobs and/or Event Hubs, but you cannot disable storing data to a table. The partitioning behavior for this table is the same as described for `performanceCounters`. The table name is the concatenation of these strings:

* `LinuxSyslog`
* A date, in the form "YYYYMMDD", which changes every 10 days

Examples include `LinuxSyslog20170410` and `LinuxSyslog20170609`.

### sinksConfig

This optional section controls enabling sending metrics to the Azure Monitor Sink in addition to the Storage account and the default Guest Metrics blade.

> [!NOTE]
> This requires System Assigned Identity to be enabled on the VMs/VMSS. 
> This can be done through portal, CLI, PowerShell, and resource manager. Steps are listed in detail [here](../../active-directory/managed-identities-azure-resources/qs-configure-portal-windows-vm.md). 
> The steps to enable this are also listed in the installation samples for AZ CLI, PowerShell etc. above. 

```json
  "sinksConfig": {
    "sink": [
      {
        "name": "AzMonSink",
        "type": "AzMonSink",
        "AzureMonitor": {}
      }
    ]
  },
```


### fileLogs

Controls the capture of log files. LAD captures new text lines as they are written to the file and writes them to table rows and/or any specified sinks (JsonBlob or EventHub).

> [!NOTE]
> fileLogs are captured by a subcomponent of LAD called `omsagent`. In order to collect fileLogs, you must ensure that the `omsagent` user has read permissions on the files you specify, as well as execute permissions on all directories in the path to that file. You can check this by running `sudo su omsagent -c 'cat /path/to/file'` after LAD is installed.

```json
"fileLogs": [
    {
        "file": "/var/log/mydaemonlog",
        "table": "MyDaemonEvents",
        "sinks": ""
    }
]
```

Element | Value
------- | -----
file | The full pathname of the log file to be watched and captured. The pathname must name a single file; it cannot name a directory or contain wildcards. The 'omsagent' user account must have read access to the file path.
table | (optional) The Azure storage table, in the designated storage account (as specified in the protected configuration), into which new lines from the "tail" of the file are written.
sinks | (optional) A comma-separated list of names of additional sinks to which log lines sent.

Either "table" or "sinks", or both, must be specified.

## Metrics supported by the builtin provider

> [!NOTE]
> The default metrics supported by LAD are aggregated across all file-systems/disks/name. For non-aggregated metrics, kindly refer to the newer Azure Monitor Sink metrics support.

The builtin metric provider is a source of metrics most interesting to a broad set of users. These metrics fall into five broad classes:

* Processor
* Memory
* Network
* Filesystem
* Disk

### builtin metrics for the Processor class

The Processor class of metrics provides information about processor usage in the VM. When aggregating percentages, the result is the average across all CPUs. In a two-vCPU VM, if one vCPU was 100% busy and the other was 100% idle, the reported PercentIdleTime would be 50. If each vCPU was 50% busy for the same period, the reported result would also be 50. In a four-vCPU VM, with one vCPU 100% busy and the others idle, the reported PercentIdleTime would be 75.

counter | Meaning
------- | -------
PercentIdleTime | Percentage of time during the aggregation window that processors were executing the kernel idle loop
PercentProcessorTime | Percentage of time executing a non-idle thread
PercentIOWaitTime | Percentage of time waiting for IO operations to complete
PercentInterruptTime | Percentage of time executing hardware/software interrupts and DPCs (deferred procedure calls)
PercentUserTime | Of non-idle time during the aggregation window, the percentage of time spent in user more at normal priority
PercentNiceTime | Of non-idle time, the percentage spent at lowered (nice) priority
PercentPrivilegedTime | Of non-idle time, the percentage spent in privileged (kernel) mode

The first four counters should sum to 100%. The last three counters also sum to 100%; they subdivide the sum of PercentProcessorTime, PercentIOWaitTime, and PercentInterruptTime.

### builtin metrics for the Memory class

The Memory class of metrics provides information about memory utilization, paging, and swapping.

counter | Meaning
------- | -------
AvailableMemory | Available physical memory in MiB
PercentAvailableMemory | Available physical memory as a percent of total memory
UsedMemory | In-use physical memory (MiB)
PercentUsedMemory | In-use physical memory as a percent of total memory
PagesPerSec | Total paging (read/write)
PagesReadPerSec | Pages read from backing store (swap file, program file, mapped file, etc.)
PagesWrittenPerSec | Pages written to backing store (swap file, mapped file, etc.)
AvailableSwap | Unused swap space (MiB)
PercentAvailableSwap | Unused swap space as a percentage of total swap
UsedSwap | In-use swap space (MiB)
PercentUsedSwap | In-use swap space as a percentage of total swap

This class of metrics has only a single instance. The "condition" attribute has no useful settings and should be omitted.

### builtin metrics for the Network class

The Network class of metrics provides information about network activity on an individual network interface since boot. LAD does not expose bandwidth metrics, which can be retrieved from host metrics.

counter | Meaning
------- | -------
BytesTransmitted | Total bytes sent since boot
BytesReceived | Total bytes received since boot
BytesTotal | Total bytes sent or received since boot
PacketsTransmitted | Total packets sent since boot
PacketsReceived | Total packets received since boot
TotalRxErrors | Number of receive errors since boot
TotalTxErrors | Number of transmit errors since boot
TotalCollisions | Number of collisions reported by the network ports since boot

### builtin metrics for the Filesystem class

The Filesystem class of metrics provides information about filesystem usage. Absolute and percentage values are reported as they'd be displayed to an ordinary user (not root).

counter | Meaning
------- | -------
FreeSpace | Available disk space in bytes
UsedSpace | Used disk space in bytes
PercentFreeSpace | Percentage free space
PercentUsedSpace | Percentage used space
PercentFreeInodes | Percentage of unused inodes
PercentUsedInodes | Percentage of allocated (in use) inodes summed across all filesystems
BytesReadPerSecond | Bytes read per second
BytesWrittenPerSecond | Bytes written per second
BytesPerSecond | Bytes read or written per second
ReadsPerSecond | Read operations per second
WritesPerSecond | Write operations per second
TransfersPerSecond | Read or write operations per second

### builtin metrics for the Disk class

The Disk class of metrics provides information about disk device usage. These statistics apply to the entire drive. If there are multiple file systems on a device, the counters for that device are, effectively, aggregated across all of them.

counter | Meaning
------- | -------
ReadsPerSecond | Read operations per second
WritesPerSecond | Write operations per second
TransfersPerSecond | Total operations per second
AverageReadTime | Average seconds per read operation
AverageWriteTime | Average seconds per write operation
AverageTransferTime | Average seconds per operation
AverageDiskQueueLength | Average number of queued disk operations
ReadBytesPerSecond | Number of bytes read per second
WriteBytesPerSecond | Number of bytes written per second
BytesPerSecond | Number of bytes read or written per second

## Installing and configuring LAD 4.0

### Azure CLI

Assuming your protected settings are in the file ProtectedSettings.json and your public configuration information is in PublicSettings.json, run this command:

```azurecli
az vm extension set --publisher Microsoft.Azure.Diagnostics --name LinuxDiagnostic --version 4.0 --resource-group <resource_group_name> --vm-name <vm_name> --protected-settings ProtectedSettings.json --settings PublicSettings.json
```

The command assumes you are using the Azure Resource Management mode of the Azure CLI. To configure LAD for classic deployment model (ASM) VMs, switch to "asm" mode (`azure config mode asm`) and omit the resource group name in the command. For more information, see the [cross-platform CLI documentation](/cli/azure/authenticate-azure-cli).

### PowerShell

Assuming your protected settings are in the `$protectedSettings` variable and your public configuration information is in the `$publicSettings` variable, run this command:

```powershell
Set-AzVMExtension -ResourceGroupName <resource_group_name> -VMName <vm_name> -Location <vm_location> -ExtensionType LinuxDiagnostic -Publisher Microsoft.Azure.Diagnostics -Name LinuxDiagnostic -SettingString $publicSettings -ProtectedSettingString $protectedSettings -TypeHandlerVersion 4.0
```

## An example LAD 4.0 configuration

Based on the preceding definitions, here's a sample LAD 4.0 extension configuration with some explanation. To apply this sample to your case, you should use your own storage account name, account SAS token, and EventHubs SAS tokens.

> [!NOTE]
> Depending on whether you use the Azure CLI or PowerShell to install LAD, the method for providing public and protected settings will differ. If using the Azure CLI, save the following settings to ProtectedSettings.json and PublicSettings.json to use with the sample command above. If using PowerShell, save the settings to `$protectedSettings` and `$publicSettings` by running `$protectedSettings = '{ ... }'`.

### Protected settings

These protected settings configure:

* a storage account
* a matching account SAS token
* several sinks (JsonBlob or EventHubs with SAS tokens)

```json
{
  "storageAccountName": "yourdiagstgacct",
  "storageAccountSasToken": "sv=xxxx-xx-xx&ss=bt&srt=co&sp=wlacu&st=yyyy-yy-yyT21%3A22%3A00Z&se=zzzz-zz-zzT21%3A22%3A00Z&sig=fake_signature",
  "sinksConfig": {
    "sink": [
      {
        "name": "SyslogJsonBlob",
        "type": "JsonBlob"
      },
      {
        "name": "FilelogJsonBlob",
        "type": "JsonBlob"
      },
      {
        "name": "LinuxCpuJsonBlob",
        "type": "JsonBlob"
      },
      {
        "name": "MyJsonMetricsBlob",
        "type": "JsonBlob"
      },
      {
        "name": "LinuxCpuEventHub",
        "type": "EventHub",
        "sasURL": "https://youreventhubnamespace.servicebus.windows.net/youreventhubpublisher?sr=https%3a%2f%2fyoureventhubnamespace.servicebus.windows.net%2fyoureventhubpublisher%2f&sig=fake_signature&se=1808096361&skn=yourehpolicy"
      },
      {
        "name": "MyMetricEventHub",
        "type": "EventHub",
        "sasURL": "https://youreventhubnamespace.servicebus.windows.net/youreventhubpublisher?sr=https%3a%2f%2fyoureventhubnamespace.servicebus.windows.net%2fyoureventhubpublisher%2f&sig=yourehpolicy&skn=yourehpolicy"
      },
      {
        "name": "LoggingEventHub",
        "type": "EventHub",
        "sasURL": "https://youreventhubnamespace.servicebus.windows.net/youreventhubpublisher?sr=https%3a%2f%2fyoureventhubnamespace.servicebus.windows.net%2fyoureventhubpublisher%2f&sig=yourehpolicy&se=1808096361&skn=yourehpolicy"
      }
    ]
  }
}
```

### Public Settings

These public settings cause LAD to:

* Upload percent-processor-time and used-disk-space metrics to the `WADMetrics*` table
* Upload messages from syslog facility "user" and severity "info" to the `LinuxSyslog*` table
* Upload appended lines in file `/var/log/myladtestlog` to the `MyLadTestLog` table

In each case, data is also uploaded to:

* Azure Blob storage (container name is as defined in the JsonBlob sink)
* EventHubs endpoint (as specified in the EventHubs sink)

```json
{
  "StorageAccount": "yourdiagstgacct",
  "ladCfg": {
    "sampleRateInSeconds": 15,
    "diagnosticMonitorConfiguration": {
      "performanceCounters": {
        "sinks": "MyMetricEventHub,MyJsonMetricsBlob",
        "performanceCounterConfiguration": [
          {
            "unit": "Percent",
            "type": "builtin",
            "counter": "PercentProcessorTime",
            "counterSpecifier": "/builtin/Processor/PercentProcessorTime",
            "annotation": [
              {
                "locale": "en-us",
                "displayName": "Aggregate CPU %utilization"
              }
            ],
            "condition": "IsAggregate=TRUE",
            "class": "Processor"
          },
          {
            "unit": "Bytes",
            "type": "builtin",
            "counter": "UsedSpace",
            "counterSpecifier": "/builtin/FileSystem/UsedSpace",
            "annotation": [
              {
                "locale": "en-us",
                "displayName": "Used disk space on /"
              }
            ],
            "condition": "Name=\"/\"",
            "class": "Filesystem"
          }
        ]
      },
      "metrics": {
        "metricAggregation": [
          {
            "scheduledTransferPeriod": "PT1H"
          },
          {
            "scheduledTransferPeriod": "PT1M"
          }
        ],
        "resourceId": "/subscriptions/your_azure_subscription_id/resourceGroups/your_resource_group_name/providers/Microsoft.Compute/virtualMachines/your_vm_name"
      },
      "eventVolume": "Large",
      "syslogEvents": {
        "sinks": "SyslogJsonBlob,LoggingEventHub",
        "syslogEventConfiguration": {
          "LOG_USER": "LOG_INFO"
        }
      }
    }
  },
  "sinksConfig": {
    "sink": [
      {
        "name": "AzMonSink",
        "type": "AzMonSink",
        "AzureMonitor": {}
      }
    ]
  },
  "fileLogs": [
    {
      "file": "/var/log/myladtestlog",
      "table": "MyLadTestLog",
      "sinks": "FilelogJsonBlob,LoggingEventHub"
    }
  ]
}
```

The `resourceId` in the configuration must match that of the VM or the virtual machine scale set.

* Azure platform metrics charting and alerting knows the resourceId of the VM you're working on. It expects to find the data for your VM using the resourceId the lookup key.
* If you use Azure autoscale, the resourceId in the autoscale configuration must match the resourceId used by LAD.
* The resourceId is built into the names of JsonBlobs written by LAD.

## View your data

Use the Azure portal to view performance data or set alerts:

:::image type="content" source="./media/diagnostics-linux/graph_metrics.png" alt-text="Screenshot shows the Azure portal with the Used disk space on metric selected and the resulting chart.":::

The `performanceCounters` data are always stored in an Azure Storage table. Azure Storage APIs are available for many languages and platforms.

Data sent to JsonBlob sinks is stored in blobs in the storage account named in the [Protected settings](#protected-settings). You can consume the blob data using any Azure Blob Storage APIs.

In addition, you can use these UI tools to access the data in Azure Storage:

* Visual Studio Server Explorer.
* [Screenshot shows containers and tables in Azure Storage Explorer.](https://azurestorageexplorer.codeplex.com/ "Azure Storage Explorer").

This snapshot of a Microsoft Azure Storage Explorer session shows the generated Azure Storage tables and containers from a correctly configured LAD 3.0 extension on a test VM. The image doesn't match exactly with the [sample LAD 3.0 configuration](#an-example-lad-40-configuration).

:::image type="content" source="./media/diagnostics-linux/stg_explorer.png" alt-text="Screenshot shows the Azure Storage Explorer.":::

See the relevant [EventHubs documentation](../../event-hubs/event-hubs-about.md) to learn how to consume messages published to an EventHubs endpoint.

## Next steps

* Create metric alerts in [Azure Monitor](../../azure-monitor/alerts/alerts-classic-portal.md) for the metrics you collect.
* Create [monitoring charts](../../azure-monitor/data-platform.md) for your metrics.
* Learn how to [create a virtual machine scale set](../linux/tutorial-create-vmss.md) using your metrics to control autoscaling.