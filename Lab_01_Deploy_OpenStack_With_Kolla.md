# Deploying OpenStack With Kolla and Docker

We'll deploy a Virtual Machine in order to run the Kola environment locally, though an external/remote VM is also an appropriate choice.

There are many possible pathways for implementing this solution:

1) Vagrant for Mac, Linux, or Windows based machines (laptop or other) along with ansible and/or shell scripts for deployment.

2) Terraform for Mac, Linux, or Windows against a cloud provider (e.g. Packet.net, amazon, digital ocean, etc.) along with ansible and/or shell scripts for deployment.

3) Manual for a local virtualization manager with a baseline Virtual Machine, along with ansible and/or shell scripts for deployment or manual on a remote system or local virtual machine (bare metal or nested capable virtual machine)

## Deploying with Vagrant
Prerequisites:
 - At least 6GB of systems memory
 - At leaset 10GB of free disk space

1) Download and install Vagrant
  https://www.vagrantup.com
2) Download and install VirtualBox
  https://www.virtualbox.org/wiki/Downloads
  You'll also want to install the guest additions for completeness that are listed on the same page as the download link for Virtualbox.
3) Get the Vagrantfile and install scripts from the class repository:

https://github.com/kumulustech/vagrant-kolla

One can do this via git, as in:

```
git clone https://github.com/kumulustech/vagrant-kolla.git vagrant-kolla
```

Or you can grab the code as a zip file:

```
curl https://github.com/kumulustech/vagrant-kolla/archive/master.zip
```

4) Change to the directory where you cloned the git repository or extracted the zip file. This is where you will launch ssh access to your deployed hosts using the 'vagrant ssh' tool, and where you will manipulate the vagrant deployed VMs. You will not need to use the VirtualBox interface for this effort.

5) Now we can launch our virtual environment which is as simple as:

```
vagrant up
```

6) Wait.  Depending on the upstream bandwidth available (and the speed of the Docker hub registry) the install can happen in as little as a few minutes, or as much as many hours.

At this point, the Vagrant run should indicate that it has finished, and we can now log into the controller for our environment, from which we will manipulate our multi-node openstack environment.

```
vagrant ssh control
sudo su -
```

You should now be the root user, and in the local directory, are a large number of scripts, including an open.rc file. This file includes the credentials needed to connect to our OpenStack environment. From that you can now run:

```
source open.rc
nova flavor-list
glance image-list
neutron net-list
```

If the nova and neutron commands return empty fields, or no data at all, we'll need to configure some local OpenStack resources, jump to step 7.  If they do return data, then we're ready to start in on our labs.


## Configuring the OpenStack Tenant and "Provider" resources

7)  Now that you have a baseline OpenStack system, we are ready to configure the OpenStack system resources including network and baseline virtual machine images.

We will:
 - install a virtual machine image
 - create a local network, router, and "floatingIP" network and pool of addresses
 - upload an ssh key and/or (depending on the image) force a password to be set on the image at boot time

We'll need the following resources:
 - an "open.rc" script,  which we can either manully create, or download via the Horizon user interface [Project->Access&Security=>API Access]
 or create one from the following template:

Regardless, we'll need to discover the administrator password, which we can get from the control node:

```
grep keystone_admin_password /etc/kolla/passwords.yml
```

Create a file called open.rc, and add the following to it:
```
#!/bin/bash
echo "running this file is not adequate, you must 'source' it"
echo "source $0   or . $0"
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD={admin_password_from_the_kolla_script}
export OS_AUTH_URL=http://${eth1_interface_ip}:35357/v3
export OS_IDENTITY_API_VERSION=3
```

'source' it to apply the environment variables to the local shell:

```
. open.rc
```

 - to work with the CLI interface, we need the tools installed. This can be done on your local machine (and often this is very useful), or on the machine where OpenStack is installed (though this adds potential complications for ssh/public/private key management).  The install scripts will usually have installed the required code onto the control machine, but for completeness, you can either follow the OpenStack.org recomendations here:

 ```
 http://docs.openstack.org/user-guide/common/cli-install-openstack-command-line-clients.html
 ```

 Or you can use this script:

```
#!/bin/bash
source ~/open.rc
yum install python-devel python-pip
pip install python-openstackclient python-keystoneclient python-glanceclient python-novaclient python-cinderclient python-neutronclient
```

 - a "cloud" image, either centos, or cirros {a lightweight "test" image}

```
   http://docs.openstack.org/image-guide/obtain-images.html
```

And again, we can install via Horizon [Project->Images=>Create Image], or via CLI:

```
source /root/open.rc
curl -o cirros-0.3.4-x86_64-disk.img http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
openstack image create --container-format bare --disk-format raw --min-disk 1 --min-ram 256 --public --file cirros-0.3.4-x86_64-disk.img cirros
rm cirros-0.3.4-x86_64-disk.img

curl -o ubuntu-xenial.img \
https://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-amd64-disk1.img
openstack image create --container-format bare --disk-format qcow2 \
--min-disk 1 --min-ram 512 --public --file \
ubuntu-xenial.img xenial
rm ubuntu-xenial.img
```

 - a network configuration.  This script can accelerate the configuration of our networks, and is already on the controller:

```
setup_network.sh
```

