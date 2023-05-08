# SAS Viya on Red Hat OpenShift  – Part 1: Reference Architecture and Deployment considerations
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

## **SAS Viya on OpenShift Reference Architecture**
SAS Viya is an integrated platform that covers the entire AI and Analytics lifecycle. Thus, it is not just a single application, but a suite of integrated applications. One of the fundamental differences here is the nature of the workload that SAS Viya brings to the OpenShift platform. This affects the need for resources (CPU, memory, storage) and entails special security-specific requirements.

Moving SAS Viya to OpenShift gives Viya unprecedented scalability that was unavailable in <a name="_int_7dqcwsaq"></a>previous SAS releases. SAS takes advantage of the scalability by breaking Viya down into different workload types and recommends assigning each workload to a class of nodes, i.e., to a Machine Pool. This ensures that the proper resources are available to specific workloads. Figure 1 shows the separation of workloads to pools.


![](sas-viya-reference-architecture-ocp.png)

**_<div align="center">Figure 1</div>_**

Note that the setup of pools is not mandatory and there might be reasons to ignore the recommendation if the existing cluster infrastructure is not suitable for such a split. Applying a workload placement strategy by using node pools does provide a lot of benefits, as it allows you to tailor-suit the cluster topology to workload requirements, you could for example choose different hardware configurations (nodes with additional storage, with GPU cards etc.). The placement of SAS workload classes can be enabled by applying predefined Kubernetes node labels and node taints.

It might be helpful for a better understanding to briefly explain the workload classes mentioned in this diagram.

### **SAS Cloud Analytic Server (CAS) (CAS NODE POOL)**
The core component at the heart of SAS Viya is the Cloud Analytics Server (CAS). It is an in-memory analytics engine. Data is loaded from the required data source into some in-memory tables then clients connect to CAS to run analytical routines against the data. The data loaded into memory for CAS can also be flushed to disk. This is why the CAS server usually has the highest resource requirements of all SAS Viya components: it is CPU- *and* memory-intensive and requires persistent storage accessible by all nodes hosting CAS pods.

CAS can be deployed in one of two modes:  SMP (Symmetric Multi Processing), and MPP (Massive Parallel Processing). In SMP mode, only one CAS pod is being deployed, in MPP mode multiple CAS pods are used where one pod takes the role of a controller while the other pods are used for running the computations.

In a default configuration, i.e. when a CAS nodepool ist being used, each CAS pod runs on a separate worker node, claiming more than 90% of the available CPU and memory resources out-of-the-box. If  there is no nodepool available for CAS, transformer patches applied during the deployment limit the resource usage of CAS to the desired amount ot allow co-existence of CAS with other workloads on the same node.

### **SAS Compute Services (COMPUTE NODE POOL)**
SAS Compute services represent the traditional SAS processing capabilities as used in all previous releases of SAS. A SAS session is launched either interactively by a user from a web application or in batch mode to run as a Kubernetes job to execute submitted SAS code to transform or analyse data. Due to this approach, SAS sessions are highly parallelizable. The number of sessions (or Kubernetes jobs) running in parallel is only limited by the available hardware resources.

The compute nodepool is a good candidate for using the cluster autoscaler if possible. Often customers have typical usage patterns that would directly benefit from this - for example, by intercepting usage peaks (scaling out for nightly batch workload, scaling in over the weekend, etc.).

### **SAS Microservices and Web Applications (STATELESS NODE POOL)**
Most services in any SAS Viya deployment are designed as microservices (also known as 12 factor apps). They are responsible providing central services like auditing, authentication, etc. Also, grouped with these services are a set of stateless web applications which are the user interfaces which are exposed to endusers, for example SAS Data Studio, SAS Model Manager and SAS Data Explorer.

### **Infrastructure Services (STATEFUL NODE POOL)**
The commodity services are basically the metadata management and storage services. They are made of several open-source technologies such as the internal SAS Postgres database, as well as Consul and RabbitMQ for messaging. This is where the critical operational data is stored. These services are rather I/O intensive and do require persistent storage.

