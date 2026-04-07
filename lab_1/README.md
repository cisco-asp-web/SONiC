## Cisco SONiC 8101-32FH-O Emulator running on Containerlab

Requirements:
* Ubuntu 22.04 or 24.04  
* **4 vCPU, minimum 10G memory per node** 
* Containerlab (tested using 0.74.3)
* docker, kvm
* Dockerized Cisco 8101 emulator image: *`c8000-clab-sonic:94`*


1. clone the repo
```
git clone https://github.com/brmcdoug/cisco8000e.git
```

2. cd into *`./SONiC/lab_1/`*
```
cd ./SONiC/lab_1/
```

3. Optional - edit the topology yaml as needed [topology.yaml](./topology.yaml)

4. Deploy the topology
```
clab deploy -t topology.yaml
```

>[Note] it takes the 8000 emulator nodes several minutes to come up. Monitor their progress with 'docker logs' command. Example: *`docker logs -f leaf-01`*. Routers are ready when logs reach a point that says something like:

```
19:59:50 INFO Sim up
Router up
```

4. ssh to routers *`(user:pw = admin:password)`* 
   
[SONiC CLI Reference](https://github.com/cisco-asp-web/SONiC/blob/main/SONiC_cli_reference.md)

5. Optional: run the accompanying ansible script to apply configs (config_db.json and frr)
```
cd ansible/

ansible-playbook -i hosts config-playbook.yaml -e "ansible_user=admin ansible_ssh_pass=password ansible_sudo_pass=password" -vv
```


