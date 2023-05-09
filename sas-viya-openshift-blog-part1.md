# SAS Viya on Red Hat OpenShift  – Part 1: Reference Architecture and Deployment Considerations
**TARGET DATE:** _May 19, 2023 | by Patrick Farley and Hans-Joachim Edert_

**<div align="center">DRAFT COPY  -  DRAFT COPY  -  DRAFT COPY  -  DRAFT COPY  -  DRAFT COPY  -  DRAFT COPY  -  DRAFT COPY  -  DRAFT COPY</div>**   

**_<div align="center">This blog was written by Patrick Farley, Associate Principal Solutions Architect (Red Hat) </div>_**
**_<div align="center">and Hans-Joachim Edert, Advisory Business Solutions Manager (SAS Institute) </div>_**

In this two-part blog, we will provide some basic technical information about SAS Viya, as well as a reference architecture for SAS Viya on Red Hat OpenShift Container Platform (OCP). This reference architecture will show how SAS Viya is containerized to run with Kubernetes on Red Hat OpenShift.

A few introductory words before we get into the technical details.

Since the launch of SAS Viya in 2020, SAS has offered a fully containerized analytic platform based on a cloud-native architecture. Due to the scale of the platform, SAS Viya requires Kubernetes as an underlying runtime environment and takes full advantage of the native benefits of this technology. 

SAS supports numerous Kubernetes distributions, both in the public and private cloud. In fact, many SAS customers - partly due to specific use cases that do not allow otherwise from a regulatory perspective, but also due to strategic considerations - prefer to run their application infrastructure in a private cloud environment. 

In these cases, Red Hat OpenShift provides an optimal foundation for the SAS software stack. OpenShift offers both a hardened Kubernetes with many highly valued enterprise features as an execution platform, but also comes with an extensive ecosystem with a particular focus on supporting DevSecOps capabilities.

SAS and Red Hat have enjoyed a productive partnership for more than a decade - while Red Hat Enterprise Linux was the preferred operating system in earlier SAS releases, today SAS has based their container images on the Red Hat Universal Base Image.

Moreover, for Red Hat OpenShift deployments, SAS takes advantage of the OpenShift Ingress Operator, the cert-utils operator, OpenShift GitOps for deployment (optionally), and integrates into OpenShift’s security approach which is based on SCCs (Security Context Constraints) as part of their deployments.
<br></br>

## **SAS Viya on OpenShift Reference Architecture**
SAS Viya is an integrated platform that covers the entire AI and Analytics lifecycle. Thus, it is not just a single application, but a suite of integrated applications. One of the fundamental differences here is the nature of the workload that SAS Viya brings to the OpenShift platform. This affects the need for resources (CPU, memory, storage) and entails special security-specific requirements.

Moving SAS Viya to OpenShift gives Viya unprecedented scalability that was unavailable in previous SAS releases. SAS takes advantage of the scalability by breaking Viya down into different workload types and recommends assigning each workload to a class of nodes, i.e., to Machine Pools. This ensures that the proper resources are available to specific workloads. _Figure 1_ shows the separation of workloads to pools.

<img width="1089" alt="image" src="sas-viya-reference-architecture-ocp.png">

**_<div align="center">Figure 1</div>_**

Note that the setup of pools is not mandatory and there might be reasons to ignore the recommendation if the existing cluster infrastructure is not suitable for such a split. Applying a workload placement strategy by using node pools provides a lot of benefits, as it allows you to tailor the cluster topology to workload requirements; you could, for example, choose different hardware configurations (nodes with additional storage, with GPU cards etc.). The placement of SAS workload classes can be enabled by applying predefined Kubernetes node labels and node taints.

It might be helpful for a better understanding to briefly explain the workload classes mentioned in this diagram.

### **SAS Cloud Analytic Server (CAS) (CAS NODE POOL)**
The core component at the heart of SAS Viya is the Cloud Analytics Server (CAS). It is an in-memory analytics engine. Data is loaded from the required data source into some in-memory tables then clients connect to CAS to run analytical routines against the data. The data loaded into memory for CAS can also be flushed to disk. This is why the CAS server usually has the highest resource requirements of all SAS Viya components: it is CPU- *and* memory-intensive and requires persistent storage accessible by all nodes hosting CAS pods.

CAS can be deployed in one of two modes:  SMP (Symmetric Multi Processing), and MPP (Massive Parallel Processing). In SMP mode, only one CAS pod is being deployed, in MPP mode multiple CAS pods are used where one pod takes the role of a controller while the other pods are used for running the computations.

In a default configuration, i.e., when a CAS node pool is being used, each CAS pod runs on a separate worker node, claiming more than 90% of the available CPU and memory resources out-of-the-box. If  there is no node pool available for CAS, transformer patches applied during the deployment limit the resource usage of CAS to the desired amount ot allow co-existence of CAS with other workloads on the same node.

