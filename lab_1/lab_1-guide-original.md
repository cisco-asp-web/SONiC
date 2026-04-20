# Lab 1 Guide: Learn Cisco 8000 Emulator, Image Management, Configuration Management [100 Min]

## Contents
- Lab 1 Guide: Learn VXR Overview, Image Management, Configuration Management, and  [100 Min]
  - [Contents](#contents)
  - [Lab Objectives](#lab-objectives)
  - [Topology](#topology)
  - [Cisco 8000 Emulator Overview](#cisco-8000-emulator-overview)
  - [Image Management](#image-management)
  - [End of Lab 1](#end-of-lab-1)
  


## Image Management

### SONiC Installation on Cisco 8000

### SONiC Upgrade and Downgrade Process

### Image Verification

### Reference Commands


#### FRR Configuration Management

FRR is an open-source routing stack that supports multiple protocols. In this lab we will focus on BGP routing protocol. 

First FRR stores it's configuration in a separate file located at */etc/sonic/frr/bgpd.conf*. There are different methods to manage the configuration for FRR.

<img src="../drawings/frr-bgp-framework.png" width="800" />

We can view FRR's BGP configuration from the SONiC CLI itself

**View Startup FRR BGP Configuration**
```
show startupconfiguration bgp
```

**View Running FRR Configuration**
```
show run bgp
```
**Save Running FRR Configuration to File**
For direct FRR configuration you use the *vtysh* command which drops you into the FRR command shell. This shell has a more of a router CLI command feel with show commands, config terminal, and config save commands. The below command drops you into FRR and tells FRR to copy the running config and save it to file.

```
vtysh
copy run start
```

### Configuring Hostname
```
sudo config hostname HOSTNAME ​
```

### REDIS Database queries and verification of config


## End of Lab 1
Lab 1 is completed, please proceed to [Lab 2](https://github.com/cisco-asp-web/SONiC/blob/main/lab_2/lab_2-guide.md)
