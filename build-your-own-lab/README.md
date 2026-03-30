## Building a SONiC Cisco 8000 Emulator lab with VXR 

A quick guide to acquiring your SONiC 8000 emulator images and installing the VXR orchestrator tool

### Contents
- [Building a SONiC Cisco 8000 Emulator lab with VXR](#building-a-sonic-cisco-8000-emulator-lab-with-vxr)
  - [Contents](#contents)
  - [Cisco dCloud Overview](#cisco-dcloud-overview)
  - [Acquire VXR Package and SONiC Images](#acquire-vxr-package-and-sonic-images)
  - [Install VXR](#install-vxr)
    - [pyvxr CLI tool](#pyvxr-cli-tool)
  - [VXR Topology File](#vxr-topology-file)
    - [Notes on the VXR topology file:](#notes-on-the-vxr-topology-file)
  - [Start the Topology Simulation](#start-the-topology-simulation)


### Cisco dCloud Overview
dCloud is Cisco's cloud based demo platform that showcases a large catalog of demos, training and sandboxes for every Cisco architecture. This is the chosen platform to demonstrate our SONiC labs as they will be available through Cisco's global DC infrastructure.

For each lab within this repository you will find a corresponding lab in dCloud. You will need a dCloud account in order to access these labs. You can login to dCloud at https://dcloud.cisco.com

Once you have scheduled a lab in dCloud Cisco AnyConnect VPN connection information will be provided for the lab instance. Once the dCloud session is deployed, all lab guides, configs, diagrams, and code are here in this github repository.

For instructions to build your own dCloud instance to host a SONiC 8000 emulator topology see [BYO dCloud Guide](https://github.com/brmcdoug/byo-dcloud/blob/main/Lab-Guide-for-BYO-dCloud-Lab.pdf)

### Acquire VXR Package and SONiC Images

1. Navigate to the [VXR Release site](http://vxr8000/8000/), select "Go to latest release"
2. Select the [General EFT 17.0 release](http://vxr-nfs-02/8000/download/eft17.0/base_release)
3. Download both the 8000-emulator-eft tarball and the 8000-sonic-eft tarball:

[8000-emulator-tar](http://vxr-nfs-02/8000/download/eft17.0/base_release/8000-emulator-eft17.0.tar)

[8000-sonic-image-tar](http://vxr-nfs-02/8000/download/eft17.0/base_release/8000-sonic-eft17.0.tar)

4. Aaaand when they're done downloading you can upload them to your Ubuntu topology host. Example:
```
scp 8000-emulator-eft17.0.tar cisco@198.18.133.100:/home/cisco/images
scp 8000-sonic-eft17.0.tar cisco@198.18.133.100:/home/cisco/images
```

### Install VXR

1. untar both packages, 8000-emulator first:
```
tar -xvf 8000-emulator-eft17.0.tar
```

Then untar the 8000-sonic package, which will extract files into the ./8000-eft/ directory:
```
tar -xvf 8000-sonic-eft17.0.tar 
```

Example tail of the output:
```
8000-eft17.0/packages/images/8000/sonic/onie-recovery-x86_64-cisco_8000-r0.iso
8000-eft17.0/packages/images/8000/sonic/onie-recovery-x86_64-cisco_8000-r0.qcow2
8000-eft17.0/packages/images/8000/sonic/sonic-cisco-8000.bin
```


2. cd into the 8000-eft17.0/ scripts directory and run the ubuntu installation script
```
cd ./8000-eft17.0/scripts
```

```
sudo ./ubuntuServerManualSetup.sh
```

The script will run for a couple minutes as it installs packages, etc.
A successful run will end with:

```
[SERVER SETUP IS COMPLETED]
```

#### pyvxr CLI tool

1. The VXR installation includes the pyvxr CLI tool which we'll use to start, stop, or check the status on our sonic virtual topologies

```
vxr.py -h
```

Example partial output:
```
$ vxr.py -h
usage: vxr.py [-h] [--version]
    {start,clean,consoles,stop,ports,restart,sim-info,sim-check,status,logs,id,bringup,toxml,vcpu-count,oir,save-xr-config,restore-xr-config,nsim-log,user_ctl,dump,ngdp,link,tcpdump,help}
```

2. Check VXR version:
```
vxr.py --version
```

Example output:
```
$ vxr.py --version
1.6.67
```

### VXR Topology File

Similar to tools like containerlab, VXR uses a yaml file for defining virtual network topology and node parameters. [topology file for our project](./topology.yaml)

1. git clone this repo to your topology host
```
git clone https://github.com/cisco-asp-web/SONiC.git
```

2. Move or copy the SONiC *`onie`* and *`sonic.bin`* files to a directory where you plan to do your work

```
ls images/8000-eft17.0/packages/images/8000/sonic/
```

```
cp images/8000-eft17.0/packages/images/8000/sonic/onie-recovery-x86_64-cisco_8000-r0.iso /home/cisco/images/

cp images/8000-eft17.0/packages/images/8000/sonic/sonic-cisco-8000.bin /home/cisco/images/sonic-cisco-8000-vxmh.bin
```

>[Note] We've renamed the sonic-cisco-8000.bin file to be *`sonic-cisco-8000-vxmh.bin`* as the particular image we're using for this project supports VXLAN multi-homing.

#### Notes on the VXR topology file:

```diff
simulation:
  skip_platform_check: True
+  sim_dir: /home/cisco/SONiC  <-- directory we'll run the simulation from; also location for log files
  skip_auto_bringup : True
  no_image_copy: true
  vxr_sim_config:
    default:
       ConfigEnableNgdp: 'true'

+devices:    <-- section where we define the nodes in our topology
# Leaf 1
  r1:
+    console_ports: [40001] <-- console access
+    memory: 10G  <-- documentation recommends 20G but we can do 10G
    platform: spitfire_f
+    linecard_types: [8102-64H]  <-- see supported platforms in the vxr release site
+    os_type: sonic  <-- vxr also supports ios-xr
    custom_mgmt_inf: True
    mgmt_intf_address: 192.168.122.101/16
    linux_username: "admin"
    linux_password: "password"
+    image: /home/cisco/images/onie-recovery-x86_64-cisco_8000-r0.iso <-- note 'absolute paths'
+    onie-install: /home/cisco/images/sonic-cisco-8000.bin  <--
    data_ports:
      - Ethernet0
      - Ethernet4
      - Ethernet8
      - Ethernet12
    vxr_sim_config:
      shelf:
        ConfigMgmtMacAddr: "00:01:02:03:04:01" 
```

### Start the Topology Simulation

1. cd into the project directory and start the topology/simulation
```
cd ./SONiC/01-build-your-lab/
```
```
vxr.py start topology.yaml
```

The topology startup will take 8-10 minutes.
A common error: missing SDK file:
```
Found FATAL errors in vxr log file
!!!!! ERROR LOADING SHARED LIBRARIES !!!!!
FATAL: !!!!! Failed to load NGDP library: libngdp-gib-24.11.4140.17-4.so !!!!!
```

If an error such as this occurs:

1. Navigate back to the VXR downloads site and click the *`more sdks here`* link: [http://vxr8000/8000/ngdp/](http://vxr8000/8000/ngdp/) 

2. Find and download the debian package that matches the missing file name. In this case: [vxr2-ngdp-24.11.4140.17_1-1_all.deb](vxr2-ngdp-24.11.4140.17_1-1_all.deb)

3. Copy the file to your topology host server:
```
scp vxr2-ngdp-24.11.4140.17_1-1_all.deb cisco@198.18.133.100:images/
```

4. Install the debian package
```
cd ./images/
sudo dpkg -i vxr2*.deb
```

or
```
sudo dpkg -i vxr2-ngdp-24.11.4140.17_1-1_all.deb 
```

Example output:
```
Selecting previously unselected package vxr2-ngdp-24.11.4140.17.
(Reading database ... 187672 files and directories currently installed.)
Preparing to unpack vxr2-ngdp-24.11.4140.17_1-1_all.deb ...
Unpacking vxr2-ngdp-24.11.4140.17 (1-1) ...
Setting up vxr2-ngdp-24.11.4140.17 (1-1) ...
```

5. Try running *`vxr.py start`* again
```
cd ../SONiC/01-build-your-lab/

vxr.py start topology.yaml 
```

Once the topology simulation is up and running the output will look something like this:
```
08:55:56 INFO Vxr up on host localhost
08:55:56 INFO Getting port vector files for:r1, r2, r3, r4
08:56:28 INFO Sim up
```

>[Note] it'll still take the 8000 emulator nodes a few minutes to fully boot

6. Test console access and monitor sonic node bootup:
```
telnet 0 40001
```

Example output:
```

```
