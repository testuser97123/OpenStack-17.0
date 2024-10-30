# Standalone Deployment Guide

## Red Hat OpenStack Platform 17.0

Creating an all-in-one OpenStack cloud for test and proof-of-concept environments

## Abstract
Install, configure, and deploy Red Hat OpenStack Platform in a test environment with the Red Hat OpenStack Platform standalone environment. Use this guide to deploy a simple single-node OpenStack cloud

# Chapter 1. All-in-one Red Hat OpenStack Platform installation
The all-in-one installation method uses TripleO to deploy Red Hat OpenStack Platform and related services with a simple, single-node environment. Use this installation to enable proof-of-concept, development, and test deployments on a single node with limited or no follow-up operations. 

1.1. Prerequisites

-    Your system must have a Red Hat Enterprise Linux 9.0 base operating system installed.
-    Your system must have two network interfaces so that internet connectivity is not disrupted while TripleO configures the second interface.
-    Your system must have 4 CPUs, 8GB RAM, and 30GB disk space. 

### Example network configuration

-    Interface **eth0** assigned to the default network **192.168.122.0/24**. Use this interface for general connectivity. This interface must have internet access.
-    Interface **eth1** assigned to the management network **192.168.25.0/24**. TripleO uses this interface for the OpenStack services. 

# Chapter 2. Overview of the all-in-one Red Hat OpenStack Platform environment

 This section contains information about installing, configuring, and deploying a simple, single-node Red Hat OpenStack Platform environment. In this scenario, there is no pre-existing undercloud dependency. Instead, the installer runs an inline heat-all instance to bootstrap the deployment process and convert the selected heat templates into Ansible playbooks that you can execute on a local machine.

Use the all-in-one installation for basic testing and development. The all-in-one installation is a good starting point and test environment for Red Hat OpenStack Platform, but if you want to perform complex operations, you must deploy a production-level scaled cloud.

### Workflow

To install, configure, and deploy a simple, single-node Red Hat OpenStack Platform environment, complete the tasks in the following basic workflow:

1.    Prepare your environment.
2.    Install packages for the all-in-one environment.
3.    Configure the all-in-one environment.
4.    Deploy the all-in-one environment. 

### Benefits of the all-in-one installation

-    Composable services.
-    Pre-defined roles.
-    Condensed single-node environment.
-    Playbooks that you can use to run the small-footprint installer in a container and generate Ansible playbooks. 

# Chapter 3. Installing the all-in-one Red Hat OpenStack Platform environment

 Before you can begin configuring, deploying, and testing your all-in-one environment, you must configure a non-root user and install the necessary packages and dependencies:

Create a non-root user on the all-in-one host:

    [root@all-in-one]# useradd stack

Set the password for the stack user:

    [root@all-in-one]# passwd stack

Disable password requirements when using sudo as the stack user:

    [root@all-in-one]# echo "stack ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/stack
    [root@all-in-one]# chmod 0440 /etc/sudoers.d/stack

Log in as the non-root user on the all-in-one host:

    [stack@all-in-one]$ ssh stack@<all-in-one>

Register the machine with Red Hat Subscription Manager. Enter your Red Hat subscription credentials at the prompt:

    [stack@all-in-one]$ sudo subscription-manager register

Attach your Red Hat subscription to the entitlement server:

    [stack@all-in-one]$ sudo subscription-manager attach --auto

[!NOTE] 
> The --auto option might not subscribe you to the correct subscription pool. Ensure that you subscribe to the correct pool, otherwise you might not be able to enable all of the repositories necessary for this installation. Use the subscription-manager list --all --available command to identify the correct pool ID. 

Lock the undercloud to Red Hat Enterprise Linux 9.0:

    [stack@all-in-one]$ sudo subscription-manager release --set=9.0
    
Run the following commands to install dnf-utils, disable all default repositories, and then enable the necessary repositories:

    [stack@all-in-one]$ sudo dnf install -y dnf-utils
    [stack@all-in-one]$ sudo subscription-manager repos --disable=*
    [stack@all-in-one]$ sudo subscription-manager repos \
    --enable=rhel-9-for-x86_64-baseos-eus-rpms \
    --enable=rhel-9-for-x86_64-appstream-eus-rpms \
    --enable=rhel-9-for-x86_64-highavailability-eus-rpms \
    --enable=openstack-17-for-rhel-9-x86_64-rpms \
    --enable=fast-datapath-for-rhel-9-x86_64-rpms

[!NOTE] 
> The all-in-one environment is a Technology Preview feature in Red Hat OpenStack Platform 17.0.


Update the base operating system and reboot the system:

    [stack@all-in-one]$ sudo dnf update
    [stack@all-in-one]$ sudo reboot