### **SAS Compute Services (COMPUTE NODE POOL)**
SAS Compute services represent the traditional SAS processing capabilities as used in all previous releases of SAS. A SAS session is launched either interactively by a user from a web application or in batch mode to run as a Kubernetes job to execute submitted SAS code to transform or analyse data. Due to this approach, SAS sessions are highly parallelizable. The number of sessions (or Kubernetes jobs) running in parallel is only limited by the available hardware resources.

The compute node pool is a good candidate for using the cluster autoscaler, if possible. Often customers have typical usage patterns that would directly benefit from this - for example, by intercepting usage peaks (scaling out for nightly batch workload, scaling in over the weekend, etc.).

### **SAS Microservices and Web Applications (STATELESS NODE POOL)**
Most services in any SAS Viya deployment are designed as microservices (also known as 12 factor apps). They are responsible providing central services like auditing, authentication, etc. Also, grouped with these services are a set of stateless web applications which are the user interfaces which are exposed to endusers, for example SAS Data Studio, SAS Model Manager and SAS Data Explorer.

### **Infrastructure Services (STATEFUL NODE POOL)**
The commodity services are basically the metadata management and storage services. They are made of several open-source technologies such as the internal SAS Postgres database, as well as Consul and RabbitMQ for messaging. This is where the critical operational data is stored. These services are rather I/O intensive and do require persistent storage.

<br></br>

## **Core Platform**
VMware vSphere 7.0 Update 1 or later is the virtual machine platform that is currently supported. The details of VMware configuration will not be covered in this document with the assumption that VMware vSphere is well known in most environments.

