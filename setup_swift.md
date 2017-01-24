1) copy the setup_swift.sh script to the VM
```
scp setup_swift.sh {vm_name}:
```

2) on the VM (ssh login), add the enable_swift: "yes" parameter to the globals.yml file
```
echo 'enable_swift: "yes"' >> /etc/kolla/globals.yml
echo 'enable_cinder: "yes"' >> /etc/kolla/globals.yml
```

3) Otherwise run both re-deploy:
```
for n in `docker ps -qa`; do docker stop $n; docker rm -v $n; done
setup_swift.sh
kolla-ansible deploy
```