```
#!/bin/bash
#   Copyright 2016 Kumulus Technologies <info@kumul.us>
#   Copyright 2016 Robert Starmer <rstarmer@gmail.com>
#
#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#       http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.source ~/admin.rc

source /root/open.rc

tenant=`openstack project list -f csv --quote none | grep admin | cut -d, -f1`

public_network=${1:-192.168.57}
ip a d ${public_network}.10/24 dev enp0s9
ip l s enp0s9 up
ip a a ${public_network}.254/24 dev br-ex
ip l s br-ex up

# Create an external "provider" network for our floating IP addresses
neutron net-create public --tenant-id ${tenant} --router:external --provider:network_type flat --provider:physical_network physnet1 --shared
#if segmented network{vlan,vxlan,gre}: --provider:segmentation_id ${segment_id}

# Add L3 information (and a pool of addresses for Floaging IP use)
neutron subnet-create public ${public_network}.0/24 --tenant-id ${tenant} --allocation-pool start=${public_network}.80,end=${public_network}.99 --dns-nameserver 8.8.8.8 --disable-dhcp
# if you need a specific route to get "out" of your public network: --host-route destination=10.0.0.0/8,nexthop=10.1.10.254

# And our Tenant/project needs a separate network
neutron net-create private --tenant-id ${tenant}
neutron subnet-create private 192.168.100.0/24 --tenant-id ${tenant} --dns-nameserver 8.8.8.8 --name private

# Stitch the two together, create a router and attach the interfaces
neutron router-create pub-router --tenant-id ${tenant}
neutron router-gateway-set pub-router public
neutron router-interface-add pub-router private

# Adjust the default security group.  This is not good practice
# But otherwise, we have no chance of connecting to our VMs via SSH or
# via HTTP

default_group=`neutron security-group-list | awk '/ default / {print $2}' | tail -n 1`
neutron security-group-rule-create --direction ingress --port-range-min 22 --port-range-max 22 --protocol tcp --remote-ip-prefix 0.0.0.0/0 ${default_group}
neutron security-group-rule-create --direction ingress --port-range-min 80 --port-range-max 80 --protocol tcp --remote-ip-prefix 0.0.0.0/0 ${default_group}
neutron security-group-rule-create --direction ingress --port-range-min 443 --port-range-max 443 --protocol tcp --remote-ip-prefix 0.0.0.0/0 ${default_group}
neutron security-group-rule-create --direction ingress --protocol icmp --remote-ip-prefix 0.0.0.0/0 ${default_group}
```

 This script will still need to have at least the 192.168.57.1 network addresses updated to map to the network on your host that maps to an external network. In the vagrant deployed environment, this is our enp0s9 network interface, and though it's not currently bridged, it should already have a statically assigned address of 192.168.57.10 to it so that our control host at least can access clients on this network.  And in fact our Laptop should be on 192.168.57.1, so we should have a chance to get there via laptop as well.

 This class of network configuration can also be accomplished via Horizon, but there is no "Wizard" to walk through the steps.

Extra Credit:  Have a look at the admin tab and find the Network resources in the Admin section, compare them to the Network section in the Project section.  Could you create a similar network via the Horizon UI?

 - an ssh public/private key pair.  It's possible to create your own or have OpenStack create one for you (again via the Horizon interface) [Project->Access&Security=>Key Pairs]

 The benefit of creating your own, especially if you are on a Windows machine, is that you can usually use the tools built into the SSH terminal client to create keys, which will generate the right private format for the local system (putty is a notorious ssh client for keeping keys in a different format from the Unix "norm"). On a Linux (or Mac) system, open a terminal session and create a ssh pair, public and private,  if you don't already have one, as follows:

```
ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa
```

NOTE: If you install the CLI tools on the OpenStack control VM, you may want to create a local keypair to use as the source of your access to the OpenStack VMs

this will create an id_rsa file (the private key, you don't normally share this), and the id_rsa.pub key (the public key, paste this on the back of your car, write it across the sky, load it into your OpenStack environment, into git, bitbucket, everywhere that you can think of etc.)

We need to get this key installed into OpenStack to make it useful, which can be done via horizon (the same Key Pairs page listed above).  Install the _PUBLIC_ key into OpenStack.

VIA CLI:

```
nova keypair-add --pub-key ~/.ssh/id_rsa.pub root
```

One last item that's needed is a description of the types of virtual machines we can deploy. These "flavors" define the available memory, # of compute cores, and "ephemeral" or persistent block storage the VM can be built from.  We need to define at least a couple basic sizes in order to launch any VMs.  Let's use the following to create a tiny, small, and medium VM:

```
openstack flavor create --public  --id  1 --ram 256 --disk 1 --vcpus 1 m1.tiny
openstack flavor create --public  --id  3 --ram 512 --disk 1 --vcpus 1 m1.small
openstack flavor create --public  --id  5 --ram 1024 --disk 2 --vcpus 2 m1.medium
```

8) Finally, we should be able to boot a VM!

```
nova boot --flavor m1.tiny --image cirros --nic net-id=`neutron net-list | awk '/private/ {print $2}'` --key-name=root cirros
```

Extra Credit:
 - What is the meaning of the "net-id" parameter
 - What happens if you try to boot a VM without the -nic net-id=.... parameter
 - How else might you determine the net-id parameter?
