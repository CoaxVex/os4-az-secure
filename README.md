# os4_az_secure Ansible role

This Ansible role will secure (disconnect from the internet) an OpenShift 4 installation on Azure. It is meant to be applied on a newly installed cluster.

After applying this role, the following be true:

* The Kubernetes API is only reachable from the Azure vnet, or from a vnet peered with the OpenShift vnet.
* Routes are by default only reachable from the Azure vnet, or from a vnet peered with the OpenShift vnet.
* Optionally, routes labeled with "external": "" are reachable from the internet.

Virtual machines will still have access to the Internet. (Look into [performing a disconnected install](https://blog.openshift.com/openshift-4-2-disconnected-install/) instead if this is what you need)

## Important notes

### Before you begin

### Access to Azure
This role uses azure_rm_xxx modules. It asumes you have a working azure profile set up for your user. The easiest way to achieve this, is to install the az commandline and perform an 'az login'. 
You will also need the 'ansible[azure]' pip module. (pip install 'ansible[azure]').

### Access to OpenShift
This role assumes that you have a KUBECONFIG set that points to the **internal** kubernetes API. Be sure to perform an 'oc login' on the internal URL, and that you have administrative access on the cluster before you begin.

### Default routes become unreachable
The default ingresscontroller is moved to different nodes and its DNS entry (\*.apps) is updated to point to an internal IP address. This means that there will be a short period where the oauth service is not reachable. You will not be able to authenticate with oauth until the infra nodes are available, the router pods are functioning, and the DNS is updated to reflect the internal loadbalancer IP.

If something goes wrong, you can authenticate with the X509 certificates found in the kubeconfig created by the OpenShift installer. To do so, replace the URL to the API with the private IP address, and issue the --insecure-skip-tls-verify option when using the oc commandline.

## Variables

The following variables are set in defaults/main.yml:

```
os4_az_secure_debug: false
os4_az_secure_cluster_name: ""
os4_az_secure_domain_name: ""
os4_az_secure_parent_domain_resource_group: ""
os4_az_secure_router_subnet_cidr: "10.3.64.0/19"
os4_az_secure_deploy_public_ingresscontroller: false
os4_az_secure_public_ingresscontroller_domain: ""
os4_az_secure_machinesets:
  - role: router
    replicas: 1
    vmSize: Standard_DS2_v2
    osDisk_diskSizeGB: 64
    zone: 1
    subnet_name: router
  - role: router
    replicas: 1
    vmSize: Standard_DS2_v2
    osDisk_diskSizeGB: 64
    zone: 2
    subnet_name: router
  - role: infra
    replicas: 1
    vmSize: Standard_DS2_v2
    osDisk_diskSizeGB: 64
    zone: 1
    public_ip: true
  - role: infra
    replicas: 1
    vmSize: Standard_DS2_v2
    osDisk_diskSizeGB: 64
    zone: 2
    public_ip: true
```

* os4_az_secure_debug: Enable some debug statements
* os4_az_secure_cluster_name: This needs to be set to the clustername you have chosen at installation time, with the 5-character generated suffix append. (cluster01-m5m6h)
* os4_az_secure_parent_domain_resource_group: The resource group on azure which has the parent DNS zone.
* os4_az_secure_router_subnet_cidr: A new subnet will be created for router pods. Choose a cidr that does not conflict with any of your other virtual networks or subnets.
* os4_az_secure_deploy_public_ingresscontroller: Set to true to enable a public ingresscontroller. Will be deployed on the router nodes and will only configure routes with the 'external: ""' label.
* os4_az_secure_public_ingresscontroller_domain: Set the default domain for public routes. (Should be different from the default)
* os4_az_secure_machinesets: A list of machinesets to create. 
  * role: 'router' or 'infra' are supported here.
  * replicas: number of replicas
  * vmSize: Choose your vmSize. (https://docs.microsoft.com/en-us/azure/virtual-machines/windows/sizes-general)
  * osDisk_diskSizeGB: disk size in GB.
  * zone: The zone in which to deploy vm's. You probably want to spread vm's across two or three zones.
  * subnet_name: setting this to 'router' will deploy the virtual machie in the router subnet. If undefined, machines will be added to the "worker" subnet. 
  * public_ip: Set to true to give each machine a public IP address. This does not mean they are reachable from the internet, as the default network security group will block all access. Infra nodes need to be given a public IP address for them to have internet access, as they are added to an internal load balancer.

If you do not need external routers, be sure to override the os4_az_secure_machinesets variable to only include infra nodes. You may also choose to use a beefier vmSize if you are going to be running other components such as monitoring or logging on the infra nodes.

## Example playbook

```
- hosts: localhost
  become: false
  connection: local
  roles:
    - role: os4-az-secure
      vars:
        - os4_az_secure_cluster_name: dev01-6vdgm
          os4_az_secure_domain_name: dev01.clusters.vargen.io
          os4_az_secure_parent_domain_resource_group: focus-os4-common
          os4_az_secure_deploy_public_ingresscontroller: true
```

It will take a couple of minutes before the machines have been provisioned on Azure and the nodes have registered.

```
oc get machines -n openshift-machine-api
oc get nodes
oc get pods -n openshift-ingress
```
