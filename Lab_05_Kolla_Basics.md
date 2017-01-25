# Kolla Basics

Kolla can be broken up into at least two main functions:
- OpenStack containerization build process (and optional registration of images)
- OpenStack container download (pull), launch, and configuration

In addition there are tools/functions to help cleanup installed resources, manage the process of upgrade and downgrade of system releases.  The current version focuses on a Docker container based environment without additional lifecycle management, meaning there's no-one there besides the Docker daemon to keep the containers running and happy.

## Kolla Build
One function Kolla supports is the creation of service containers for nearly all the OpenStack service tools, and the underlying support resources (databases, rabbit/MQ, etc.).  While we can currently leverage the Kolla 3.0.0 (Newton) containers in docker hub, it's also possible to create and leverage local build and local re-build containers.

To build a container, we need to know the following container parameters:
type:
  one of source or binary: are we downloading source from OpenStack, or linux packages from an Ubuntu, RedHat, Suse, etc.
base:
  what is the base OS?  centos, ubuntu, etc.
tag:
  can be anything, but most commonly (for Kolla at least) is the release number (3.0.0 for Newton for example)

let's rebuild the mistral container:
```
kolla-build --type source --base ubuntu --tag 3.0.0 mistral
```
The output should end up with something like:
```
INFO:kolla.image.build:=========================
INFO:kolla.image.build:Successfully built images
INFO:kolla.image.build:=========================
INFO:kolla.image.build:mistral-engine
INFO:kolla.image.build:openstack-base
INFO:kolla.image.build:base
INFO:kolla.image.build:mistral-executor
INFO:kolla.image.build:mistral-base
INFO:kolla.image.build:mistral-api
```
We should also see the resultant containers showing up as available in the docker images library:
```
docker images | grep mistral
```

NOTE: Unless we push the image into a repository, we'd have to manually create this image anywhere that it is/was needed (e.g. every control host).

## Kolla Deploy/Reprovision/Destroy

The main function for Kolla however, is to install the created containers, ensuring any underlying resources are appropriately configured.

To deploy a new service, we have to first manipulate and configure either the "globals.yml" resource model (usually in /etc/kolla/globals.yml), or the "all.yml" file (/usr/local/share/kolla/ansible/group_vars/all.yml).  These files (gobals overrides all) describe the services that should be installed.

In addition, there are default configuratinos that will end up in /etc/kolla/{project}/configurations.  While it is possible to edit these configurations directly (and restart the services for the changes to take effect), Kolla provides a mechanism to override any resource within the configuration without having to manipulate the "default" configurations.  This simply requires creating a sub-directory in /etc/kolla called config/{project}/file.conf.

An example is the desire to run the qemu virtualization controller rather than kvm.  In order to accomplish this, we create:
```
/etc/kolla/config/nova-compute/nova.conf
```
If this file wasn't there before, we can update whatever configuration we need to manipulate by adding it to that file:
```
cat /etc/kolla/config/nova/nova-compute.conf
[libvirt]
virt_type=qemu
```

If we make a change to a configuration, or to the images we have available, or the list of resources to deploy, we can reconfigure the environment with Ansible:
```
kolla-ansible redeploy -i {multi-node-inventory}
```

## Deployment

This is really the most common use case.

1) Get the repository:
```
git clone https://github.com/openstack/kolla -b stable/newton
```

2) Install kolla
```
pip install kolla/
```
Note: this installs all the tools defined in the kolla directory (git repo)

3) set up basic "Global" config:
cp -r kolla/etc /etc/kolla/

4) Create passwords (rather than manually configuring them in /etc/kolla/passwords.yml)
```
kolla-genpwd
```

This will autogenerate the passwords for all of the service level interations.  You may still want to edit the value of keystone_admin_password: to something other than the long random string created.

5) Update/configure a multi-node inventory file.

The default multinode inventory lives here:
/usr/local/share/kolla/ansible/inventory/multinode.

Edit/update as appropriate for your systems.

6) Run the pre-checks or possible the bootstrap service:

```
kolla-ansible prechecks -i /usr/local/share/kolla/ansible/inventory/multinode

and/or
kolla-ansible bootstrap-servers -i /usr/local/share/kolla/ansible/inventory/multinode
```

7) Finally, deploy!
```
kolla-ansible deploy -i /usr/local/share/kolla/ansible/inventory/multinode

8) At this point you have a running OpenStack system, but you still need to:

- add images
- add nova flavor definitions
- add neutron networks, and routers
