
# SAS Viya on Red Hat OpenShift 
Authors: Patrick Farley and Hans-Joachim Edert

In this blog, we will provide some basic technical information about SAS, as well as a reference architecture for SAS Viya on Red Hat OpenShift.

This reference architecture will show how SAS Viya is containerized to run with Kubernetes on the Red Hat OpenShift Container Platform. As SAS rolled out Viya for Kubernetes, they made a specific design decision to take advantage of core services provided by that specific Kubernetes distribution. For example, on Azure AKS, SAS takes advantage of Active Directory for RBAC controls and Azure Security Vault for key management, among other things. SAS followed the same pattern when moving to AWS EKS and Google GKE. For OpenShift deployments, SAS takes advantage of the OpenShift Ingress Operator, cert manager, OpenShift GitOps for deployment. They also take advantage of OpenShift Routes and SCCs (Security Context Constraints) as part of their deployments.  

## SAS Viya on OpenShift Reference Architecture
SAS Viya is a platform for Analytics. Moving SAS Viya to OpenShift gives Viya unprecedented scalability that was unavailable in the legacy product. SAS takes advantage of the scalability by breaking Viya down into different workload types and then assigning each workload to a class of nodes. This ensures that the proper resources are available to specific workloads.

<img width="1089" alt="image" src="https://user-images.githubusercontent.com/48925593/233742961-78c57b46-be7d-4843-8511-5fcc5799b7c9.png">

_Figure 1_

In the current iteration of SAS Viya on OpenShift, SAS only supports VMware vSphere as the deployment platform. At the time of this document, BareMetal is on the roadmap as an alternative for on-prem deployments. This is not a statement of will or will not work. It is what SAS has tested and is willing to support.  

Although SAS Viya on OpenShift is currently fully supported only on VMware vSphere, it is possible to deploy it on different infrastructure providers, such as Azure, AWS and BareMetal under the [“SAS Support for Alternative Kubernetes Distributions”](https://support.sas.com/en/technical-support/services-policies.html#k8s) policy.



## Core Platform
VMware vSphere 7.0.1 is the virtual machine platform that is currently supported. The details of VMware configuration will not be covered in this document with the assumption that VMware vSphere is well known in most environments. There are discussions about other platforms like Nutanix and others. However, this is only a roadmap topic for now. 

On top of VMware vSphere 7.0.1 is Red Hat OpenShift 4.10 - 4.12. The specific OpenShift requirements to support SAS Viya deployment, are:

- **OpenShift Ingress Operator**

   SAS has specific requirements for forwarding cookies during transaction execution. As such, they used special techniques using the HA-Proxy to make that happen. So, in this iteration only the OpenShift Ingress Operator is supported. 

- **cert-utils-operator**

   SAS requires the use of this operator to manage certificates for TLS support and create keystores.

- **cert-manager**

   SAS relies on cert-manager to generate certificates for TLS. However, they do support OpenSSL-based cert managers as well. 

- **Security Context Constraints (SCCs)**

   SAS makes use of SCCs with OpenShift. They require multiple custom SCCs to support SAS Viya Services. The SAS documentation provides important information about the required SCCs, which helps to understand their use in your environment and to address any concerns. 

- **OpenShift Routes**

   SAS prefers the use of native features with the environments with their products, so they take advantage of OpenShift routes. During deployment, yaml setting changes for the cluster ingress controller are provided to apply the proper configuration for OpenShift Routes. 


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

The SAS Viya Deployment Operator provides an automated method for deploying and updating your SAS Viya deployment. It runs in the OpenShift cluster and watches for declarative representations of SAS Viya deployments in the form of custom resources (CRs) of the type SASDeployment. The operator can watch for SASDeployments in the namespace where it is deployed (namespace mode) or in all the namespaces in a cluster (cluster-wide mode). Using namespace mode means that the operator and the SAS Viya deployment are located in the same namespace. Operators in cluster-wide mode have their own namespaces.

**Note:** SAS recommends using only one mode of operator in a cluster. When a new SASDeployment is created or an existing SASDeployment is updated, the operator updates the SAS Viya deployment to match the state that is described in the CR. The operator determines the release of SAS Viya that it is working with. It also uses the appropriately versioned tools for that release while deploying and updating the SAS Viya software. Thus, a single instance of the operator can manage all SAS Viya deployments in the cluster.

The SAS Viya Deployment Operator requires updates occasionally but not at the same frequency as SAS Viya. Although there is a manual way to deploy SAS Viya, it HIGHLY recommended that deployments use the deployment operator. For specific information about using the deployment operator please follow instructions found in the [SAS documentation](https://documentation.sas.com/doc/en/itopscdc/v_039/itopscon/n1ob03chiyninhn19sm1ssfgss7w.htm).




