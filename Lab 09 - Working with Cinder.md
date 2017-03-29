# OpenStack Block Storage - cinder

For Cinder, we will have to enable additional servcies and re-build our OpenStack enviornment.

1) Delete any running VMs (they need to go away to allow us to re-create the containers)
2) We will also loose our image and networks, so the network and image scripts can re-create them quickly.

```
echo 'enable_cinder: "yes"' >> /etc/kolla/globals.yml

kolla-ansible destroy -i multinode --yes-i-really-really-mean-it
kolla-ansible deploy -i multinode
```

Once that's done, we can start creating Volumes, though it's possible you did this via the earlier Nova Extra Credit.

- create a volume and associate it with a running VM
- log in to the VM, and create a file system on it:

```
mkfs.ext4 /dev/vdb
mount /dev/vdb /mnt
touch /mnt/file_to_prove_that_our_disk_is_the_same_one_we_initially_created
```

 - create a backup of this volume, and once complete
  - remove the file
  - detach the volume restore from backup
 - validate that the file is back

## Extra Credit
- create a second project
- transfer the volume from your project to the other project and associate it with a vm in that project (create a network, VM, etc.)
