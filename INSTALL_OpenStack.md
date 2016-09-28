ansible-hdp installation guide
------------------------------

* These Ansible playbooks can build a Cloud environment in a private OpenStack.

---


# Build setup

Before building anything, the build node / workstation from where Ansible will run should be prepared.

This node must be able to connect to the cluster nodes via SSH and to the OpenStack APIs via HTTPS.

As OpenStack environments are usually private, you might need to build such a node in the OpenStack environment.


## CentOS/RHEL 7

1. Install the required packages

  ```
  sudo yum -y install epel-release || sudo yum -y install http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
  sudo yum install gcc gcc-c++ python-virtualenv python-pip python-devel libffi-devel openssl-devel libyaml-devel sshpass git vim-enhanced -y
  ```


1. Create and source the Python virtual environment

   ```
   virtualenv ~/ansible; source ~/ansible/bin/activate 
   ```


1. Install the required Python packages inside the virtualenv

   ```
   pip install setuptools --upgrade
   pip install pip --upgrade   
   pip install functools32 pytz ansible shade
   ```


1. Turn off SSL validation (required if your OpenStack endpoints don't use trusted certs)
  
  ```
  sed -i '/{$/a "verify": false,' ~/ansible/lib64/python2.7/site-packages/os_client_config/defaults.json
  ```


1. Install the SSH private key

  The build node / workstation will need to login via SSH to the cluster nodes.
  
  For this to succeed, the SSH private key needs to be placed on the build node / workstation, normally under .ssh, for example: `~/.ssh/field.pem`. It can be placed under any path as this file will be referenced later.
  
  The SSH public key must be present on the OpenStack environment as it will be referenced when the nodes will be built (this can be checked on the Dashboard, under `Compute` -> `Access and Security` -> `Key Pairs` tab).


## Ubuntu 14+

1. Install required packages:

  ```
  sudo apt-get update
  sudo apt-get -y install unzip python-virtualenv python-pip python-dev sshpass git libffi-dev libssl-dev libyaml-dev vim
  ```


1. Create and source the Python virtual environment

   ```
   virtualenv ~/ansible; source ~/ansible/bin/activate  
   ```


1. Install the required Python packages inside the virtualenv

   ```
   pip install setuptools --upgrade
   pip install pip --upgrade
   pip install functools32 pytz ansible shade
   ```


1. Turn off SSL validation (required if your OpenStack endpoints don't use trusted certs)
  
  ```
  sed -i '/{$/a "verify": false,' ~/ansible/local/lib/python2.7/site-packages/os_client_config/defaults.json
  ```


1. Install the SSH private key

  The build node / workstation will need to login via SSH to the cluster nodes.
  
  For this to succeed, the SSH private key needs to be placed on the build node / workstation, normally under .ssh, for example: `~/.ssh/field.pem`. It can be placed under any path as this file will be referenced later.
  
  The SSH public key must be present on the OpenStack environment as it will be referenced when the nodes will be built (this can be checked on the Dashboard, under `Compute` -> `Access and Security` -> `Key Pairs` tab).


# Setup the OpenStack credentials

1. Download the OpenStack RC file

  Login to your OpenStack dashboard, and download your user specific OpenStack RC file.
  This is usually found on `Compute` -> `Access and Security` under the `API Access` tab. Download the v3 if available.


1. Apply the OpenStack credentials

  Copy the file to the build node / workstation in a private location (for example the user's home folder).

  And `source` the file so it populates the existing session with the OpenStack environment variables.
  Type your OpenStack account password when prompted.

  ```
  source ~/ansible/bin/activate
  source ~/*-openrc.sh
  Please enter your OpenStack Password: 
  ```

  You can verify if it worked by trying to list the existing OpenStack instances:
  ```
  nova --insecure list
  ```


# Clone the repository

Upload the ansible-hdp repository to the build node / workstation, preferable under the home folder.

If the build node / workstation can directly download the repository, run the following:

```
cd && git clone https://github.com/hortonworks/ansible-hdp.git
```

If your GitHub SSH key is installed, you can use the SSH link:

```
cd && git clone git@github.com:hortonworks/ansible-hdp.git
```


# Set the OpenStack variables

Modify the file at `~/ansible-hdp/inventory/openstack/group_vars/all` to set the OpenStack configuration.


## cloud_config
This section contains variables that are cluster specific and are used by all nodes:

| Variable        | Description                                                                                                |
| --------------- | ---------------------------------------------------------------------------------------------------------- |
| name_prefix     | A prefix that will precede the name of all nodes. Usually the cluster name to uniquely identify the nodes. |
| name_suffix     | A suffix that will be appended to the name of all nodes. Usually it's a domain, but can be anything or even the empty string `''`. |
| zone            | The name of the OpenStack zone.                         |
| admin_username  | The Linux user with sudo permissions. This user is specific to the image used. For example, in a CentOS image, it can be `centos` or in a Ubuntu image it can be `ubuntu`. |
| ssh.keyname     | The name of the SSH key that will be placed on cluster nodes at build time. This SSH key must already exist in the OpenStack environment. |
| ssh.privatekey  | Local path to the SSH private key that will be used to login into the nodes. This is the key uploaded to the build node as part of the Build Setup, step 5. |


## nodes config

This section contains variables that are node specific.

Nodes are separated by groups, for example master, slave, edge.

There can be any number of groups.

And groups can have any names and any number of nodes but they should correspond with the host groups in the Ambari Blueprint.


| Variable        | Description                                                               |
| --------------- | ------------------------------------------------------------------------- |
| group           | The name of the group. Must be unique in the OpenStack Zone. This is the reason why the default contains the `name_prefix`. Other groups can be added to correspond with the required architecture. |
| count           | The number of nodes to be built in this group. |
| image           | The name or ID of the OS image to be used. A list of the available images can be found by running `nova --insecure image-list`. |
| flavor          | The name or ID of the flavor / size of the node. A list of all the available flavors can be found by running `nova --insecure flavor-list`. |                                                      |
| public_ip       | If the Public IP of the cluster node should be used when connecting to it. Required if the build node does not have access to the private IP range of the cluster nodes. |
| ambari_server   | Set it to `true` if the group also runs an Ambari Server. The number of nodes in this group should be 1. If there are more than 1 node, ambari-server will be installed on all of them, but only the first one (in alphabetical order) will be used by the Ambari Agents. |


# Build the Cloud environment

Run the script that will build the Cloud environment:

```
cd ~/ansible-hdp*/ && bash build_openstack.sh
```

You may need to load the environment variables if this is a new session:

```
source ~/ansible/bin/activate
source ~/*-openrc.sh
```


# Set the cluster variables

Modify the file at `~/ansible-hdp/playbooks/group_vars/all` to set the cluster configuration.

| Variable        | Description                                                                                                |
| --------------- | ---------------------------------------------------------------------------------------------------------- |
| ambari_version  | The Ambari version, in the full, 4-number form, for example: `2.4.1.0`. |


# Prepare the nodes

Run the script that will prepare the nodes for the Ambari installation:

```
cd ~/ansible-hdp*/ && bash prepare_nodes_openstack.sh
```

You may need to load the environment variables if this is a new session:

```
source ~/ansible/bin/activate
source ~/*-openrc.sh
```


# Install Ambari

Run the script that will install and configure Ambari Agents and Ambari Server:

```
cd ~/ansible-hdp*/ && bash install_ambari_openstack.sh
```

You may need to load the environment variables if this is a new session:

```
source ~/ansible/bin/activate
source ~/*-openrc.sh
```