Log back in to the host after the reboot.

Install the TripleO command line interface (CLI):

    [stack@all-in-one]$ sudo dnf install -y python3-tripleoclient

# Chapter 4. Configuring the all-in-one Red Hat OpenStack Platform environment

 To create an all-in-one Red Hat OpenStack Platform environment, include four environment files with the openstack tripleo deploy command. You must create two of the configuration files, shown below:

- $HOME/containers-prepare-parameters.yaml
- $HOME/standalone_parameters.yaml 

Two environment files are provided for you in the /usr/share/openstack-tripleo-heat-templates/ directory:

- /usr/share/openstack-tripleo-heat-templates/environments/standalone/standalone-tripleo.yaml
- /usr/share/openstack-tripleo-heat-templates/roles/Standalone.yaml 

You can customize the all-in-one environment for development or testing. Include modified values for the parameters in either the standalone-tripleo.yaml or Standalone.yaml configuration files in a newly created yaml file in your home directory. Include this file in the openstack tripleo deploy command. 

#### 4.1. Generating YAML files for the all-in-one Red Hat OpenStack Platform (RHOSP) environment

To generate the containers-prepare-parameters.yaml and standalone_parameters.yaml files, complete the following steps:

Generate the containers-prepare-parameters.yaml file that contains the default ContainerImagePrepare parameters:

    [stack@all-in-one]$ openstack tripleo container image prepare default --output-env-file $HOME/containers-prepare-parameters.yaml

Edit the containers-prepare-parameters.yaml file and include your Red Hat credentials in the ContainerImageRegistryCredentials parameter so that the deployment process can authenticate with registry.redhat.io and pull container images successfully:

    parameter_defaults:
      ContainerImagePrepare:
      ...
      ContainerImageRegistryCredentials:
        registry.redhat.io:
          <USERNAME>: "<PASSWORD>"

#### 3. Set the ContainerImageRegistryLogin parameter to true in the containers-prepare-parameters.yaml:

    parameter_defaults:
      ContainerImagePrepare:
      ...
      ContainerImageRegistryCredentials:
        registry.redhat.io:
          <USERNAME>: "<PASSWORD>"
      ContainerImageRegistryLogin: true
    
If you want to use the all-in-one host as the container registry, omit this parameter and include --local-push-destination in the openstack tripleo container image prepare command. 

Create the $HOME/standalone_parameters.yaml file and configure basic parameters for your all-in-one RHOSP environment, including network configuration and some deployment options. In this example, network interface eth1 is the interface on the management network that you use to deploy RHOSP. eth1 has the IP address 192.168.25.2:

    [stack@all-in-one]$ export IP=192.168.25.2
    [stack@all-in-one]$ export VIP=192.168.25.3
    [stack@all-in-one]$ export NETMASK=24
    [stack@all-in-one]$ export INTERFACE=eth1
    [stack@all-in-one]$ export DNS1=1.1.1.1
    [stack@all-in-one]$ export DNS2=8.8.8.8
    
    [stack@all-in-one]$ cat <<EOF > $HOME/standalone_parameters.yaml
    parameter_defaults:
      CloudName: $IP
      CloudDomain: localdomain
      ControlPlaneStaticRoutes: []
      Debug: true
      DeploymentUser: $USER
      KernelIpNonLocalBind: 1
      DockerInsecureRegistryAddress:
        - $IP:8787
      NeutronPublicInterface: $INTERFACE
      NeutronDnsDomain: localdomain
      NeutronBridgeMappings: datacentre:br-ctlplane
      NeutronPhysicalBridge: br-ctlplane
      StandaloneEnableRoutedNetworks: false
      StandaloneHomeDir: $HOME
      StandaloneLocalMtu: 1500
    EOF
    
If you use only a single network interface, you must define the default route:
    
    ControlPlaneStaticRoutes:
      - ip_netmask: 0.0.0.0/0
        next_hop: $GATEWAY
        default: true

If you have an internal time source, or if your environment blocks access to external time sources, use the NtpServer parameter to define the time source that you want to use:

    parameter_defaults:
      NtpServer:
        - clock.example.com

If you want to use the all-in-one RHOSP installation in a virtual environment, you must define the virtualization type with the NovaComputeLibvirtType parameter:

    parameter_defaults:
      NovaComputeLibvirtType: qemu

The Load-balancing service (octavia) does not require that you configure SSH. However, if you want SSH access to the load-balancing instances (amphorae), add the OctaviaAmphoraSshKeyFile parameter with a value of the absolute path to your public key file for the stack user: OctaviaAmphoraSshKeyFile: "/home/stack/.ssh/id_rsa.pub" 

