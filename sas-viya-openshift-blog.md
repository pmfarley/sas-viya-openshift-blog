
# SAS Viya on Red Hat OpenShift 
_Authors: Patrick Farley and Hans-Joachim Edert_

In this blog, we will provide some basic technical information about SAS Viya, as well as a reference architecture for SAS Viya on Red Hat OpenShift Container Platform (OCP). This reference architecture will show how SAS Viya is containerized to run with Kubernetes on Red Hat OpenShift.

A few introductory words before we get into the technical details.

Since the launch of SAS Viya in 2020, SAS has offered a fully containerized analytic platform based on a cloud-native architecture. Due to the scale of the platform, SAS Viya requires Kubernetes as an underlying runtime environment and takes full advantage of the native benefits of this technology.  

SAS supports numerous Kubernetes distributions, both in the public and private cloud. In fact, many SAS customers - partly due to specific use cases that do not allow otherwise from a regulatory perspective, but also due to strategic considerations - prefer to run their application infrastructure in a private cloud environment. 

In these cases, Red Hat OpenShift provides an optimal foundation for the SAS software stack. OpenShift offers both a hardened Kubernetes with many highly valued enterprise features as an execution platform, but also comes with an extensive ecosystem with a particular focus on supporting DevSecOps capabilities.

SAS and Red Hat have enjoyed a productive partnership for more than a decade - while Red Hat Enterprise Linux was the preferred operating system in earlier SAS releases, today SAS has based their container images on the Red Hat Universal Base Image.

Moreover, for Red Hat OpenShift deployments, SAS takes advantage of the OpenShift Ingress Operator, the cert-utils operator, OpenShift GitOps for deployment (optionally), and integrates into OpenShift’s security approach which is based on SCCs (Security Context Constraints) as part of their deployments.

## SAS Viya on OpenShift Reference Architecture
SAS Viya is an integrated platform that covers the entire AI and Analytics lifecycle. Thus, it is not just a single application, but a suite of integrated applications. One of the fundamental differences here is the nature of the workload that SAS Viya brings to the OpenShift platform. This affects the need for resources (CPU, memory, storage) and entails special security-specific requirements.

Moving SAS Viya to OpenShift gives Viya unprecedented scalability that was unavailable in previous SAS releases. SAS takes advantage of the scalability by breaking Viya down into different workload types and recommends assigning each workload to a class of nodes, i.e., to a Machine Pool. This ensures that the proper resources are available to specific workloads. The following diagram shows the separation of workloads to pools.

<img width="1089" alt="image" src="https://user-images.githubusercontent.com/48925593/235716577-0ff2874b-3b9f-4404-b5e4-82961a4f6904.png">

_<div align="center">Figure 1</div>_

Note that the setup of pools is not mandatory and there might be good reasons to ignore the recommendation if the existing cluster infrastructure is not suitable for such a split. The placement of SAS workload classes can be enabled by applying predefined Kubernetes node labels and node taints.

