# Terraform ICP VMware HA

This Terraform example configurations uses the [VMware vSphere provider](https://www.terraform.io/docs/providers/vsphere/index.html) to provision virtual machines on VMware
and [TerraForm Module ICP Deploy](https://github.com/ibm-cloud-architecture/terraform-module-icp-deploy) to prepare VMs and deploy [IBM Cloud Private](https://www.ibm.com/cloud-computing/products/ibm-cloud-private/) on them.  This Terraform template automates best practices learned from installing ICP on VMware at numerous client sites in production.

This template provisions a non-HA cluster with ICP 3.1 community edition.

![](./static/icp_ce_vmware.png)


### Pre-requisites

* Working copy of [Terraform](https://www.terraform.io/intro/getting-started/install.html)
* The example assumes the VMs are provisioned from a template that has ssh public keys loaded in `${HOME}/.ssh/authorized_keys`. After VM creation, terraform will SSH into the VM to prepare and start installation of ICP using the SSH private key provided. If your VM template uses a different user from root, update the `ssh_user` section in [variables.tf](variables.tf#L264)
* The template is tested on VM templates based on **Ubuntu 16.04** and **RedHat 7.5**.

### VM Template image preparation

1. Create a VM image (RHEL or Ubuntu 16.04).
   * The automation will create an additional block device at unit number 1 (i.e. `/dev/sdb`) for local docker container images and attempt to configure Docker in direct-lvm or overlay2 mode, depending on your operating system. RHEL 7.4 and older will default to devicemapper, RHEL 7.5 and newer will default to Overlay2. All supported Ubuntu versions will default to Overlay2. You may pre-install docker, but it must be configured in direct-lvm or overlay2 mode. For direct-lvm the block device must be set as the second disk. The simplest way of getting this to work is to create a template with a single OS disk without installing docker and let the automation configure direct-lvm or overlay2 mode as appropriate.

   * Additional block devices are also mounted at various directories that hold ICP data depending on the node role. For instance, an additional block device at unit number 4 (i.e. `/dev/sdd`) with the possibility to use a different datastore than the other disks is mounted at `/var/lib/etcd`. The reason for this is to allow the allocation of a different tier datastore with potentially better performance for I/O disk intensive ICP components such as ETCD.

1. Ensure that the `ssh_user` can call `sudo` without password.

1. Install any other needed packages:
   1. Shared storage clients, e.g. if the shared storage is NFS for the registry and audit directories, install NFS clients.
   1. In environments with restricted outbound internet, the terraform automation requires and attempts to install `PyYAML`.  You can pre-install it into the template:
      - `yum install PyYAML` on RHEL
      - `apt install python-yaml` on Ubuntu
   1. Install VMware Tools or `open-vm-tools`
   1. For docker installation, you **MUST** provide ICP shipped Docker binary file. For Ubuntu based installations, you could install any additional pre-requisites on your template beforehand:
      - `apt install libtdl7 thin-provisioning-tools lvm2` on Ubuntu
   1. For RHEL templates, VMware guest customization requires `perl`.
   1. For RHEL templates, either disable the firewall (`systemctl disable firewalld`), or add `firewall_enabled: true` to `icp-config.yaml`.
   
1. Shutdown the VM Convert to a template, make note the name of the template.

### Using the Terraform templates

1. git clone or download the templates

1. Export your vSphere username and password into environment variables on the system you will run Terraform from.  This can be done via the following commands:
    - `export VSPHERE_USER={myusername}` replacing `{myusername}` with your vSphere username
    - `export VSPHERE_PASSWORD={mypassword}` replacing `{mypassword}` with your vSphere password

1. Create a `terraform.tfvars` file to reflect your environment.  Please see [variables.tf](variables.tf) and below tables for variable names and descriptions.  Here is an example `terraform.tfvars` file:

   ```
   #######################################
   ##### vSphere Access Credentials ######
   #######################################
   vsphere_server = "10.25.0.20"

   # Set username/password as environment variables VSPHERE_USER and VSPHERE_PASSWORD

   ##############################################
   ##### vSphere deployment specifications ######
   ##############################################
   # Following resources must exist in vSphere
   vsphere_datacenter = "DC1"
   vsphere_cluster = "Cluster1"
   vsphere_resource_pool = "ICP31_pool/terraform_icp_31"
   network_label = "LabPrivate"
   datastore = "LabDatastore"
   datastore_etcd = "Tier0_LabDatastore"
   template = "rhel_75_base_template"
   # Folder to provision the new VMs in, does not need to exist in vSphere
   folder = "terraform_icp_31"

   ##################################
   ##### ICP deployment details #####
   ##################################

   ##### ICP instance name #####
   # MUST consist of only lower case alphanumeric characters and '-'
   instance_name = "user1-icp-31"


   ##### Network #####
   staticipblock = "10.30.0.0/24"
   staticipblock_offset = 2
   gateway = "10.0.0.1"
   netmask = "16"
   dns_servers = [ "10.0.0.11", "10.0.0.12" ]

   ##### Local Terraform connectivity details #####
   ssh_user = "virtuser"
   ssh_password = "SuperPa88w0rd"

   ##### ICP installation method #####
   icp_inception_image = "ibmcom/icp-inception:3.1.0"
   docker_package_location="nfs:10.16.0.10:/icp/3.1/icp-docker-18.03.1_x86_64.bin"

   ##### ICP admin user password #####
   # Non default admin user password 'admin' recommended
   icppassword = "SuperPa88w0rd"

   ##### ICP Cluster Components #####
   master = {
       nodes = "1"
       vcpu = "8"
       memory = "16384"
       docker_disk_size = "250"
       thin_provisioned = "true"
       thin_provisioned_etcd = "false"
   }
   proxy = {
       nodes = "1"
       vcpu = "4"
       memory = "8192"
       thin_provisioned = "true"
   }
   worker = {
       nodes = "3"
       vcpu = "8"
       memory = "8192"
       thin_provisioned = "true"
   }
   management = {
       nodes = "1"
       vcpu = "4"
       memory = "16384"
       thin_provisioned = "true"
   }

   ##### NFS Server #####
   registry_mount_src = "10.0.0.5:/storage/user1-icp-31/registry"
   audit_mount_src = "10.0.0.5:/storage/user1-icp-31/audit"
   ```

1. Run `terraform init` to download dependencies (modules and plugins)

1. Run `terraform plan` to investigate deployment plan

1. Run `terraform apply` to start deployment

### ICP installation method

You should provide the ICP community edition Docker image in Docker Hub to get installed (plus the Docker installation binary that ships with it if you are installing on RHEL) in your `terraform.tfvars` file:

   ```
   ##### ICP installation method #####
   icp_inception_image = "ibmcom/icp-inception:3.1.0"
   docker_package_location="nfs:<nfs_server_ip_address>:<path_within_your_nfs_server>/icp-docker-18.03.1_x86_64.bin"
   ```

### Terraform configuration

#### vSphere variables

| name | required                        | value        |
|----------------|------------|--------------|
| `vsphere_server`   | yes          | IP or hostname of vSphere server |
| `vsphere_user`   | no, moved to environment variables          | Username for vSphere server |
| `vsphere_password`     | no, moved to environment variables          | Password for vSphere user     |
| `allow_unverified_ssl`   | no           | SSL certificate verification when connecting to vSphere, `true` by default. |
| `vsphere_datacenter` | yes         | Name of the vSphere datacenter to deploy VMs to |
| `vsphere_cluster` | yes         | Name of the vSphere cluster to deploy VMs to (must be under the vSphere datacenter) |
| `vsphere_resource_pool` | no         | Path of the Resource Pool to deploy VMs to (must be under the vSphere cluster), will be in the format like `/path/to/target`, by default will add VMs to root resource pool in the cluster |
| `network_label` | yes         | Network label to place all VMs on |
| `datastore` | yes         | Name of the datastore to place all disk images in. |
| `datastore_etcd` | no         | Name of the datastore to use for etcd storage. |
| `folder` | no         | Name of the VM folder to create where all created VMs are placed in, if not supplied, will place in root folder. |
| `template` | yes         | Name of the VM template to use to create all VM images |

#### Network variables

*Although the template defaults configure the cluster using DHCP addresses, it is strongly recommended to configure values for each of these variables so that the cluster is configured using static IP addresses.*

| name |  value                                  |
|------------|------------------|
| **VMware network** |  |
| `gateway` | Default gateway to configure for all VMs.  Leave blank to retrieve from DHCP. |
| `netmask` | Number of bits, in CIDR notation, of the subnet netmask  (e.g. 16).  Set to `0` to retrieve from DHCP. |
| `dns_servers` | List of DNS servers to configure in the VMs.  By default, uses Google public DNS (`8.8.8.8` and `8.8.4.4`).  Set to blank to retrieve from DHCP. |
| **VM IP Calculation** | |
| `staticipblock` |  Subnet to place all VMs in, in CIDR notation.  Ensure that the subnet has enough useable address for all created VMs.  For example, 192.168.0.0/24 will contain 256 addresses.  It is recommended to set this variable so that the cluster is configured with static IP addresses. The default is set to `0.0.0.0/0` to retrieve from DHCP (not recommended). |
| `staticipblock_offset`  | Specify the starting offset of the staticipblock to begin assigning IP addresses from.  e.g. with staticipblock 192.168.0.0/16, offset of 10 will cause IP address assignment to begin at 192.168.0.11. |
| **ICP Overlay network** |  |
| `network_cidr` | Container overlay network subnet; this subnet is internal to the cluster but should not overlap with other subnets in the environment.  Default is `192.168.0.0/16` |
| `service_network_cidr` | Service network subnet; this is internal to the cluster but should not overlap with other subnets in the environment.  Default is `10.10.10.0/24` |


#### ICP Installation variables

| name | default value                       | value        |
|----------------|------------|--------------|
| `ssh_user` | `root` | User that terraform will SSH as, must have passwordless sudo access. |
| `ssh_password` | &lt;none&gt; | Password for SSH user, needed to SSH in machines and add the Terraform-created SSH keys for password-less SSH installation |
| `docker_package_location` | &lt;none&gt; | location of ICP docker package,  e.g. `http://<myhost>/icp-docker-18.03_x86_64.bin` or `nfs:<myhost>:/path/to/icp-docker-18.03_x86_64.bin` |
| `icp_inception_image` | `ibmcom/icp-inception:3.1.0` | Name of the `icp-inception` image to use.  You may need to change it to install a different version of ICP. |
| `instance_name` | `icptest` | Name of the ICP installation. Will be used as basename for VMs. MUST consist of only lower case alphanumeric characters and '-' |
| `domain` | `<instance_name>.icp` | Specify domain name to be used for linux customization on the VMs. |

#### ICP Configuration Variables

The ICP configuration can further be customized by editing the [icp-config.yaml](icp-config.yaml) file and by following the [product documentation](https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.0/installing/config_yaml.html).  The Terraform templates here expose the key variables that have infrastructure dependencies.

| name | required                        | value        |
|----------------|------------|--------------|
| `registry_mount_src` | yes | Source of the shared storage for the registry directory `/var/lib/registry` |
| `registry_mount_type` | no | Type of mountpoint for the registry shared storage directory.  `nfs` by default. |
| `registry_mount_options` | no | Mount options to pass to the registry mountpoint.  `defaults` by default. |
| `audit_mount_src` | yes | Source of the shared storage for the audit directory `/var/lib/icp/audit` |
| `audit_mount_type` | no | Type of mountpoint for the audit shared storage directory.  `nfs` by default. |
| `audit_mount_options` | no | Mount options to pass to the audit mountpoint.  `defaults` by default. |
| `icppassword` | yes | Password for the initial admin user in ICP. `admin` by default |
| `disable_management_services` | no | List of management services to disable. Default: `["istio", "vulnerability-advisor", "storage-glusterfs", "storage-minio"]` |
