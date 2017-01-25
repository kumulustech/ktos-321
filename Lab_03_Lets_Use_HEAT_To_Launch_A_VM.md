# Orchestrating OpenStack - HEAT

While there are already small elements of automation embedded in all of the OpenStack services, there is always a need for another layer.  One such layer is the concept of a template that describes the relationships between elements, and this is the space that HEAT fills in the OpenStack service space.  There are other automation elements as well, such as Mistral which is a workflow based model for automation, but they have more to do with application level automation, while HEAT focuses on the system.

## The Setup

We will require a real cloud image and can use the Ubuntu image we installed initially. The goal is to have a VM running that we can use as a web server or other useful service resource.

Verify that you have an image in Glance to start with.

### Exercise
- load a ubuntu image into glance if it's not there
  - how can you determine what is already there?
- ensure you have a ssh key installed as well
  - how can you determine what keys already exists?

## The template

Everything revolves around the template, so this is the one we'll be using to explore the basic features of HEAT:

```
heat_template_version: 2013-05-23

description: Simple template to deploy a bastion host with the CLI tools

parameters:
  key_name:
    type: string
    label: Key Name
    description: Name of key-pair to be used for compute instance
  image:
    type: string
    label: Image Name
    default: ubuntu
    description: Image to be used for compute instance
  instance_type:
    type: string
    label: Instance Type
    default: m1.large
    description: Type of instance (flavor) to be used
  network:
    type: string
    label: Network Name
    default: private
    description: Network name to associate server with
  public_net:
    type: string
    label: Public Network
    default: public
    description: Network name for Floating IPs
  tenant_name:
    type: string
    label: Tenant Project Name
    default: admin
  user_name:
    type: string
    label: Tenant User Name
    default: admin
  user_pass:
    type: string
    label: Tenant User Password
    default: admin1

  # floating_ip_id:
  #   type: string
  #   label: ID of Floating IP

resources:
  cloud_tools:
    type: OS::Nova::Server
    properties:
      key_name: { get_param: key_name }
      image: { get_param: image }
      flavor: { get_param: instance_type }
      networks:
        - port: { get_resource: server_1_port }
      name: { get_param: user_name }
      admin_user: ubuntu
      user_data:
        str_replace:
          template: |
            #!/bin/bash
            #
            # Setup a CentOS VM with the OpenStack Cloud Tools

            user=`ls /home | head -1`
            # Create a local password
            passwd $user <<EOF
            DiffiCultPassWordToRemember
            DiffiCultPassWordToRemember
            EOF

            cat >> ~/.bash_profile <<EOF
            export PS1='[\u@\h \W($user_name@$tenant_name)]\$ '
            EOF

          params:
            $tenant_name: { get_param: tenant_name}
            $user_name: { get_param: user_name }
            $password: { get_param: user_pass }

  server_1_port:
    type: OS::Neutron::Port
    properties:
      network: { get_param: network }
  server_1_floating_ip:
    type: OS::Neutron::FloatingIP
    properties:
      floating_network: { get_param: public_net }
  server_1_floating_ip_association:
    type: OS::Neutron::FloatingIPAssociation
    properties:
      floatingip_id: { get_resource: server_1_floating_ip }
      port_id: { get_resource: server_1_port }

  # server_1_floating_ip_association:
  #   type: OS::Neutron::FloatingIPAssociation
  #   properties:
  #     floatingip_id: { get_param: floating_ip_id }
  #     port_id: { get_resource: server_1_port }

outputs:
  floating_ip:
    description: The Floating IP address of the deployed server
    value: { get_attr: [server_1_floating_ip, floating_ip_address] }
  server_info:
    description: values of server
    value: { get_attr: [cloud_tools, show]}
```

The input to this script includes parameters like the ssh key that would have been uploaded in an earlier lab, and the username and project name.  While some of the parameters are defaults that should be adequate to launch the instance.

### Launch via Horizon
Find the Orchestration section of Horizon and:

- load the current template above
- provide the required parameters on the second page
- launch the instance

Assuming the instance launches appropriately, investigate the current state as described both by the Orchestration pages (including the graphical interaction diagram and the functional state of the separate orchestration elements)

- look at the instances tab of the Compute table, do the resources match
- the same question can be asked for network

### Launch via CLI

We have two options:

```
heat stack create
```

or

```
openstack stack create
```
- Install the heat client if the above commands don't work.
```
pip install python-heatclient
```

- Use the create command to launch the stack, passing the required (and not defaulted) parameters on the command line.


### Extra Credit

- It is also possible to pass the parameters via a properties file.  Create such a file and launch the stack again (hint: YAML)
- Update the template by adding a second compute instance (it doesn't need a floating IP etc.), load the updated template, and Update the stack
