# Managing your first VM

1) In the previous lab, we stood up our OpenStack environment, and established a baseline service configuration for an administrative user. In our open.rc file we can see a callout for the Tenant/Project {synonyms for a resource segregation} and the user, both of which are "admin" in our case. We'll continue to use this user/project for the next few sections, though normally we'd want to build out a user with less access to the overall system.

We've deployed a simple CirrOS based VM and our first task will be to find a way to log into the machine.  There are a few options we should explore:

1) via the Horizon UI/VNC interface
2) via the ```nova get-vnc-console {vm_name} novnc``` command
3) via direct ssh (from the control node only in the lab enviornment)

We should be able to leverage any of these solutions from our laptop against the currently running system, and we'll start with the 'easy' path and leverage the Horizon web user interface. Find the VM we spun up in the first lab in horizon (login with the credentials from the open.rc file, or user: admin, password: admin1).

Horizon should be available at http://192.168.56.10.

Select select the VM from the list (under Project=>Compute), and then select our VM (cirros), and then the Console view from the tabs across the top of the page.

 - Can you log in?
 - What credentials would you use?
 - Is this a normal way one would expect to log in?

Slightly more difficult: Extra Credit:
- Leverage the nova vnc-console command listed above (select the appropriate VM).

```
nova get-vnc-console cirros novnc
```

Can you gain access to the CirrOS VM by pasting the novnc address into your laptop browser?

## Cloud Init

With our CirrOS VM, we are able to use the console model for access, and as we've seen the credentials are prominently placed above the login prompt.  This is unusual (and obviously not recommended), and was developed specifically to simplify the testing of basic VM based IaaS functionality.

As one might expect, this is unusual, and in most cases, we will be using a "Cloud Image" that often does not have a password associated with the default account.  The concept behind this approach is to instead leverage password-less public key access, and SSH service access to gain access to our virtual machines, and in fact OpenStack is well configured for this sort of function. However, there are times when we are not able to gain access via a direct SSH capable network path, and it would be useful to apply a password to the default system user. In fact, it might even be useful to establish the default user name so that we can get access, and perhaps derive a bit of obfuscation to that user instead of leveraging the OS version default.

Let's look at a few Cloud-Init based approaches to setting user and password configurations for this purpose. We'll continue to access our VMs via the console service, although we'll also be able to set SSH password access, and by mapping a FloatingIP address to our deployed VM, we can even access the service via SSH directly (and with a password).

Cloud-init is a declarative model for initial system bring-up configuration (and in fact post bringup re-configuration as well), and many cloud environments leverage this as a solution.  There's plenty of documentation on things you can do here:  https://cloudinit.readthedocs.io/en/latest/

But First, we're going to use the simplest model to just pass a little script so that we can get into our VM.

The script for an Ubuntu cloud image (with ubuntu as the default cloud user) looks like this:

```
#!/bin/bash
passwd ubuntu << EOF
ubuntu
ubuntu
EOF
```

Since Cloud-Init is such a critical component of services like OpenStack, we will find that when we deploy a VM via Horizon (and via CLI) that we can pass this script directly as a part of the boot definition. First we'll use the Horizon wizard to launch a VM and in the "User Data" section, add our little script. We'll leverage the Ubuntu cloud image we installed as a part of our deployment process for this deployment.  Create a file (perhaps call it user-data.sh, though the name is not critical, it's just good practice to remember what we are putting in this file).  We'll use this file in the next section as an alternative to the Horizon model for passing cloud data.

Now go to the VNC console, can we get in?

Extra Credit:
- Again, leverage the VNC command line to get a direct VNC access link
- Be a good cloud citizen, delete the VMs from your running environment

#SSH access via public key
- Look at the output of the ```nova help``` command, and find the floatingIP commands.  Can you use this set of commands to allocate a floating IP to either of our current VMs?  

Did you already clean up your VMs? Good, more time to practice launching them!
 - launch an ubuntu VM - either include a cloud init script like the one listed above to set a password, and/or include the ssh key_name we uploaded from the previous section.
 - create a floatingIP
 - associate the floating IP to the vm (hint: the nova commands are much simpler to use than the neutron command variants!)

In the previous section, we either uploaded a new public key (and have the private key in our /root directory) or we leveraged the public key we loaded initially when we built out the system as the 'root' key.  We can check that we have a public key registered via the nova command line, or via the Horizon UI (Project=>Compute=>Access&Security KeyPairs tab).  Use the nova or openstack CLI client help command to discover they "key" command(s).

If we created a new public/private pair (as was described in the instructions), ensure that the private key has the appropriate permissions:
```
chmod 600 id_rsa
```

We can then use this key to log in to our Ubuntu VM to which we associated the Floating IP in the previous section:
```
ssh ubuntu@{floating_ip_address} -i ./id_rsa
```

This is great, but what if we want to use the cloud-init tools to just set a password instead?

On an ubuntu instance, we need to allow password based ssh, and we need to set a password for the ubuntu user.

Via a bash script:
```
#!/bin/bash
cat /etc/ssh/sshd_config | sed -e 's/PasswordAuthentication.*/PasswordAuthentication yes/' > /etc/ssh/sshd_config
systemctl restart sshd

passwd ubuntu <<EOF
newpassword
newpassword
EOF
```

Create a file (user-data.txt), place the contents into the file, and lauch an ubuntu VM with --user-data user-data.txt and no --key-name parameter.  Or launch a VM via horizon. In either case, you'll need an m1.medium (1core 1G, 4G disk) VM.

Were you able to log in via password:
  - via VNC
  - via SSH (don't forget to add a FloatingIP to the VM)

Extra Credit
 - Look at http://cloudinit.readthedocs.io/en/latest/topics/examples.html
   Can you create a #cloud-config version of the user-data script?