In the current iteration of SAS Viya on OpenShift, SAS only supports VMware vSphere as the deployment platform. At the time of this document, BareMetal is on the roadmap as an alternative for on-premise deployments.  When deployed on a different infrastructure provider, such as Azure, AWS, Google or BareMetal, SAS Viya runs under the [“SAS Support for Alternative Kubernetes Distributions”](https://support.sas.com/en/technical-support/services-policies.html#k8s) policy.



## Core Platform
VMware vSphere 7.0.1 is the virtual machine platform that is currently supported. The details of VMware configuration will not be covered in this document with the assumption that VMware vSphere is well known in most environments. 

At the time this blog was written, Red Hat OpenShift versions 4.10 - 4.12 are supported for SAS Viya. Additional details are provided later within this document; for some of the specific OpenShift components that support SAS Viya deployment; but they are provided here, at a high-level:

- **OpenShift Ingress Operator**

   SAS has specific requirements for forwarding cookies during transaction execution. As such, they used special techniques using the HA-Proxy to make that happen. So, in this iteration only the OpenShift Ingress Operator is supported. 

- **OpenShift Ingress Operator**

   SAS has specific requirements for forwarding cookies during transaction execution. As such, they used special techniques using the HAProxy to make that happen. So, in this iteration only the OpenShift Ingress Operator is supported. 

- **OpenShift Routes**

   SAS prefers the use of native features with the environments with their products, so they take advantage of OpenShift routes. 

- **cert-utils-operator**

   SAS requires the use of this operator to manage certificates for TLS support and create keystores. 
   
- **cert-manager**

   SAS Viya supports two different certificate generators, which are used for enforcing full-stack TLS. The default generator uses OpenSSL and is supplied out-of-the-box by SAS. Alternatively, you can optionally deploy and use cert-manager to generate the certificates used to encrypt the pod-to-pod communication.  
   
   
- **Security Context Constraints**
   
   Security Context Constraints (SCCs) provide permissions to pods and are required to allow them to run. SAS requires multiple custom SCCs to support SAS Viya Services with OpenShift. The SAS documentation provides information about the required SCCs to help understand their use in your environment and to address any security concerns.  Further details about the required custom SCCs are provided later within this document.
   


### Deployment options – Red Hat OpenShift

There are various methods for [installing Red Hat OpenShift Container Platform on VMware vSphere](https://docs.openshift.com/container-platform/4.12/installing/installing_vsphere/preparing-to-install-on-vsphere.html),  including:

- _Installer-provisioned infrastructure_ (IPI) installation, which allows the installation program to pre-configure and automate the provisioning of the required resources.
- The [_Assisted Installer_](https://docs.openshift.com/container-platform/4.12/installing/installing_on_prem_assisted/installing-on-prem-assisted.html), which provides a user-friendly installation solution offered directly from the [Red Hat Hybrid Cloud Console](https://console.redhat.com/openshift/assisted-installer/clusters/~new).
- _User-provisioned infrastructure_ (UPI) installation, which provides a manual type of installation with the most control over the installation and configuration process.

An IPI installation results in an OCP cluster with the vSphere cloud provider configuration settings from the installation, which enables additional automation and dynamic provisioning after installation:
- [Persistent storage using the VMware CSI or in-tree driver operator](https://docs.openshift.com/container-platform/4.12/storage/container_storage_interface/persistent-storage-csi-vsphere.html).
- [Host Node management for Node pools using MachineSets](https://docs.openshift.com/container-platform/4.12/machine_management/index.html).
- [Autoscaling Nodes using the MachineAutoScaler and ClusterAutoScaler operators](https://docs.openshift.com/container-platform/4.12/machine_management/applying-autoscaling.html). 

Information about configuring these capabilities is supplied later within this document.

An [OCP installation on VMware vSphere](https://docs.openshift.com/container-platform/4.12/installing/installing_vsphere/preparing-to-install-on-vsphere.html) using the UPI or Assisted Installer methods can also be set up [post-installation with the vSphere cloud provider configuration](https://access.redhat.com/documentation/en-us/assisted_installer_for_openshift_container_platform/2022/html-single/assisted_installer_for_openshift_container_platform/index#vsphere-post-installation-configuration_installing-on-vsphere), to provide similar automation and dynamic provisioning capability as an IPI installation. This is out of the scope of this blog.


## SAS Viya Components

### SAS Cloud Analytic Server (CAS) (CAS Node Pool)
The core component of SAS Viya is the Cloud Analytics Server (CAS). CAS is the heart of the SAS Viya deployment. It is basically an in-memory analytics engine. Data is loaded from the required data source into some in-memory tables then clients connect to CAS to run analytical routines against the data. The data loaded into memory for CAS can also be flushed to disk. This is why SAS Viya requires persistent storage accessible by all compute nodes. 
The best way to understand CAS is to understand the terminology and concepts used in the server and how it all fits together to achieve the goals mentioned above.

### CAS Server Controller 
CAS consists of from 1 to N hosts. These hosts can be a part of a cluster in a node pool that carries specific resource requirements. One host is designated the "controller". The controller executes one process as the "server controller" and one process for each "server worker". The controller is the repository for the global state of the server. The client connects to the server controller using either the binary API port or the REST API port. Once the user has been authenticated, the server controller creates a session process on the controller and each worker that is to be included as part of this session. The client connection is then transferred to the session controller. At this point, the client can start submitting work to CAS.

### CAS Server Worker  
The purpose of the worker nodes is to execute the same request as the other workers and controllers, and process the data that is local to that worker. This allows the user or admin to spread data across the cluster to allow for local parallel processing of data that will come back together to produce a result. Just like the controller nodes, the worker nodes support one process as the server worker and one process for each session.

### Two Different Execution Modes 
CAS can be deployed in one of two modes:  SMP (Symmetric Multi Processing), and MPP (Massive Parallel Processing). 
- In SMP mode, there is just one node running CAS. This allows a single node to take advantage of CAS without having to invest in a large cluster or cloud environment. You get the advantage of running the analytics on multiple processors and the same functionality as an MPP system. Hadoop is not available in SMP mode. 
- In MPP mode, CAS supports workers to help offload the analytics and spread the data out to allow for parallel processing on the multiple workers. MPP allows the user to create a session with as many worker nodes as they need up to the current number of workers available. In this environment Hadoop is available if desired. With Hadoop, the controller is the name node and the workers run the analytics on the CAS data. CAS data is known as “in-memory tables”. These two modes allow an administrator a lot of freedom to configure CAS in a way that helps the users. A user might want to bring up a CAS in SMP mode on a laptop to work on an application that will eventually be deployed onto a large MPP cluster. 

### Sessions
CAS uses sessions to track users and offers a full Security interface to protect data at the file and column levels. The sessions provide isolation for the user, which protects the integrity of the server. The purpose of connecting to CAS is to execute server requests. A user must create a session to submit a request. The user has two ways to connect to the server: 
- REST interface (HTTP-based)  
- Binary interface (ProtoBuf-based)  

The user must be authenticated by CAS to create a session. The supported authentication methods include Kerberos, username/password, LDAP/PAM/OAUTH. Once the user has been authenticated, the server controller creates a session controller process for the user and a session worker process for each worker in the session. As you can see, the server is not just one process, but a series of interconnected processes across many hosts.

### SAS Microservices and Web Applications (Stateless Node Pool) 
SAS Viya has many services often referred to as microservices. These services are found in the stateless node pool. They consist of several services such as auditing, authentication, etc. Also, grouped with these services are a set of stateless web applications that support the SAS Viya deployment. Some of the web apps that run as part of the stateless node pool are things like SAS Data Studio, SAS Model Manager and SAS Data Explorer.

### SAS Compute Services (Compute Node Pool) 
SAS Compute Services are a set of SAS Viya microservices that provide API endpoints for requesting a SAS Compute Server session. The compute service also provides API endpoints for creating and managing compute contexts, which are specifications that hold all the information that is needed to run a compute server. 

### Infrastructure Services (Stateful Node Pool) 
The commodity services are basically the data management and storage services. They are made of several open-source technologies such as the internal SAS Postgres database, as well as Consul and RabbitMQ for messaging. This is where the critical operational data is stored. These services are I/O intensive. 

### OpenShift Routes (System or Connect Node Pool)
This node pool is where system and connection services will be deployed. This is how external processes access internal SAS services. The OpenShift Ingress services runs in this node pool along with SAS Connect services. SAS Connect allows other SAS Viya deployments including legacy deployments such as SAS Viya 9 or Viya 3.5 to submit jobs into the SAS Viya OpenShift deployment. This is a great method of migration. The migration can begin by having the old jobs submitting to the new platform. 

### OpenShift Monitoring and Logging (Default Node Pool) 
This is the node pool set up for non-SAS workloads. When customers have external applications that need to connect to SAS Compute or CAS services, this is where the application can run. Also, any non-tainted pod will run in this node pool. Other services such as monitoring and logging services will run in this node pool such as Grafana, Prometheus Kibana, Elasticsearch, etc. 



## SAS Viya on OpenShift Deployment

### Deployment Prerequisites

The following table summarizes cluster requirements in a Red Hat OpenShift environment:

| Required Component | Detailed Requirements |
|--------------------|-----------------------|
| Kubernetes         | Red Hat OpenShift Container Platform (OCP) 4.10 - 4.12 on VMware vSphere 7.0.1 or later. These versions align with the supported versions of Kubernetes (1.23.x - 1.25.x). SAS has tested only with a user-provisioned infrastructure (UPI) installation.|
| Compliant machines | Red Hat Enterprise CoreOS (RHCOS) is the only operating system that Red Hat supports for OCP control plane nodes. Red Hat enables you to use Red Hat Enterprise Linux on the worker nodes, but SAS has tested only with RHCOS. Recommended node sizes are provided in [Sizing Recommendations for OpenShift](https://documentation.sas.com/doc/en/itopscdc/v_039/itopssr/n08i2gqb3vflqxn0zcydkgcood20.htm#p04uz29tbignsin10sk5ld8h6jn0).
| Cluster ingress | Only the OpenShift Ingress Operator is supported. You must modify the kustomization.yaml file to enable routes. For more information, see [Create the File](https://documentation.sas.com/doc/en/itopscdc/v_039/dplyml0phy0dkr/n0g237aqo6pz1in1t19wjb94j9bi.htm#n08dvkro120s4on1g8wbqxpbcspt) in _SAS Viya Platform: Deployment Guide_. Verify that the requirements in [Cluster Ingress Requirements](https://documentation.sas.com/doc/en/itopscdc/v_039/itopssr/n1ika6zxghgsoqn1mq4bck9dx695.htm#p1eeoaos2nm7jsn1q3ymhui62j9v) have also been met.
| Kubernetes LoadBalancer services | Usage varies based on the underlying infrastructure. See the [endpointPublishingStrategy configuration parameter](https://docs.openshift.com/container-platform/4.7/networking/ingress-operator.html#nw-ingress-controller-configuration-parameters_configuring-ingress) section of the OpenShift Ingress Operator documentation for information about whether a LoadBalancer service is used by your cluster infrastructure. If you are using a load balancer, application gateway, or reverse proxy to function as the "front door" to your cluster, see [(Optional) Additional Requirements for Proxy Environments](https://documentation.sas.com/doc/en/itopscdc/v_039/itopssr/n1ika6zxghgsoqn1mq4bck9dx695.htm#n1l60mf73bkmqqn11tg9ba750v0f) for information about required configuration.|
| Node labels | One or more nodes should be fully dedicated to (that is, labeled and tainted for) the CAS server. This recommendation depends on whether your deployment uses the CAS server. SAS strongly recommends labeling at least one node for compute workloads. **Note**: If your deployment includes SAS Workload Management, you must label at least one node for the compute workload class. For more information, see [Plan the Workload Placement](https://documentation.sas.com/doc/en/itopscdc/v_039/dplyml0phy0dkr/p0om33z572ycnan1c1ecfwqntf24.htm) in _SAS Viya Platform: Deployment Guide_. |
| A certificate generator to enable TLS | The OpenSSL-based certificate generator supplied by SAS is used by default. You can instead use cert-manager if you install it in the cluster and take a few additional steps. **Note**: SAS Viya platform deployments on OCP 4.11 using cert-manager require a fix that is only available in cert-manager v1.10 and later. For more information about your options for TLS support and certificate management, see [TLS Requirements](https://documentation.sas.com/doc/en/itopscdc/v_039/itopssr/n18dxcft030ccfn1ws2mujww1fav.htm#p0bskqphz9np1ln14ejql9ush824). |
| cert-utils-operator | This operator from the Red Hat Communities is required in order to manage certificates for TLS support and to create keystores. For more information, see [https://github.com/redhat-cop/cert-utils-operator/blob/master/README.md](https://github.com/redhat-cop/cert-utils-operator/blob/master/README.md). |


### Deployment Operator
SAS provides an operator for deploying and updating SAS Viya. However, that operator is not a certified operator so it will not be found in the OperatorHub or in the Red Hat Marketplace. This is a point in time statement. SAS is actively working with Red Hat on certifying its operator. The SAS deployment is one of the largest ISV deployments on OpenShift. As of this document’s publication there are over 120 containers being certified. 

The SAS Viya Deployment Operator provides an automated method for deploying and updating your SAS Viya deployment. It runs in the OpenShift cluster and watches for declarative representations of SAS Viya deployments in the form of custom resources (CRs) of the type _SASDeployment_. The operator can watch for _SASDeployments_ in the namespace where it is deployed (namespace mode) or in all the namespaces in a cluster (cluster-wide mode). Using namespace mode means that the operator and the SAS Viya deployment are located in the same namespace. Operators in cluster-wide mode have their own namespaces.

**Note:** SAS recommends using only one mode of operator in a cluster. When a new _SASDeployment_ is created or an existing _SASDeployment_ is updated, the operator updates the SAS Viya deployment to match the state that is described in the CR. The operator determines the release of SAS Viya that it is working with. It also uses the appropriately versioned tools for that release while deploying and updating the SAS Viya software. Thus, a single instance of the operator can manage all SAS Viya deployments in the cluster.

The SAS Viya Deployment Operator requires updates occasionally but not at the same frequency as SAS Viya. Although there is a manual way to deploy SAS Viya, it HIGHLY recommended that deployments use the deployment operator. For specific information about using the deployment operator please follow instructions found in the [SAS documentation](https://documentation.sas.com/doc/en/itopscdc/v_039/itopscon/n1ob03chiyninhn19sm1ssfgss7w.htm).

### OpenShift GitOps
OpenShift Pipelines and OpenShift GitOps are two components of the Red Hat OpenShift Container Platform (OCP) that provide a turnkey CI/CD automation solution for  continuous integration (CI) and continuous delivery (CD) tasks. They are based on the Tekton and ArgoCD open-source projects.

OpenShift GitOps can be used to provide additional automation for a SAS Viya deployment; monitoring a Git repository for changes to the CR manifest and automatically syncing its’ contents to the cluster.  Pushing the CR manifest to the Git repository triggers a sync with OpenShift GitOps. The CR will be deployed to Kubernetes, which in turn triggers the Operator and the deployment to start.