In the current iteration of SAS Viya on OpenShift, SAS only supports VMware vSphere as the deployment platform. At the time of this document, BareMetal is on the roadmap as an alternative for on-premise deployments.  When deployed on a different infrastructure provider, such as Azure, AWS, Google or BareMetal, SAS Viya runs under the “[SAS Support for Alternative Kubernetes Distributions](https://support.sas.com/en/technical-support/services-policies.html#k8s)” policy.

At the time this blog was written, Red Hat OpenShift versions 4.10 - 4.12 are supported for SAS Viya. SAS works to align their SAS Viya Kubernetes support levels with Red Hat OpenShift and typically adds support for the latest OpenShift version updates within 1-2 months of a given OpenShift version release. Additional details are provided later in Part 2 of this document; for some of the specific OpenShift components that support SAS Viya deployment; but they are provided here, at a high-level:

- **OpenShift Ingress Operator**

   SAS has specific requirements for forwarding cookies during transaction execution. As such, they used special techniques using the HAProxy to make that happen. So, in this iteration only the OpenShift Ingress Operator is supported. 
   
- **OpenShift Routes** 

   SAS prefers the use of native features with the environments with their products, so they take advantage of OpenShift routes.
   
- **cert-utils-operator**

   SAS requires the use of this operator to manage certificates for TLS support and create keystores.
   
- **cert-manager**

   SAS Viya supports two different certificate generators, which are used for enforcing full-stack TLS. The default generator uses OpenSSL and is supplied out-of-the-box by SAS. Alternatively, you can optionally deploy and use cert-manager to generate the certificates used to encrypt the pod-to-pod communication. 
   
- **Security Context Constraints**

   Security Context Constraints (SCCs) provide permissions to pods and are required to allow them to run. SAS requires several custom SCCs to support SAS Viya Services with OpenShift. The SAS documentation provides information about the required SCCs to help understand their use in your environment and to address any security concerns.  Further details about the required custom SCCs are provided in Part 2 of this document.
<br></br>

### **Deployment options – Red Hat OpenShift**
There are various methods for [installing Red Hat OpenShift Container Platform on VMware vSphere,](https://docs.openshift.com/container-platform/4.12/installing/installing_vsphere/preparing-to-install-on-vsphere.html)  including:

- _Installer-provisioned infrastructure_ (IPI) installation, which allows the installation program to pre-configure and automate the provisioning of the required resources.
- The [_Assisted Installer_,](https://docs.openshift.com/container-platform/4.12/installing/installing_on_prem_assisted/installing-on-prem-assisted.html) which <a name="_int_0dxvuolc"></a>provides a user-friendly installation solution offered directly from the [Red Hat Hybrid Cloud Console.](https://console.redhat.com/openshift/assisted-installer/clusters/~new)
- _User-provisioned infrastructure_ (UPI) installation, which provides a manual type of installation with the most control over the installation and configuration process.

An IPI installation results in an OCP cluster with the vSphere cloud provider configuration settings from the installation, which enables additional automation and dynamic provisioning after installation:

- [Persistent storage using the VMware CSI or in-tree driver operator](https://docs.openshift.com/container-platform/4.12/storage/container_storage_interface/persistent-storage-csi-vsphere.html).
- [Host Node management for Node pools using MachineSets](https://docs.openshift.com/container-platform/4.12/machine_management/index.html).
- [Autoscaling Nodes using the MachineAutoScaler and ClusterAutoScaler operators](https://docs.openshift.com/container-platform/4.12/machine_management/applying-autoscaling.html). 

Information about configuring these capabilities is supplied within Part 2 of this document.

An [OpenShift installation on VMware vSphere](https://docs.openshift.com/container-platform/4.12/installing/installing_vsphere/preparing-to-install-on-vsphere.html) using the UPI or Assisted Installer methods can also be set up [post-installation with the vSphere cloud provider configuration](https://access.redhat.com/documentation/en-us/assisted_installer_for_openshift_container_platform/2022/html-single/assisted_installer_for_openshift_container_platform/index#vsphere-post-installation-configuration_installing-on-vsphere), to provide similar automation and dynamic provisioning capability as an IPI installation. This is out of the scope of this blog.
<br></br>

### **Deployment options – SAS Viya**
There are several approaches for deploying SAS Viya on Red Hat OpenShift, which are described in the SAS Operations Guide:

- Manually by running `kubectl` commands
- Using the SAS Deployment Operator
- Using the `sas-orchestration` command line utility

#### ***1. Manual deployment***
After purchasing a SAS Viya license, customers receive a set of deployment templates (known as the “deployment assets” tarball) in YAML format which they need to modify to create the final deployment manifest (usually called “site.yaml”). SAS uses the `kustomize` tool for modifying the templates. Common customizations include the definition of a mirror repository, configuring TLS, high-availability, storage and other site-specific settings. The final deployment manifest can then be submitted to Kubernetes using multiple `kubectl` commands. 

_CLUSTER ADMIN:_ Note that the final manifest contains objects which require elevated privileges for deployment, for example Custom Resource Definitions (CRDs), `PodTemplates`, `ClusterRoleBindings` etc., which means that in most cases the SAS project team will need support from the OpenShift administration team to carry out the deployment. SAS has tagged all resources that need to be deployed according to the required permissions. This enables task sharing between the project team (with namespace-admin permissions) and the administration team (with cluster-admin permissions). However, it is important to keep in mind that this dependency will come up again with later updates for example.

#### ***2. SAS Deployment Operator***
For that reason, using the SAS Deployment Operator might provide a better solution. SAS provides an operator for deploying and updating SAS Viya. The SAS Deployment Operator is not (yet) a certified operator so it will not be found in the OperatorHub or in the Red Hat Marketplace.

The SAS Viya Deployment Operator provides an automated method for deploying and updating the SAS Viya environments. It runs in the OpenShift cluster and watches for declarative representations of SAS Viya deployments in the form of Custom Resources (CRs) of the type `SASDeployment`. When a new `SASDeployment` CR is created or an existing CR is updated, the Deployment Operator performs an initial deployment or updates an existing deployment to match the state that is described in the CR. A single instance of the operator can manage all SAS Viya deployments in the cluster.

_CLUSTER ADMIN:_ As part of a DevOps pipeline, the operator can largely automate deployments and deployment updates, reducing dependency on the OpenShift administration team. For example, the SAS Deployment Operator nicely integrates with OpenShift GitOps, which are is a component of the Red Hat OpenShift Container Platform (OCP) that provide a turnkey CI/CD automation solution for continuous integration (CI) and continuous delivery (CD) tasks. OpenShift GitOps can be used to provide additional automation for a SAS Viya deployment by monitoring a Git repository for changes to the SAS CR manifest and automatically syncing its’ contents to the cluster. Pushing the CR manifest to the Git repository then triggers a sync with OpenShift GitOps. The CR will be deployed to Kubernetes, which in turn triggers the Operator and the deployment to start. _Figure 2_ illustrates this workflow:

<img width="1089" alt="image" src="sas-viya-deployment-operator-ocp.png">

**_<div align="center">Figure 2</div>_**

For additional information, see the SAS blog titled “[Deploying SAS Viya using Red Hat OpenShift GitOps](https://communities.sas.com/t5/SAS-Communities-Library/Deploying-SAS-Viya-using-Red-Hat-OpenShift-GitOps/ta-p/780616)”

#### ***3. `sas-orchestration` command line utility***
The `sas-orchestration` utility offers the flexibility of both worlds: as a container image it can be launched manually on a Linux shell to create and submit the final deployment manifest (in other words:  it combines the `kustomize` and `kubectl` actions into one step) or it could be used as a step in a CI/CD pipeline, for example as a task in OpenShift Pipelines, Jenkins or Github Actions etc.

For more information, check the blog article titled “[New SAS Viya Deployment Methods](https://communities.sas.com/t5/SAS-Communities-Library/New-SAS-Viya-Deployment-Methods/ta-p/856206)”.

<br></br>
### **Conclusion**
… We hope you found this blog helpful … Stay tuned for the second installment where we will be discussing security and storage considerations.

<br></br>
<br></br>