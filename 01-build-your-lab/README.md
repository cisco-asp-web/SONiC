## Building a SONiC Cisco 8000 Emulator lab with VXR 

A quick guide to acquiring your SONiC 8000 emulator images and installing the VXR orchestrator tool

### Contents
- [Building a SONiC Cisco 8000 Emulator lab with VXR](#building-a-sonic-cisco-8000-emulator-lab-with-vxr)
  - [Contents](#contents)
  - [Cisco dCloud Overview](#cisco-dcloud-overview)
  - [Acquire VXR Package and SONiC Images](#acquire-vxr-package-and-sonic-images)
  - [Install VXR](#install-vxr)
  - [VXR Topology File](#vxr-topology-file)


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

### VXR Topology File

Similar to tools like containerlab, VXR uses a yaml file for defining virtual network topology and node parameters. [topology file for our project](./topology.yaml)

1. git clone this repo to your topology host
```
git clone https://github.com/cisco-asp-web/SONiC.git
```

2. Optional: move or copy the SONiC *`onie`* and *`sonic.bin`* files to a directory where you plan to do your work

```
ls images/8000-eft17.0/packages/images/8000/sonic/
```

```
cp images/8000-eft17.0/packages/images/8000/sonic/onie-recovery-x86_64-cisco_8000-r0.iso SONiC/01-build-your-lab/

cp images/8000-eft17.0/packages/images/8000/sonic/sonic-cisco-8000.bin SONiC/01-build-your-lab/
```

3. Construct VXR topology file