In the current iteration of SAS Viya on OpenShift, SAS only supports VMware vSphere as the deployment platform. At the time of this document, BareMetal is on the roadmap as an alternative for on-premise deployments.  When deployed on a different infrastructure provider, such as Azure, AWS, Google or BareMetal, SAS Viya runs under the “[SAS Support for Alternative Kubernetes Distributions](https://support.sas.com/en/technical-support/services-policies.html#k8s)” policy.

## **Core Platform**
VMware vSphere 7.0 Update 1 or later is the virtual machine platform that is currently supported. The details of VMware configuration will not be covered in this document with the assumption that VMware vSphere is well known in most environments.

At the time this blog was written, Red Hat OpenShift versions 4.10 - 4.12 are supported for SAS Viya. SAS works to align their SAS Viya Kubernetes support levels with Red Hat OpenShift and typically adds support for the latest OpenShift version updates within 1-2 months of a given OpenShift version release. Additional details are <a name="_int_owinntrs"></a>provided later within this document; for some of the specific OpenShift components that support SAS Viya deployment; but they are <a name="_int_jsjdqbki"></a>provided here, at a high-level:

- **OpenShift Ingress Operator**
  SAS has specific requirements for forwarding cookies during transaction execution. As such, they used special techniques using the HAProxy to make that happen. So, in this iteration only the OpenShift Ingress Operator is supported. 
- **OpenShift Routes** 
  SAS prefers the use of native features with the environments with their products, so they take advantage of OpenShift routes.
- **cert-utils-operator**
  SAS requires the use of this operator to manage certificates for TLS support and create keystores.
- **cert-manager**
  SAS Viya supports two different certificate generators, which are used for enforcing full-stack TLS. The default generator uses OpenSSL and is supplied out-of-the-box by SAS. Alternatively, you can optionally deploy and use cert-manager to generate the certificates used to encrypt the pod-to-pod communication. 
- **Security Context Constraints**
  Security Context Constraints (SCCs) provide permissions to pods and are required to allow them to run. SAS requires several custom SCCs to support SAS Viya Services with OpenShift. The SAS documentation provides information about the required SCCs to help understand their use in your environment and to address any security concerns.  Further details about the required custom SCCs are provided later within this document.

### **Deployment options – Red Hat OpenShift**
There are various methods for [installing Red Hat OpenShift Container Platform on VMware vSphere,](https://docs.openshift.com/container-platform/4.12/installing/installing_vsphere/preparing-to-install-on-vsphere.html)  including:

- *Installer-provisioned infrastructure* (IPI) installation, which allows the installation program to pre-configure and automate the provisioning of the required resources.
- The [*Assisted Installer*,](https://docs.openshift.com/container-platform/4.12/installing/installing_on_prem_assisted/installing-on-prem-assisted.html) which <a name="_int_0dxvuolc"></a>provides a user-friendly installation solution offered directly from the [Red Hat Hybrid Cloud Console.](https://console.redhat.com/openshift/assisted-installer/clusters/~new)
- *User-provisioned infrastructure* (UPI) installation, which provides a manual type of installation with the most control over the installation and configuration process.

An IPI installation results in an OCP cluster with the vSphere cloud provider configuration settings from the installation, which enables additional automation and dynamic provisioning after installation:

- [Persistent storage using the VMware CSI or in-tree driver operator](https://docs.openshift.com/container-platform/4.12/storage/container_storage_interface/persistent-storage-csi-vsphere.html).
- [Host Node management for Node pools using MachineSets](https://docs.openshift.com/container-platform/4.12/machine_management/index.html).
- [Autoscaling Nodes using the MachineAutoScaler and ClusterAutoScaler operators](https://docs.openshift.com/container-platform/4.12/machine_management/applying-autoscaling.html). 

Information about configuring these capabilities is supplied later within this document.

An [OpenShift installation on VMware vSphere](https://docs.openshift.com/container-platform/4.12/installing/installing_vsphere/preparing-to-install-on-vsphere.html) using the UPI or Assisted Installer methods can also be set up [post-installation with the vSphere cloud provider configuration](https://access.redhat.com/documentation/en-us/assisted_installer_for_openshift_container_platform/2022/html-single/assisted_installer_for_openshift_container_platform/index#vsphere-post-installation-configuration_installing-on-vsphere), to provide similar automation and dynamic provisioning capability as an IPI installation. This is out of the scope of this blog.

### **Deployment options – SAS Viya**
There are several approaches for deploying SAS Viya on Red Hat OpenShift, which are described in the SAS Operations Guide:

- Manually by running **kubectl** commands
- Using the SAS Deployment Operator
- Using the **sas-orchestration** command line utility

#### ***Manual deployment***
After purchasing a SAS Viya license, customers receive a set of deployment templates (known as the “deployment assets” tarball) in YAML format which they need to modify to create the final deployment manifest (usually called “site.yaml”). SAS uses the **kustomize** tool for modifying the templates. Common customizations include the definition of a mirror repository, configuring TLS, high-availability, storage and other site-specific settings. The final deployment manifest can then be submitted to Kubernetes using multiple **kubectl** commands. 

**CLUSTER ADMIN:** Note that the final manifest contains objects which require elevated privileges for deployment, for example Custom Resource Definitions (CRDs), *PodTemplates*, *ClusterRoleBindings* etc., which means that in most cases the SAS project team will need support from the OpenShift administration team to carry out the deployment. SAS has tagged all resources that need to be deployed according to the required permissions. This enables task sharing between the project team (with namespace-admin permissions) and the administration team (with cluster-admin permissions). However, it is important to keep in mind that this dependency will come up again with later updates for example.

#### ***SAS Deployment Operator***
For that reason, using the SAS Deployment Operator might provide a better solution. SAS provides an operator for deploying and updating SAS Viya. The SAS Deployment Operator is not (yet) a certified operator so it will not be found in the OperatorHub or in the Red Hat Marketplace.

The SAS Viya Deployment Operator provides an automated method for deploying and updating the SAS Viya environments. It runs in the OpenShift cluster and watches for declarative representations of SAS Viya deployments in the form of Custom Resources (CRs) of the type **SASDeployment**. When a new **SASDeployment** CR is created or an existing CR is updated, the Deployment Operator performs an initial deployment or updates an existing deployment to match the state that is described in the CR. A single instance of the operator can manage all SAS Viya deployments in the cluster.

**CLUSTER ADMIN:** As part of a DevOps pipeline, the operator can largely automate deployments and deployment updates, reducing dependency on the OpenShift administration team. For example, the SAS Deployment Operator nicely integrates with OpenShift GitOps, which are is a component of the Red Hat OpenShift Container Platform (OCP) that provide a turnkey CI/CD automation solution for continuous integration (CI) and continuous delivery (CD) tasks. OpenShift GitOps can be used to provide additional automation for a SAS Viya deployment by monitoring a Git repository for changes to the SAS CR manifest and automatically syncing its’ contents to the cluster. Pushing the CR manifest to the Git repository then triggers a sync with OpenShift GitOps. The CR will be deployed to Kubernetes, which in turn triggers the Operator and the deployment to start. Figure 2 illustrates this workflow:

![](sas-viya-deployment-operator-ocp.png)

**_<div align="center">Figure 2</div>_**

For additional information, see the SAS blog titled “[Deploying SAS Viya using Red Hat OpenShift GitOps](https://communities.sas.com/t5/SAS-Communities-Library/Deploying-SAS-Viya-using-Red-Hat-OpenShift-GitOps/ta-p/780616)”

#### ***sas-orchestration command line utility***
The **sas-orchestration** utility offers the flexibility of both worlds: as a container image it can be launched manually on a Linux shell to create and submit the final deployment manifest (in other words:  it combines the **kustomize** and **kubectl** actions into one step) or it could be used as a step in a CI/CD pipeline, for example as a task in OpenShift Pipelines, Jenkins or Github Actions etc.

For more information, check the blog article titled “[New SAS Viya Deployment Methods](https://communities.sas.com/t5/SAS-Communities-Library/New-SAS-Viya-Deployment-Methods/ta-p/856206)”.

### **Conclusion**
… We hope you found this blog helpful … Stay tuned for the second installment where we will be discussing security and storage considerations.


# SAS Viya on Red Hat OpenShift – Part 2: Security and storage considerations
**TARGET DATE:** _May 19, 2023 | by Patrick Farley and Hans-Joachim Edert_

**<div align="center">DRAFT COPY  -  DRAFT COPY  -  DRAFT COPY  -  DRAFT COPY  -  DRAFT COPY  -  DRAFT COPY  -  DRAFT COPY  -  DRAFT COPY</div>**   

**_<div align="center">This blog was written by Patrick Farley, Associate Principal Solutions Architect (Red Hat) </div>_**
**_<div align="center">and Hans-Joachim Edert, Advisory Business Solutions Manager (SAS Institute) </div>_**


## **~~SAS Viya Components~~**
~~High-level description here.~~
### **~~SAS Cloud Analytic Server (CAS) (CAS NODE POOL)~~**
~~The core <a name="_int_vm98npzb"></a>component of SAS Viya is the Cloud Analytics Server (CAS). CAS is the heart of the SAS Viya deployment. It is basically an in-memory analytics engine. Data is loaded from the required data source into some in-memory tables then clients connect to CAS to run analytical routines against the data. The data loaded into memory for CAS can also be flushed to disk. This is why SAS Viya <a name="_int_qsmmpuz1"></a>requires persistent storage accessible by all <a name="_int_j1zuuzgx"></a>compute nodes.~~ 

~~The best way to understand CAS is to understand the terminology and concepts used in the server and how it all fits together to achieve the goals mentioned above.~~
#### ***~~CAS Server Controller~~***
~~CAS consists of from 1 to N hosts. These hosts can be a part of a cluster in a node pool that carries specific resource requirements. One host is <a name="_int_b32zhzjt"></a>designated the "controller". The controller executes one process as the "server controller" and one process for each "server worker". The controller is the repository for the global state of the server. The client connects to the server controller using either the binary API port or the REST API port. Once the user has been authenticated, the server controller creates a session process on the controller and each worker that is to be included as part of this session. The client connection is then transferred to the session controller. At this point, the client can start <a name="_int_7ox2ottu"></a>submitting work to CAS.~~
#### ***~~CAS Server Worker~~***
~~The purpose of the worker nodes is to execute the same request as the other workers and controllers and process the data that is local to that worker. This allows the user or admin to spread data across the cluster to allow for local parallel processing of data that will come back together to produce a result. Just like the controller nodes, the worker nodes support one process as the server worker and one process for each session.~~
#### ***~~Two Different Execution Modes~~***
~~CAS can be deployed in one of two modes:  *SMP (Symmetric <a name="_int_yeakioye"></a>Multi Processing)*, and *MPP (Massive Parallel Processing)*.~~ 

- ~~*SMP mode:* Just one node is running CAS. This allows a single node to take advantage of CAS without having to invest in a large cluster or cloud environment. You get the advantage of running the analytics on multiple processors and the same functionality as an MPP system. Hadoop is not available in SMP mode.~~ 
- ~~*MPP mode:* CAS supports workers to help offload the analytics and spread the data out to allow for parallel processing on the multiple worker nodes. MPP allows the user to create a session with as many worker nodes as they need up to the current number of workers available. In this environment Hadoop is available if desired. With Hadoop, the controller is the name node and the workers run the analytics on the CAS data. CAS data is known as “in-memory tables”.~~ 

~~These two modes allow an administrator a lot of freedom to configure CAS in a way that helps the users. A user might want to bring up a CAS in SMP mode on a laptop to work on an application that will eventually be deployed onto a large MPP cluster.~~ 
#### ***~~Sessions~~***
~~CAS uses sessions to track users and offers a full Security interface to protect data at the file and column levels. The sessions <a name="_int_odb83so3"></a>provide isolation for the user, which protects the integrity of the server. The purpose of connecting to CAS is to execute server requests. A user must create a session to <a name="_int_4cautj8c"></a>submit a request. The user has two ways to connect to the server:~~ 

- ~~REST interface (HTTP-based)~~  
- ~~Binary interface (ProtoBuf-based)~~  

~~The user must be authenticated by CAS to create a session. The supported authentication methods include Kerberos, username/password, LDAP/PAM/OAUTH. Once the user has been authenticated, the server controller creates a session controller process for the user and a session worker process for each worker in the session. As you can see, the server is not just one process, but a series of interconnected processes across many hosts.~~

### **~~SAS Microservices and Web Applications (STATELESS NODE POOL)~~** 
~~SAS Viya has many services often referred to as microservices. These services are found in the stateless node pool. They consist of several services such as auditing, authentication, etc. Also, grouped with these services are a set of stateless web applications that support the SAS Viya deployment. Some of the web apps that run as part of the stateless node pool are things like SAS Data Studio, SAS Model Manager and SAS Data Explorer.~~
### **~~SAS Compute Services (COMPUTE NODE POOL)~~** 
~~SAS Compute Services are a set of SAS Viya microservices that <a name="_int_g3rj0osc"></a>provide API endpoints for requesting a SAS Compute Server session. The <a name="_int_7n1p2cc1"></a>compute service also <a name="_int_2f0lo2rp"></a>provides API endpoints for creating and managing <a name="_int_h3xm1i2l"></a>compute contexts, which are specifications that hold all the information that is needed to run a <a name="_int_2fo3fsbl"></a>compute server.~~ 
### **~~Infrastructure Services (STATEFUL NODE POOL)~~** 
~~The commodity services are basically the data management and storage services. They are made of several open-source technologies such as the internal SAS Postgres database, as well as Consul and RabbitMQ for messaging. This is where the critical operational data is stored. These services are I/O intensive.~~ 
### **~~OpenShift Routes (CONNECT NODE POOL)~~**
~~This node pool is where system and connection services will be deployed. This is how external processes access internal SAS services. The OpenShift Ingress services <a name="_int_plvfipup"></a>runs in this node pool along with SAS Connect services. SAS Connect allows other SAS Viya deployments including legacy deployments such as SAS Viya 9 or Viya 3.5 to submit jobs into the SAS Viya OpenShift deployment. This is a great method of migration. The migration can begin by having the old jobs submitting to the new platform.~~ 
### **~~OpenShift Monitoring and Logging (DEFAULT NODE POOL)~~** 
~~This is the node pool set up for non-SAS workloads. When customers have external applications that need to connect to SAS Compute or CAS services, this is where the application can run. Also, any non-tainted pod will run in this node pool. Other services such as <a name="_int_h0ouliqo"></a>monitoring and logging services will run in this node pool such as Grafana, Prometheus Kibana, Elasticsearch, etc.~~ 

## **SAS Viya on OpenShift Deployment**
### **Deployment Prerequisites**  
The following table provides the deployment prerequisites that are required before starting a deployment:

|**Required Component**|**Detailed Requirements**|
| :- | :- |
|Red Hat OpenShift|<p>Red Hat OpenShift Container Platform (OCP) 4.10.*x* to 4.12.x on VMware vSphere 7.0.1 or later. These versions align with the supported versions of Kubernetes (1.23.x - 1.25.x).</p><p>SAS has tested only with a user-provisioned infrastructure (UPI) installation.</p>|
|Compliant machines|<p>Red Hat Enterprise CoreOS (RHCOS) is the only operating system that Red Hat supports for OCP control plane nodes. RHCOS is recommended for worker nodes, but Red Hat Enterprise Linux is supported as an option. </p><p>SAS has tested only with RHCOS.</p><p>Recommended node sizes are provided in [Sizing Recommendations for OpenShift](https://go.documentation.sas.com/doc/en/sasadmincdc/v_039/itopssr/n08i2gqb3vflqxn0zcydkgcood20.htm#p04uz29tbignsin10sk5ld8h6jn0).</p>|
|Cluster ingress|<p>Only the OpenShift Ingress Operator is supported.</p><p>You must modify the kustomization.yaml file to enable routes. For more information, see [Create the File](https://documentation.sas.com/doc/en/itopscdc/v_039/dplyml0phy0dkr/n0g237aqo6pz1in1t19wjb94j9bi.htm#n08dvkro120s4on1g8wbqxpbcspt) in *SAS Viya: Deployment Guide*.</p><p>Verify that the requirements in [Cluster Ingress Requirements](https://documentation.sas.com/doc/en/itopscdc/v_039/itopssr/n1ika6zxghgsoqn1mq4bck9dx695.htm#p1eeoaos2nm7jsn1q3ymhui62j9v) have also been met.</p>|
|Kubernetes LoadBalancer services|<p>Usage varies based on the underlying infrastructure. See the [endpointPublishingStrategy configuration parameter](https://docs.openshift.com/container-platform/4.12/networking/ingress-operator.html#nw-ingress-controller-configuration-parameters_configuring-ingress) section of the *OpenShift Ingress Operator* documentation for information about whether a LoadBalancer service is used by your cluster infrastructure.</p><p>If you are using a load balancer, application gateway, or reverse proxy to function as the "front door" to your cluster, see [(Optional) Additional Requirements for Proxy Environments](https://go.documentation.sas.com/doc/en/sasadmincdc/v_039/itopssr/n1ika6zxghgsoqn1mq4bck9dx695.htm#n1l60mf73bkmqqn11tg9ba750v0f) for information about required configuration.</p>|
|Node labels|<p>One or more nodes should be fully dedicated to (that is, labeled and tainted for) the CAS server. This recommendation depends on whether your deployment uses the CAS server.</p><p>SAS strongly recommends labeling at least one node for compute workloads.</p><p>**Note:** If your deployment includes SAS Workload Management, you must label at least one node for the compute workload class. For more information, see [Plan the Workload Placement](https://go.documentation.sas.com/doc/en/sasadmincdc/v_039/dplyml0phy0dkr/p0om33z572ycnan1c1ecfwqntf24.htm) in *SAS Viya Platform: Deployment Guide*.</p>|
|A certificate generator to enable TLS|<p>The OpenSSL-based certificate generator supplied by SAS is used by default. You can instead use cert-manager if you install it in the cluster and take a few additional steps.</p><p>**Note:** SAS Viya platform deployments on OCP 4.11 using cert-manager require a fix that is only available in cert-manager v1.10 and later.</p><p>For more information about your options for TLS support and certificate management, see [TLS Requirements](https://go.documentation.sas.com/doc/en/sasadmincdc/v_039/itopssr/n18dxcft030ccfn1ws2mujww1fav.htm#p0bskqphz9np1ln14ejql9ush824).</p>|
|cert-utils-operator|This community-supported operator from Red Hat is required to manage certificates for TLS support and create keystores. For more information, see <https://github.com/redhat-cop/cert-utils-operator/blob/master/README.md>.|


### **Required and Optional SCCs**
The required and optional custom Security Context Constraints (SCCs) for running SAS Viya on Red Hat OpenShift are listed here, along with a description and why they are needed.  

In a Red Hat OpenShift environment, each Kubernetes pod is started with a default association with the **restricted** SCC, which limits the privileges that each pod can request.

Most SAS Viya platform pods are deployed in the **restricted** SCC, which applies the highest level of security. Two other predefined SCCs are used by default. In addition, a few custom SCCs are either required by essential SAS Viya platform components, such as the CAS server, or optional with specific SAS offerings that might be included in your software order.

**CLUSTER ADMIN:** A SCC acts like a request for privileges from the OpenShift API. In an OpenShift environment, each Kubernetes pod starts up with an association with a specific SCC, which limits the privileges that the pod can request. 

An administrator configures each pod to run with a certain SCC by granting the corresponding service account for that pod access to the SCC. For example, if pod A requires its own SCC, an administrator must grant access to that SCC for the service account under which pod A is launched. 

Use the OpenShift CLI tool (**oc**) to apply the SCC, and to assign the SCC to a service account. Refer to the SCC example files provided in the **$deploy/sas-bases/examples** folder.

1. Apply the SCC with the following command:

**oc apply -f example-scc.yaml** 

1. Bind the SCC to the service account with the following command:

**oc -n <name-of-namespace> adm policy add-scc-to-user <SCC Name> -z <Service Account Name>**

For additional details about SCCs, please see the following:

- For a full list and description of the required SCCs, see [Security Context Constraints and Service Accounts](https://documentation.sas.com/doc/en/itopscdc/v_039/dplyml0phy0dkr/p1h8it1wdu2iaxn1bkd8anfcuxny.htm#p09z7ivwp61280n1jezh6i6qmoml) in *SAS Viya Platform: Deployment Guide*.
- For SCC types, see [SCCs and Pod Service Accounts](https://documentation.sas.com/doc/en/itopscdc/v_039/itopssr/n0bqwd5t5y2va7n1u9xb57lfa9wx.htm#p1qz3rq1f758xkn1pctnlw7c3kn6) in *System Requirements for the SAS Viya Platform*.
- For more information for each SCC, see the README.md file (for Markdown format) below the **$deploy/sas-bases/examples** folder or below **$deploy/sas-bases/docs** (for HTML format).
- For more information about applying SCCs with OpenShift, see the Red Hat blog titled “[Managing SCCs in OpenShift]”.


|**Service Account Name / SCC Name**|**REQUIRED<br>or<br>Optional**|**When needed**|
| :- | :-: | :- |
|**sas-cas-server**|**REQUIRED**|CAS with cloud native storage|
|**cas-server-scc-host-launch**|**Optional**|CAS with host launch storage|
|**cas-server-scc-sssd**|**Optional**|CAS with a custom SSSD|
|**sas-opendistro**|**REQUIRED**|<p>Internal instance of OpenSearch</p><p>**NOTE**: A MachineConfig can be used as an alternative to this SCC.</p>|
|**sas-connect-spawner**|**Optional**|<p>SAS/CONNECT servers launch in the Spawner pod, versus dynamically launched their own pods.</p><p>**NOTE**: Only needed with legacy SAS clients. Current SAS clients use dynamically launched pods. See the [SAS Platform Operations Guide](https://documentation.sas.com/doc/en/itopscdc/v_039/dplyml0phy0dkr/p0om33z572ycnan1c1ecfwqntf24.htm#n1or76vxaxyq38n162ayf8jqals1), for more information.</p>|
|**sas-esp-project / nonroot**|**Optional**|SAS Event Stream Processing is included in your deployment|
|**sas-microanalytic-score**|**Optional**|The sas-micro-analytic-score pod uses NFS volume mounts|
|**sas-model-publish-kaniko / anyuid**|**Optional**|The kaniko service is used to publish models|
|**sas-model-repository**|**Optional**|The sas-model-repository pod uses NFS volume mounts|
|**sas-programming-environment / sas-watchdog**|**Optional**|SAS Watchdog is included in your deployment|
|**sas-programming-environment / anyuid** |**Optional**|SAS Watchdog is included in your deployment|
|**sas-programming-environment / hostmount-anyuid** |**Optional**|SAS Watchdog is included in your deployment, and using hostPath mounts.|

#### ***sas-cas-server***
**REQUIRED:** Every deployment on OpenShift must apply one of the SCCs for the CAS server. By default, in a greenfield SAS Viya deployment for a new customer, we expect CAS to use cloud native storage and not need host launch capabilities.  So, at a minimum, the **cas-server-scc** SCC would be applied.

**OPTIONAL:** If host launch or SSSD are required, then one of those related SCCs would be need to be applied in place of the **cas-server-scc** SCC.  Typically, host launch is only needed for existing SAS customers who are migrating to the cloud and already have access controls to their data defined using host users and groups.

If the custom group "CAS Host Account Required" is used, apply the **cas-server-scc-host-launch** SCC. 
If the CAS server is configured to use a custom SSSD, apply the **cas-server-scc-sssd** SSC. For more information on enabling SSSD, see [Configure SSSD](https://documentation.sas.com/doc/en/itopscdc/v_039/dplyml0phy0dkr/n08u2yg8tdkb4jn18u8zsi6yfv3d.htm#n0pshy2wfr9aw0n1g9p5sbbhzyqr).

***Why the SCC is needed:*** 

- CAS relies on SETUID, SETGID, and CHOWN capabilities. CAS is launched by a SETUID root executable called caslaunch. By default, caslaunch starts the CAS controller running under the runAsUser/runAsGroup values and a root process named launchsvcs. Caslaunch connects these processes with a pipe. 
- The SETUID and SETGID capabilities are required by launchsvcs in order to launch session processes under the user’s operating-system (host) identity instead of launching them using runAsUser/runAsGroup values. 
- The CHOWN capability is necessary to support Kerberos execution, which requires modification of the ownership of the cache file that is created by a direct Kerberos connection. By default, the cache file is owned by runAsUser/runAsGroup identities, but in order to support Kerberos, it must be owned by the user’s host identity.


#### ***sas-connect-spawner***
**OPTIONAL:** By default, no SCC is required for SAS/CONNECT and you can skip this item. The SCC is required only if you intend to launch your SAS/CONNECT servers in the Spawner pod, rather than in their own pods. 

***Why the SCC is needed*:** When launching SAS/CONNECT servers in the Spawner pod, the SAS/CONNECT Launcher must be able to launch the SAS/CONNECT Server under end user identity.

#### ***sas-esp-project***
**OPTIONAL:** To determine if your deployment includes SAS Event Stream Processing, look for it in the "License Information" section of your Software Order Email (SOE) for the list of products that are included in your order. If your SOE is unavailable, look for $deploy/sas-bases/examples/sas-esp-operator in your deployment assets. If that directory exists, then your deployment includes SAS Event Stream Processing. If it does not exist, skip this SCC.

To run SAS Event Stream Processing projects with a user other than "sas", you must bind the **sas-esp-project** service account to the **nonroot** SCC.

The **nonroot** SCC is a standard SCC defined by OpenShift. For more information about the **nonroot** SCC, see [Managing SCCs in OpenShift].

#### ***sas-microanalytic-score***
**OPTIONAL:** To determine if the **sas-microanalytic-score** SCC is needed for your deployment, check for a README file in your deployment assets at $deploy/sas-bases/overlays/sas-microanalytic-score/service-account/README.md. If the README file is present, then the SCC is available for deployments on OpenShift.

***Why the SCC is needed:*** Security context constraints are required in an OpenShift cluster if the **sas-micro-analytic-score** pod needs to mount an NFS volume. If the Python environment is made available through an NFS mount, the service account requires NFS volume mounting privileges.

#### ***sas-model-publish-kaniko***
**OPTIONAL:** To determine if the **sas-model-publish-kaniko** service account exists in your deployment, check for a README file in your deployment assets at $deploy/sas-bases/examples/sas-model-publish/kaniko/README.md. If the README file is present and you plan to publish models with SAS Model Manager or SAS Intelligent Decisioning to containers using kaniko, you must bind the **sas-model-publish-kaniko** service account to the **anyuid** SCC. The **anyuid** SCC is a standard SCC defined by OpenShift. 

For more information about the **anyuid** SCC, see [Managing SCCs in OpenShift]. 

#### ***sas-model-repository***
**OPTIONAL:** To determine if the **sas-model-repository** SCC is needed for your deployment, check for a README file in your deployment assets at $deploy/sas-bases/overlays/sas-model-repository/service-account/README.md. If the README file is present, then the SCC is available and might be required for deployments on OpenShift.

***Why the SCC is needed:*** The **sas-model-repository** pod requires a service account with privileges if the Python environment is made available through an NFS mount. NFS volumes are not permitted in the **restricted** SCC, so an SCC that has NFS in the allowed volumes section is required.

#### ***sas-opendistro***
**REQUIRED:** Every SAS Viya platform deployment contains sas-opendistro. 

For optimal performance, deploying OpenSearch software requires the **vm.max\_map\_count** kernel parameter to be set on the OpenShift nodes running the stateful workloads to ensure there is adequate virtual memory available for use with mmap for accessing the search indices. 

There are two methods available to set the **vm.max\_map\_count** kernel parameter:

- Use an init container to add the kernel parameter, which requires the **sas-opendistro** SCC.
- Use a MachineConfig to add the kernel parameter, which does not require a custom SCC.

**OPENSEARCH SYSCTL INIT CONTAINER**

If your deployment uses an internal instance of OpenSearch, the **sas-opendistro** SCC is required.

Deploying OpenSearch on OpenShift requires changes to a few kernel settings. The sysctl-transformer.yaml file can apply the necessary sysctl parameters to configure the kernel, but it requires the following special privileges: **allowPrivilegeEscalation** option enabled, **allowPrivilegedContainer** option enabled, and **runAsUser** set to **RunAsAny**.

Therefore, before you apply the **sas-opendistro-scc.yaml** file, you must modify it to enable these privileges. 

***Why the SCC is needed:*** To provide a method to set this kernel argument automatically, the SAS Viya platform includes an optional transformer as part of the internal-elasticsearch overlay that adds an init container to automatically set this parameter. The init container must be run at a privileged level (using both the **privileged = true** and **allowPrivilegeEscalation = true** security context options) since it modifies the kernel parameters of the host. The container terminates after it sets the kernel parameter, and the OpenSearch software will then proceed to start as a non-privileged container.

**OPENSEARCH SYSCTL MACHINECONFIG**

` `If privileged containers are not allowed in your environment, a *MachineConfig* can be used to set the **vm.max\_map\_count** kernel parameter for OpenSearch, as an alternative to using this SCC with the init container. All nodes that run workloads in the [stateful workload class](https://documentation.sas.com/doc/en/itopscdc/v_039/dplyml0phy0dkr/p0om33z572ycnan1c1ecfwqntf24.htm) are affected by this requirement.

Perform the following steps; refer to [**Adding kernel arguments to nodes**](https://docs.openshift.com/container-platform/4.12/nodes/nodes/nodes-nodes-managing.html#nodes-nodes-kernel-arguments_nodes-nodes-jobs) in the OpenShift documentation.

1. List existing **MachineConfig** objects for your OpenShift Container Platform cluster to determine how to label your machine config:

**oc get machineconfig**

1. Create a **MachineConfig** object file that identifies the kernel argument 
   (for example, **05-worker-kernelarg-<a name="_hlk133266210"></a>vm.max\_map\_count.yaml**)

**apiVersion: machineconfiguration.openshift.io/v1**

**kind: MachineConfig**

**metadata:**

`  `**labels:**

`    `**machineconfiguration.openshift.io/role: worker**

`  `**name: 05-worker-kernelarg-vmmaxmapcount**

**spec:**

`  `**kernelArguments:**

`    `**- vm.max\_map\_count=262144**

3. Create the new machine config:

**oc create -f 05-worker-kernelarg-vm.max\_map\_count.yaml**

4. Check the machine configs to see that the new one was added:

**oc get machineconfig**

4. Check the nodes:

**oc get nodes**

You can see that scheduling on each worker node is disabled as the change is being applied.

4. Check that the kernel argument worked by going to one of the worker nodes and verifying with the sysctl command or by listing the kernel command line arguments (in /proc/cmdline on the host):

**oc debug node/vsphere-k685x-worker-4kdtl**

**sysctl vm.max\_map\_count**

**exit**

**Example output**

**Starting pod/vsphere-k685x-worker-4kdtl-debug ...**

**To use host binaries, run `chroot /host`**

**sh-4.2# sysctl vm.max\_map\_count** 

**vm.max\_map\_count = 262144**

**sh-4.2# exit**

If listing the /proc/cmdline file, you should see the **vm.max\_map\_count=262144** argument added to the other kernel arguments.

#### ***sas-watchdog-scc***
**OPTIONAL:** SAS Watchdog is included in every SAS Viya platform deployment. If you choose to deploy SAS Watchdog, the SCC is required. For more information, see the README file at $deploy/sas-bases/overlays/sas-programming-environment/watchdog/README.md (for Markdown format) or at $deploy/sas-bases/docs/configuring\_sas\_compute\_server\_to\_use\_sas\_watchdog.htm (for HTML format).

***Why the SCC is needed:*** SAS Watchdog monitors processes to ensure that they comply with the terms of LOCKDOWN system option. It emulates the restrictions imposed by LOCKDOWN by restricting access only to files that exist in folders that are allowed by LOCKDOWN. It therefore requires elevated privileges provided by the custom SCC.

Alternatively, you can bind the **hostmount-anyuid** or **anyuid** SCC to the **sas-programming-environment** service account. If you have already bound the SAS Watchdog service account to the **sas-watchdog** SCC, you cannot bind it again.

Using the **anyuid** SCC is preferred. Using the **hostmount-anyuid** SCC is only required if you use hostPath mounts.

**Note:** The **hostmount-anyuid** and **anyuid** SCCs are standard SCCs defined by OpenShift. For more information, see [Managing SCCS in OpenShift][Managing SCCs in OpenShift].



### **Workload Node Placement**
The SAS Viya platform consists of multiple [Workload Classes](https://documentation.sas.com/doc/en/itopscdc/v_039/dplyml0phy0dkr/p0om33z572ycnan1c1ecfwqntf24.htm#n0jo17lrlk83rsn1vvs2wqmewkt7). Each class of workload has a unique set of attributes that you must consider when planning your deployment. When planning the placement of the workload, it is helpful to think beyond the initial deployment and to also consider the Kubernetes maintenance life cycle.

**IMPORTANT** SAS strongly recommends labeling and tainting all your nodes, especially the CAS nodes. Note, however, that the connect workload class is only required if you are not using dynamically launched pods. For more information about dynamically launched pods, see [connect](https://documentation.sas.com/doc/en/itopscdc/v_039/dplyml0phy0dkr/p0om33z572ycnan1c1ecfwqntf24.htm#n1or76vxaxyq38n162ayf8jqals1).

Properties associated with the pods in your deployment describe the type of work that they perform. By using the following commands, you can configure where you want each class of pods to run, thereby enabling you to manage the associated workload.

The following table summarizes the SAS Viya platform default deployment for each workload class:

|***Default Taints and nodeAffinity Settings per Class***|
| :- |
|**Workload Class**|**Default Settings**|
|CAS workloads|<p>Prefer to schedule on nodes that are labeled: <br>**workload.sas.com/class=cas**</p><p>Tolerate the taint: <br>**workload.sas.com/class=cas:NoSchedule**</p>|
|<p>Connect workloads</p><p>**Note:** This workload class is not required if you are using dynamically launched pods.</p>|<p>Prefer to schedule on nodes that are labeled: <br>**workload.sas.com/class=connect**</p><p>Tolerate the taint: <br>**workload.sas.com/class=connect:NoSchedule**</p>|
|Compute workloads|<p>Prefer to schedule on nodes that are labeled: <br>**workload.sas.com/class=compute**</p><p>Tolerate the taint: <br>**workload.sas.com/class=compute:NoSchedule**</p>|
|Stateful workloads|<p>Prefer to schedule on nodes that are labeled: <br>**workload.sas.com/class=stateful**</p><p>Tolerate the following taints:</p><p>- **workload.sas.com/class=stateful:NoSchedule**</p><p>- **workload.sas.com/class=stateless:NoSchedule**</p>|
|Stateless workloads|<p>Prefer to schedule on nodes that are labeled: <br>**workload.sas.com/class=<a name="_hlk133860260"></a>stateless**</p><p>Tolerate the following taints:</p><p>- **workload.sas.com/class=stateless:NoSchedule**</p><p>- **workload.sas.com/class=stateful:NoSchedule**</p>|

#### ***Manual Workload Placement Configuration***
SAS requires that you identify the node or nodes on which CAS pods should be scheduled. SAS further recommends that you identify the nodes on which all classes should be scheduled. Follow the commands provided in “[Place the Workload on Nodes](https://documentation.sas.com/doc/en/itopscdc/v_039/dplyml0phy0dkr/p0om33z572ycnan1c1ecfwqntf24.htm#n0wj0cyrn1pinen1wcadb0rx6vbm)**”** from the SAS Viya Platform Operations manual to manually configure the nodes with the labels and taints to properly place the workload on the nodes in your deployment.

#### ***Automated Workload Placement Configuration***
Red Hat OpenShift provides machine management as an automation method to manage the underlying cloud platform through a machine object, which is a subset of the node object.  This allows for the definition of compute machine sets that can be sized and matched to workloads, and scaled to meet workload demand.

Node labels and taints can be included within the compute MachineSet definitions to automate their workload placement configuration at the compute machine creation time.

### **OpenShift Machine Management and Autoscaling**
You can use machine management to flexibly work with underlying infrastructure of cloud platforms like vSphere to manage the OpenShift Container Platform cluster. You can control the cluster and perform auto-scaling, such as scaling up and down the cluster based on specific workload policies.

The OpenShift Container Platform cluster can horizontally scale up and down when the load increases or decreases. It is important to have a cluster that adapts to changing workloads.

Machine management is implemented as a CRD object that defines a new unique object Kind in the cluster and enables the Kubernetes API server to handle the object’s entire lifecycle. The Machine API Operator provisions the following resources: *Machine*, *MachineSet*, *ClusterAutoScaler*, *MachineAutoScaler*, and *MachineHealthCheck*.

As a cluster administrator, you can perform the following tasks with compute machine sets:

- [Create a compute machine set on vSphere](https://docs.openshift.com/container-platform/4.8/machine_management/creating_machinesets/creating-machineset-vsphere.html#creating-machineset-vsphere).
- [Manually scale a compute machine set](https://docs.openshift.com/container-platform/4.12/machine_management/manually-scaling-machineset.html#manually-scaling-machineset) by adding or removing a machine from the compute machine set
- [Modify a compute machine set](https://docs.openshift.com/container-platform/4.12/machine_management/modifying-machineset.html#modifying-machineset) through the MachineSet YAML configuration file.
- [Delete a machine](https://docs.openshift.com/container-platform/4.12/machine_management/deleting-machine.html#deleting-machine).
- [Create infrastructure compute machine sets](https://docs.openshift.com/container-platform/4.12/machine_management/creating-infrastructure-machinesets.html#creating-infrastructure-machinesets).
- Configure and deploy a [machine health check](https://docs.openshift.com/container-platform/4.12/machine_management/deploying-machine-health-checks.html#deploying-machine-health-checks) to automatically fix damaged machines in a machine pool

Autoscale your cluster to ensure flexibility to changing workloads. To [autoscale](https://docs.openshift.com/container-platform/4.12/machine_management/applying-autoscaling.html#applying-autoscaling) your OpenShift Container Platform cluster, you must first deploy a cluster autoscaler, and then deploy a machine autoscaler for each <a name="_int_jvmpgk2v"></a>compute machine set. 

- The [*cluster autoscaler*](https://docs.openshift.com/container-platform/4.12/machine_management/applying-autoscaling.html#cluster-autoscaler-about_applying-autoscaling) increases and decreases the size of the cluster based on deployment needs. 
- The [*machine autoscaler*](https://docs.openshift.com/container-platform/4.12/machine_management/applying-autoscaling.html#machine-autoscaler-about_applying-autoscaling) adjusts the number of machines in the machine sets that you deploy in your OpenShift Container Platform cluster.

#### ***Example Machine Management YAML Files***
Example YAML files are available for the ClusterAutoScaler, MachineAutoScaler and MachineSet definitions from the following repo: <https://github.com/redhat-gpst/sas-viya-openshift>

The following table provides the details about the example definition files provided for each of the SAS Viya [Workload Classes](https://documentation.sas.com/doc/en/itopscdc/v_039/dplyml0phy0dkr/p0om33z572ycnan1c1ecfwqntf24.htm#n0jo17lrlk83rsn1vvs2wqmewkt7), based on the [minimum sizing recommendations for OpenShift](https://documentation.sas.com/doc/en/itopscdc/v_039/itopssr/n08i2gqb3vflqxn0zcydkgcood20.htm#p04uz29tbignsin10sk5ld8h6jn0).

|<a name="_hlk134185241"></a>**Workload Class**|**Example MachineSet file**|**Example MachineAutoScaler file**|
| :- | :- | :- |
|<p>CAS workloads (SMP)</p><p>CAS workloads (MPP)</p>|<p>**cas-smp-machineset.yaml**</p><p>**cas-mpp-machineset.yaml**</p><p></p>|<p>**cas-smp-autoscaler.yaml**</p><p>**cas-mpp-autoscaler.yaml**</p>|
|Connect workloads|**connect-machineset.yaml**|**connect-autoscaler.yaml**|
|Compute workloads|**compute-machineset.yaml**|**compute-autoscaler.yaml**|
|Stateful workloads|**stateful-machineset.yaml**|**stateful-autoscaler.yaml**|
|Stateless workloads|**stateless-machineset.yaml**|**stateless-autoscaler.yaml**|


#### ***MachineSet***
To deploy the machine set, you create an instance of the *MachineSet* resource.

Create a MachineSet definition YAML file for each SAS Viya workload class needed, using the examples available above.

1. Create a YAML file for the *MachineSet* resource that contains the customized resource definition for your selected SAS Viya workload class, using the examples available from the repo above.
   Ensure that you set the <*clusterID*> and <*role*> parameter values that apply to your environment.
1. If you are not sure which value to set for a specific field, you can check an existing machine set from your cluster:

**oc get machinesets -n openshift-machine-api**

1. Check values of a specific machine set:

**oc get machineset <machineset\_name> -n openshift-machine-api -o yaml**

1. Create the new MachineSet CR:

**oc create -f cas-smp-machineset.yaml**

1. View the list of machine sets:

**oc get machineset -n openshift-machine-api**

When the new machine set is available, the DESIRED and CURRENT values match. If the machine set is not available, wait a few minutes and run the command again.

For more information about defining MachineSets, refer to the [OpenShift documentation](https://docs.openshift.com/container-platform/4.12/machine_management/creating_machinesets/creating-machineset-vsphere.html#machineset-yaml-vsphere_creating-machineset-vsphere).


#### ***ClusterAutoScaler***
Applying autoscaling to an OpenShift Container Platform cluster involves deploying a cluster autoscaler and then deploying machine autoscalers for each machine type in your cluster.

To deploy the cluster autoscaler, you create an instance of the *ClusterAutoscaler* resource.

1. <a name="_hlk133586577"></a>Create a YAML file for the *ClusterAutoscaler* resource that contains the customized resource definition (for example, **clusterautoscaler.yaml**).
2. Create the resource in the cluster:

**oc create -f clusterautoscaler.yaml**

**IMPORTANT**: Ensure that the *maxNodesTotal* value in the *ClusterAutoscaler* resource definition that you create is large enough to account for the total possible number of machines in your cluster. This value must encompass the number of control plane machines and the possible number of compute machines that you might scale to.

For more information about defining the *ClusterAutoscaler* resource definition, refer to the [OpenShift documentation](https://docs.openshift.com/container-platform/4.12/machine_management/applying-autoscaling.html#cluster-autoscaler-cr_applying-autoscaling).


#### ***MachineAutoScaler***
To deploy the machine autoscaler, you create an instance of the *MachineAutoscaler* resource.

1. Create a YAML file for the *MachineAutoscaler* resource that contains the customized resource definition (for example, <a name="_hlk133580713"></a>**cas-mpp-autoscaler.yaml**).

1. Create the resource in the cluster:

**oc create -f cas-mpp-autoscaler.yaml**

For more information about defining the *MachineAutoScaler* resource definition, refer to the [OpenShift documentation](https://docs.openshift.com/container-platform/4.12/machine_management/applying-autoscaling.html#machine-autoscaler-about_applying-autoscaling).



### **SAS Viya Customization**

#### ***Cloud Native Storage Integration***
OpenShift on VMware vSphere supports the dynamic volume provisioning provided by the in-tree and CSI vSphere storage provider.

This provides RWO mode

OpenShift Data Foundation (ODF) can also utilize the dynamic volume provisioning provided by the in-tree and CSI vSphere storage provider, and provide object and file storage; RWO and RWX modes.

For additional information, see the SAS blogs titled “

[SAS Viya Temporary Storage on Red Hat OpenShift – Part 1](https://communities.sas.com/t5/SAS-Communities-Library/SAS-Viya-Temporary-Storage-on-Red-Hat-OpenShift-Part-1/ta-p/858834)

[SAS Viya Temporary Storage on Red Hat OpenShift – Part 2: CAS DISK CACHE](https://communities.sas.com/t5/SAS-Communities-Library/SAS-Viya-Temporary-Storage-on-Red-Hat-OpenShift-Part-2-CAS-DISK/ta-p/859250)

DOC DRAFT END



**Background information for blog discussed in 2022…**
…the focus would be on the security topics, especially on the bits where we need additional privileges which are typically not given to project owners on OCP.

- I would start with saying that SAS Viya is a rather large software stack, not a single application
- Most of the stack is based on microservices, but there are other components as well: a database (postgres), a key-value store (consul), a search engine (OpenSearch), a message queue (RabbitMQ) and some runtime engines written by SAS (the SAS compute and the CAS engines)
- We try to “play by the book” when it comes to deployments on OpenShift:
  - Everything is deployed into a given target namespace (when possible)
  - We use operators wherever it makes sense (for operating the non-microservice components and for managing the software lifecycle)
  - Which is a great technology for us to lower the administration efforts of the platform (especially when considering the size)
  - Our components use the restricted SCC (Security Context Constraints) wherever possible
  - Which again is great in eliminating many potential attack vectors for most of our components (a lot of stuff that you do not have to worry about)
  - Most components can be deployed using a non-cluster-admin account
  - Which supports the common operations model found at customer sites, where OCP (OpenShift Container Platform) admins usually create the projects and provide project-level permissions to their internal clients
- There are some exceptions – some cannot be avoided due to Kubernetes standards; others are required because we need to support traditional programming language capabilities (backwards compatibility)
  - We use operators and therefore need to deploy some global resources (CRDs)
  - Some service accounts require a custom SCC, e.g., because we need to be able to pin the container process to a specific (non-root) id
  - Unfortunately, this impacts the operations model mentioned above – we do need support from OCP admins to be able to deploy … The extent of that extra support is documented and separated from the rest.
  - We are trying to minimize the extent of these non-standard situations. So far, we have removed one requirement completely and lowered the “severity” in two more cases. This is ongoing work, but honestly speaking we might not be able to come down to “zero” …

For me, the main conclusion for all of this would be that we try our best to stick to the best practices provided by Red Hat. There are some cases where we must leave the path but even in these situations, we still follow Red Hat’s guidelines.

[Managing SCCs in OpenShift]: https://cloud.redhat.com/blog/managing-sccs-in-openshift "opens in a new browser tab"
