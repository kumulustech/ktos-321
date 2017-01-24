# Managing your first VM

1) In the previous lab, we stood up our OpenStack environment, and established a baseline service configuration for an administrative user.  We'll continue to use this user/project for the next few sections, though normally we'd want to build out a user with less access to the overall system.

We've deployed a VM, and our first task will be to find a way to log into the machine.  There are a few options we should explore:

1) via the Horizon UI/VNC interface
2) via the ```nova get-vnc-console {vm_name} novnc``` command
3) via direct ssh (from the control node only in the lab enviornment)

We should be able to leverage any of these solutions from our laptop against the currently running system. We'll take the 'easy' path and leverage the Horizon interface. Find the VM we spun up in the first lab in horizon (login with the credentials from the open.rc file, or user: admin, password: admin1).

Horizon should be available at http://192.168.56.10.

Select select the VM from the list (under project/compute), and then select the console view.

Can you log in?  What credentials would you use?
How might one remedy this situation?

Extra Credit:
- Leverage the nova vnc-console command listed above (select the appropriate VM).  Can you gain access to your VM that way?

## Cloud Init

Let's look at one approach, which uses the cloud-init service.  Cloud-init is a declarative model for initial system bring-up configuration (and in fact post bringup re-configuration as well), and many cloud environments leverage this as a solution.  There's plenty of documentation on things you can do here:  https://cloudinit.readthedocs.io/en/latest/

But we're going to use the simplest model to just pass a little script so that we can get into our VM.

The script looks like this:
```
#!/bin/bash
passwd ubuntu << EOF
ubuntu
ubuntu
EOF
```

And since cloud-init is such a critical component of services like OpenStack, we will find that when we deploy a VM via Horizon (and in fact via CLI) that we can pass the same sort of script.  Use the Horizon wizard to launch a VM and in the "User Data" section, add our little script. This time, select an Ubuntu image (which we installed in the previous section)

Now go to the VNC console, can we get in?
Extra Credit:
- Again, leverage the VNC command line to get a direct VNC access link
- Be a good cloud citizen, delete the VMs from your running enviornment

Extra Extra Credit:
- Look at the output of the ```nova help``` command, and find the floatingIP commands.  Can you use this set of commands to allocate a floating IP to either of our current VMs?
Did you already clean up your VMs? Good, more time to practice launching them!
