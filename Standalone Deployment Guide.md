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

1. Create a non-root user on the all-in-one host:

    [root@all-in-one]# useradd stack

2. Set the password for the stack user:

    [root@all-in-one]# passwd stack

3. Disable password requirements when using sudo as the stack user:

    [root@all-in-one]# echo "stack ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/stack
    [root@all-in-one]# chmod 0440 /etc/sudoers.d/stack

4. Log in as the non-root user on the all-in-one host:

    [stack@all-in-one]$ ssh stack@<all-in-one>

5. Register the machine with Red Hat Subscription Manager. Enter your Red Hat subscription credentials at the prompt:

    [stack@all-in-one]$ sudo subscription-manager register

6. Attach your Red Hat subscription to the entitlement server:

    [stack@all-in-one]$ sudo subscription-manager attach --auto

[!NOTE] 
> The --auto option might not subscribe you to the correct subscription pool. Ensure that you subscribe to the correct pool, otherwise you might not be able to enable all of the repositories necessary for this installation. Use the subscription-manager list --all --available command to identify the correct pool ID. 

7. Lock the undercloud to Red Hat Enterprise Linux 9.0:

    [stack@all-in-one]$ sudo subscription-manager release --set=9.0
    
8. Run the following commands to install dnf-utils, disable all default repositories, and then enable the necessary repositories:

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


1. Update the base operating system and reboot the system:

    [stack@all-in-one]$ sudo dnf update
    [stack@all-in-one]$ sudo reboot

2. Log back in to the host after the reboot.

3. Install the TripleO command line interface (CLI):

    [stack@all-in-one]$ sudo dnf install -y python3-tripleoclient

# Chapter 4. Configuring the all-in-one Red Hat OpenStack Platform environment

 To create an all-in-one Red Hat OpenStack Platform environment, include four environment files with the openstack tripleo deploy command. You must create two of the configuration files, shown below:

- $HOME/containers-prepare-parameters.yaml
- $HOME/standalone_parameters.yaml 

Two environment files are provided for you in the /usr/share/openstack-tripleo-heat-templates/ directory:

- /usr/share/openstack-tripleo-heat-templates/environments/standalone/standalone-tripleo.yaml
- /usr/share/openstack-tripleo-heat-templates/roles/Standalone.yaml 

You can customize the all-in-one environment for development or testing. Include modified values for the parameters in either the standalone-tripleo.yaml or Standalone.yaml configuration files in a newly created yaml file in your home directory. Include this file in the openstack tripleo deploy command. 

4.1. Generating YAML files for the all-in-one Red Hat OpenStack Platform (RHOSP) environment
Copy link

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

3. Set the ContainerImageRegistryLogin parameter to true in the containers-prepare-parameters.yaml:

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
Copy link

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