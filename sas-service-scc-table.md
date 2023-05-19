The following table gives an overview of all cases where a custom SCC (or an SCC other than restricted) is used: 


|**SAS SERVICE** | **SERVICE ACCOUNT NAME / SCC NAME** |**REQUIRED or OPTIONAL**|**WHEN NEEDED**|
| :- | :- | :-: | :- | 
| **CAS SERVER** | `sas-cas-server`\*|**REQUIRED**|CAS server needs to access cloud native storage|
| |`sas-cas-server-host`|**Optional**|CAS server runs in host launch configuration|
| |`sas-cas-server-sssd`|**Optional**|CAS server needs additional sssd configuration|
| **SAS COMPUTE** |`sas-programming-environment` / `nonroot`** |**REQUIRED**|SAS compute sessions must be launched with specific user/group IDs|
|  |`sas-programming-environment` / `hostmount-anyuid`** |**Optional**|Only needed if SAS compute session needs to<br>access data on hostPath mounts|
|  |`sas-programming-environment` / `sas-watchdog`|**Optional**|LOCKDOWN mode needs to be enforced|
| **SAS/CONNECT** |`sas-connect-spawner`|**Optional**|Only needed with legacy SAS clients|
| **MAS** |`sas-microanalytic-score`|**Optional**|If NFS volume mounts are needed|
| **SAS MODEL MANAGEMENT** |`sas-model-publish-kaniko` / `anyuid`**|**Optional**|If analytical models are published into runtime container images|
|  |`sas-model-repository`|**Optional**|If NFS volume mounts are needed|
| **SAS EVENT STREAM PROCESSING** |`sas-esp-project` / `nonroot`**|**Optional**|If SAS Event Stream Processing is included in<br>your deployment|
| **OPENSEARCH** |`sas-opendistro`|**Optional** \*** |For optimal performance, deploying OpenSearch<br>software requires changes to a kernel setting.<br>Optionally, you can use a MachineConfig to apply the kernel parameter as documented below.|

