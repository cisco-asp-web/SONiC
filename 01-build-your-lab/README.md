## Building a SONiC Cisco 8000 Emulator lab with VXR 

A quick guide to acquiring your SONiC 8000 emulator images and installing the VXR orchestrator tool

## Contents
* Overview of Cisco dCloud [LINK](#cisco-dcloud-overview)
* Acquire VXR Package and SONiC Images [LINK](#acquire-vxr-package-and-sonic-images)


## Cisco dCloud Overview
dCloud is Cisco's cloud based demo platform that showcases a large catalog of demos, training and sandboxes for every Cisco architecture. This is the chosen platform to demonstrate our SONiC labs as they will be available through Cisco's global DC infrastructure.

For each lab within this repository you will find a corresponding lab in dCloud. You will need a dCloud account in order to access these labs. You can login to dCloud at https://dcloud.cisco.com

Once you have scheduled a lab in dCloud Cisco AnyConnect VPN connection information will be provided for the lab instance. Once the dCloud session is deployed, all lab guides, configs, diagrams, and code are here in this github repository.

For instructions to build your own dCloud instance to host a SONiC 8000 emulator topology see [BYO dCloud Guide](https://github.com/brmcdoug/byo-dcloud/blob/main/Lab-Guide-for-BYO-dCloud-Lab.pdf)

## Acquire VXR Package and SONiC Images

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

5. untar both packages, 8000-emulator first:
```
tar -xvf 8000-emulator-eft17.0.tar
```

```
tar -xvf 8000-sonic-eft17.0.tar 
```

6. cd into the 8000-eft / scripts directory and run the ubuntu installation script
```

```