# Chapter 5. Deploying the all-in-one Red Hat OpenStack Platform environment

 To deploy your all-in-one environment, complete the following steps:

    Log in to registry.redhat.io with your Red Hat credentials:

    [stack@all-in-one]$ sudo podman login registry.redhat.io

    Export the environment variables that the deployment command uses. In this example, deploy the all-in-one environment with the eth1 interface that has the IP addresses 192.168.25.2 and 192.168.25.3 on the management network:

    [stack@all-in-one]$ export IP=192.168.25.2
    [stack@all-in-one]$ export VIP=192.168.25.3
    [stack@all-in-one]$ export NETMASK=24
    [stack@all-in-one]$ export INTERFACE=eth1

    Set the hostname. If the node is using localhost.localdomain, the deployment will fail.

    [stack@all-in-one]$ hostnamectl set-hostname all-in-one.example.net
    [stack@all-in-one]$ hostnamectl set-hostname all-in-one.example.net --transient

    Run the deployment command. Ensure that you include all .yaml files relevant to your environment:

    [stack@all-in-one]$ sudo openstack tripleo deploy \
      --templates \
      --local-ip=$IP/$NETMASK \
      --control-virtual-ip=$VIP \
      -e /usr/share/openstack-tripleo-heat-templates/environments/standalone/standalone-tripleo.yaml \
      -r /usr/share/openstack-tripleo-heat-templates/roles/Standalone.yaml \
      -e $HOME/containers-prepare-parameters.yaml \
      -e $HOME/standalone_parameters.yaml \
      --output-dir $HOME \
      --standalone

After a successful deployment, you can use the clouds.yaml configuration file in the /home/$USER/.config/openstack directory to query and verify the OpenStack services:
    
    [stack@all-in-one]$ export OS_CLOUD=standalone
    [stack@all-in-one]$ openstack endpoint list
    
To access the dashboard, go to to http://192.168.25.2/dashboard and use the default username admin and the password from the $HOME/config/openstack/clouds.yaml file:

    [stack@all-in-one]$ cat $HOME/.config/openstack/clouds.yaml | grep password:

# Chapter 6. Creating Ansible playbooks with the all-in-one Red Hat OpenStack Platform environment


The deployment command applies Ansible playbooks to the environment automatically. However, you can modify the deployment command to generate Ansible playbooks without applying them to the deployment, and run the playbooks later.

Include the --output-only option in the deploy command to generate the standalone-ansible-XXXXX directory. This directory contains a set of Ansible playbooks that you can run on other hosts.

To generate the Ansible playbook directory, run the deploy command with the option --output-only:

    [stack@all-in-one]$ sudo openstack tripleo deploy \
      --templates \
      --local-ip=$IP/$NETMASK \
      -e /usr/share/openstack-tripleo-heat-templates/environments/standalone/standalone-tripleo.yaml \
      -r /usr/share/openstack-tripleo-heat-templates/roles/Standalone.yaml \
      -e $HOME/containers-prepare-parameters.yaml \
      -e $HOME/standalone_parameters.yaml \
      --output-dir $HOME \
      --standalone \
      --output-only

To run the Ansible playbooks, run the ansible-playbook command, and include the inventory.yaml file and the deploy_steps_playbook.yaml file:

    [stack@all-in-one]$ cd standalone-ansible-XXXXX
    [stack@all-in-one]$ sudo ansible-playbook -i inventory.yaml deploy_steps_playbook.yaml

# Chapter 7. Examples

Use the following examples to understand how to launch a compute instance post-deployment with various network configurations. 

#### 7.1. Example 1: Launching an instance with one NIC on the project and provider networks

Use this example to understand how to launch an instance with the private project network and the provider network after you deploy the all-in-one Red Hat OpenStack Platform environment. This example is based on a single NIC configuration and requires at least three IP addresses.
Prerequisites

To complete this example successfully, you must have the following IP addresses available in your environment:

-    One IP address for the OpenStack services.
-    One IP address for the virtual router to provide connectivity to the project network. This IP address is assigned automatically in this example.
-    At least one IP address for floating IPs on the provider network. 

### Procedure

Create configuration helper variables:

    # standalone with project networking and provider networking
    export OS_CLOUD=standalone
    export GATEWAY=192.168.25.1
    export STANDALONE_HOST=192.168.25.2
    export PUBLIC_NETWORK_CIDR=192.168.25.0/24
    export PRIVATE_NETWORK_CIDR=192.168.100.0/24
    export PUBLIC_NET_START=192.168.25.4
    export PUBLIC_NET_END=192.168.25.15
    export DNS_SERVER=1.1.1.1

Create a basic flavor:

    $ openstack flavor create --ram 512 --disk 1 --vcpu 1 --public tiny

