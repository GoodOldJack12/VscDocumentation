# Orchestration Using Terraform

HashiCorp Terraform <https://www.terraform.io/> is an
infrastructure as code tool (IaC). Users can deploy a data center
infrastructure using a declarative configuration language known as
HashiCorp Configuration Language (HCL), or using JSON.
Terraform has
some advantages over OpenStack Heat service. It is has a simple syntax, it
can provision virtual infrastructures across multiple cloud providers
(not only OpenStack) and it provides important features like network port forwarding rules (see
[floating-ip](configure_instances.md#floating-ip-addresses)).
Terraform is
currently one of the most popular infrastructure automation tools
available. VSC Cloud also provides some template examples that could be
used to deploy virtual infrastructures within VSC Tier-1 Cloud in an
automated way
(<https://github.com/hpcugent/openstack-templates/tree/master/terraform>).

Terraform
client is available for different Operating Systems like Windows, Linux
or macOS (<https://www.terraform.io/downloads>) but it is also available
from UGent login node _login.hpc.ugent.be_.

## Create application credentials for Terraform

Terraform
uses OpenStack application credentials to authenticate to VSC Cloud
Tier-1 public API. It is a good practice to generate a new application
credential just to be used with Terraformframework. The process is the same
described in section
[application credentials](access.md#application-credentials).

:::{note}
Make sure you download the new application credential as yaml file
instead of openRC.
:::

At this point you should have a _clouds.yaml_ text file with these lines:

```bash
# This is a clouds.yaml file, which can be used by OpenStack tools as a source
# of configuration on how to connect to a cloud. If this is your only cloud,
# just put this file in ~/.config/openstack/clouds.yaml and tools like
# python-openstackclient will just work with no further config. (You will need
# to add your password to the auth section)
# If you have more than one cloud account, add the cloud entry to the clouds
# section of your existing file and you can refer to them by name with
# OS_CLOUD=openstack or --os-cloud=openstack
clouds:
  openstack:
    
    auth:
      
      auth_url: https://cloud.vscentrum.be:13000
      
      application_credential_id: "xxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
      application_credential_secret: "xxxxxxxxxxxxxxxxxxxxxxxxxxx"
    
      
        
    region_name: "regionOne"
        
      
    interface: "public"
    identity_api_version: 3
    auth_type: "v3applicationcredential"

```

As the file comments state, you should copy the current clouds.yaml to
your VSC login node **$HOME** _login.hpc.ugent.be:
~/.config/openstack/clouds.yaml_, or locally if you have installed
Terraform in
your own laptop or computer. Terraform will use this file to authenticate to
OpenStack API automatically.

## Getting Terraform examples

You can connect to UGent login node `login.hpc.ugent.be` to use
terraform. Login to the login node with your VSC account first:

```shell
ssh -A vscxxxxx@login.hpc.ugent.be
```

If this is the first time using Terraform, download the VSC Terraform
examples from github from
<https://github.com/hpcugent/openstack-templates>:

```shell
git clone https://github.com/hpcugent/openstack-templates
```

Make sure you have *`~/.config/openstack/clouds.yaml`* available from
the login node (see previous [section](#create-application-credentials-for-terraform)).

:::{warning}
Do not share your application's credential file `clouds.yaml` or put this
file in a public place.
:::

```shell
chmod 600 ~/.config/openstack/clouds.yaml
```

## Generate Terraform template variables

Terraform
requires some variables to know which resources are available from the
cloud provider for the user or project. You do not have to include these
variables manually, we have included a script to gather these variable
IDs automatically. From the Terraform directory cloned from git in the
previous step (see previous
[section](#getting-terraform-examples)), go to the scripts directory:

```shell
cd ~/openstack-templates/terraform/scripts
```

And now run the script (usually you only have to run this script once).

```shell
./modify_variable.sh
```

This step will take some seconds. The script will contact the VSC
OpenStack public API to gather all the resources available for your
project and fetch all the resource's IDs. Usually you only have to run
this script once, unless something was changed/updated for your
project's resources (like a new network or floating IPs) or if you want
to deploy a new Terraform template from scratch.

You will see some messages like this (IDs and IPs may change depending
on your project's resources).

```console
Image id: 3313f95f-634c-4127-b65f-cde6d6f797cf. (Image name: Rocky-9)
Flavor name: CPUv1.medium.
Image id: 0684c8d3-15c7-4a6e-8fd6-3850e62f3ad4. (Image name: Windows10)
Flavor name: CPUv1.medium.
Root FS volume size based on flavor disk size: 30.
VM network id: 4d72c0ec-c000-429e-89c6-8c3607a28b3d.
VM subnet id: f0bc8307-568f-457d-adff-219005a054e2.
NFS network id: 119d8617-4000-47c0-9c6e-589b3afce144.
NFS subnet id: e4e07edd-39cf-42ea-9fe4-5bf2891d2592.
VSC network id: f6eba915-06ad-4e50-bc4b-1538cdc39296.
VSC subnet id: b5ed8dc2-6d3f-42d4-87f8-3ffee19c1a9c.
Using first ssh access key "ssh-ed25519".
Using floating ip id: 64f2705c-43ec-4bdf-864e-d18fee013e3f. (floating ip: 193.190.80.3)
Using VSC floating ip: 172.24.49.7.
Modifying ../environment/main.tf file.
Using ssh forwarded ports: 59604 58458 55222 51217.
Using http forwarded port: 56871.
Using rdp forwarded port: 53557.
Modifying provider.tf files.

SSH commands for VMs access:

(MyVM)           ssh -p 59604 <user>@193.190.80.3
(MyVM-nginx)     ssh -p 58458 <user>@193.190.80.3
(MyVM-vsc_net)   ssh -p 55222 <user>@193.190.80.3
(MyVM-nfs_share) ssh -p 51217 <user>@193.190.80.3
(MyVM-windows)   xfreerdp /u:admin /port:53557 /v:193.190.80.3
```

After this step your Terraform templates will be ready to be deployed.

:::{note}
Please note that the script shows you the ssh command to connect to each
VM after instantiation (including the port which is generated
automatically by the script). You can copy this list or you can review
it later. Also note that you should use a valid user to connect to the
VM, for instance for CentOS images is *centos*, for Ubuntu images is
*ubuntu* and so on. You can also try to connect as *root* user, in that
case the system will show you a message with the user that you should
use.
:::

## Modify default Terraform modules

In [section](#getting-terraform-examples) we have downloaded the
Terraform module examples from the VSC repository. If you deploy these
modules as it is it will deploy several VM examples by default such as:

1.  **MyVM:** simple VM with 20Gb persistent volume and ssh access with port
    forwarding.

2.  **MyVM-nginx:** Like previous example but with an ansible playbook to install nginx
    and access to port 80 besides ssh.

3.  **MyVM-vsc_net:** Similar to the first example but also includes a VSC network
    interface (only available for some projects).

4.  **MyVM-nfs_share:** Similar to the first example but it creates a NFS share filesystem
    and it mounts it during instantiation (only available for some
    projects).

5.  **MyVM-windows:** Windows VM with Remote Desktop access.

But usually you do not want to deploy all these examples, you can just
keep the required module and comment out the rest. You can do this from
environment directory:

```shell
cd ~/openstack-templates/terraform/environment
```

And edit *`main.tf`* Terraform file with any text editor like vim or
nano. If you want to deploy just the simple VM (first example) only keep
these lines (remenber variable IDs may change depending on your
project's resources):

```
module "vm_with_pf_rules_with_ssh_access" {
  source   = "../modules/vm_with_pf_rules_with_ssh_access"

  vm_name              = "MyVM"
  floating_ip_id       = "64f2705c-43ec-4bdf-864e-d18fee013e3f"
  vm_network_id        = "4d72c0ec-c000-429e-89c6-8c3607a28b3d"
  vm_subnet_id         = "f0bc8307-568f-457d-adff-219005a054e2"
  access_key           = "ssh-ed25519 AAAAC3Nz_A02TxLd9 lsimngar_varolaptop"
  image_id             = "d00d6dfd-998c-4bcb-bbf4-496ef84d5b64"
  flavor_name          = "CPUv1.small"
  ssh_forwarded_port   = "56469"
  root_fs_volume_size  = "20"
}
```

And remove or comment out the rest of the lines. In the previous example
Terraform will deploy a simple VM and use 20Gb for a persistent volume
and port 56469 to connect via ssh (it also creates all required security
groups).

## Deploy Terraform templates

If you have followed the previous steps now you can init and deploy your
infrastucture to Tier-1 VSC cloud.

You have to inititate Terraform first, if you didnt have deployed any
template yet do this just once.

Move to environment directory first:

```shell
cd ~/openstack-templates/terraform/environment
```

This command performs several different initialization steps in order to
prepare the current working directory for use with Terraform:

```shell
terraform init
```

Now you can check and review your Terraform plan, from the same
directory:

```shell
terraform plan
```

You will see a list of the resources required to deploy your
infrastructure, Terraform also checks if there is any systax error in
your templates. Your infrastructure is not deployed yet, review the plan
and then just deploy it to VSC Tier-1 Cloud running:

```shell
terraform apply
```

Terraform will show your plan again and you will see this message:

```console
..
..
Do you want to perform these actions?
Terraform will perform the actions described above.
Only ’yes’ will be accepted to approve.
Enter a value:
```

Type **yes** and press enter and wait a few seconds or minutes. If
everything is correct and if you have enough quota Terraform will show
you a message after creating all the required resources.

```console
..
..
module.vm_with_pf_rules_with_ssh_access.openstack_compute_instance_v2.instance_01:
Still creating... [1m30s elapsed]
module.vm_with_pf_rules_with_ssh_access.openstack_compute_instance_v2.instance_01:
Creation complete after 1m35s [id=88c7d037-5c44-45b7-acce-f5e4e58b1c35]
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.
```

Your cloud infrastructure is ready to be used.

:::{tip}
It is important to keep a backup of your terraform directory, specially
all the files within the environment directory:
`~/openstack-templates/terraform/environment`
:::


Terraform generates several files in this directory to keep track of any
change in your infrastructure. If for some reason you lost or remove
these files you will not able to modify or change the current Terraform
plan (only directly from OpenStack).


:::{warning}
It is possible to execute `./modify_variable.sh` script several times,
(for instance, if you have changed the variables within
`modify_variable.config` file). This will update your current Terraform
deployment. Any Terraform variables change may affect to your current
running VM(s) as well, `terrafrom apply` will inform you about this and
it will request confirmation. The external ports for port forwarding
are generated randomly when you run `modify_variable.sh` by default, but
you can keep the current port forwarding rules if you add `-k` option:
`./modify_variable.sh -k`
:::

You can also modify and add more resources for the current templates.
This task is out of the scope of this document, please refer to official
Terraform documentation to add you own changes
<https://www.terraform.io/docs> or ask to VSC Cloud admins via email at
<cloud@vscentrum.be>.