Download CirrOS and create an OpenStack image:

    $ wget https://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
    $ openstack image create cirros --container-format bare --disk-format qcow2 --public --file cirros-0.4.0-x86_64-disk.img

Configure SSH:

    $ ssh-keygen -m PEM -t rsa -b 2048 -f ~/.ssh/id_rsa_pem
    $ openstack keypair create --public-key ~/.ssh/id_rsa_pem.pub default

Create a simple network security group:

    $ openstack security group create basic

Configure the new network security group:

Enable SSH:

    $ openstack security group rule create basic --protocol tcp --dst-port 22:22 --remote-ip 0.0.0.0/0

Enable ping:

    $ openstack security group rule create --protocol icmp basic

Enable DNS:

    $ openstack security group rule create --protocol udp --dst-port 53:53 basic

Create Neutron networks:

    $ openstack network create --external --provider-physical-network datacentre --provider-network-type flat public
    $ openstack network create --internal private
    $ openstack subnet create public-net \
        --subnet-range $PUBLIC_NETWORK_CIDR \
        --no-dhcp \
        --gateway $GATEWAY \
        --allocation-pool start=$PUBLIC_NET_START,end=$PUBLIC_NET_END \
        --network public
    $ openstack subnet create private-net \
        --subnet-range $PRIVATE_NETWORK_CIDR \
        --network private

Create a virtual router:

    # NOTE: In this case an IP will be automatically assigned
    # from the allocation pool for the subnet.
    $ openstack router create vrouter
    $ openstack router set vrouter --external-gateway public
    $ openstack router add subnet vrouter private-net

Create a floating IP:

    $ openstack floating ip create public

Launch the instance:

    $ openstack server create --flavor tiny --image cirros --key-name default --network private --security-group basic myserver

Assign the floating IP:

    $ openstack server add floating ip myserver <FLOATING_IP>

Replace FLOATING_IP with the address of the floating IP that you create in a previous step.

Test SSH:

    ssh cirros@<FLOATING_IP>

Replace FLOATING_IP with the address of the floating IP that you create in a previous step. 


#### 7.2. Example 2: Launching an instance with one NIC on the provider network


Use this example to understand how to launch an instance with the provider network after you deploy the all-in-one Red Hat OpenStack Platform environment. This example is based on a single NIC configuration and requires at least four IP addresses.
Prerequisites

To complete this example successfully, you must have the following IP addresses available in your environment:

-    One IP address for the OpenStack services.
-    One IP address for the virtual router to provide connectivity to the project network. This IP address is assigned automatically in this example.
-    One IP address for DHCP on the provider network.
-    At least one IP address for floating IPs on the provider network. 

### Procedure

Create configuration helper variables:

    # standalone with project networking and provider networking
    export OS_CLOUD=standalone
    export GATEWAY=192.168.25.1
    export STANDALONE_HOST=192.168.25.2
    export VROUTER_IP=192.168.25.3
    export PUBLIC_NETWORK_CIDR=192.168.25.0/24
    export PUBLIC_NET_START=192.168.25.4
    export PUBLIC_NET_END=192.168.25.15
    export DNS_SERVER=1.1.1.1

Create a basic flavor:

    $ openstack flavor create --ram 512 --disk 1 --vcpu 1 --public tiny

Download CirrOS and create an OpenStack image:

    $ wget https://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
    $ openstack image create cirros --container-format bare --disk-format qcow2 --public --file cirros-0.4.0-x86_64-disk.img

Configure SSH:

    $ ssh-keygen -m PEM -t rsa -b 2048 -f ~/.ssh/id_rsa_pem
    $ openstack keypair create --public-key ~/.ssh/id_rsa_pem.pub default

Create a simple network security group:

    $ openstack security group create basic

Configure the new network security group:

Enable SSH:

    $ openstack security group rule create basic --protocol tcp --dst-port 22:22 --remote-ip 0.0.0.0/0

Enable ping:

    $ openstack security group rule create --protocol icmp basic

Create Neutron networks:

    $ openstack network create --external --provider-physical-network datacentre --provider-network-type flat public
    $ openstack network create --internal private
    $ openstack subnet create public-net \
        --subnet-range $PUBLIC_NETWORK_CIDR \
        --gateway $GATEWAY \
        --allocation-pool start=$PUBLIC_NET_START,end=$PUBLIC_NET_END \
        --network public \
        --host-route destination=0.0.0.0/0,gateway=$GATEWAY \
        --dns-nameserver $DNS_SERVER

Launch the instance:

    $ openstack server create --flavor tiny --image cirros --key-name default --network public --security-group basic myserver

Test SSH:

    ssh cirros@<VM_IP>

Replace VM_IP with the address of the virtual machine that you create in the previous step. 
