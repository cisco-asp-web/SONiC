# Lab 1 Guide: Learn VXR Overview, Image Management, Configuration Management [100 Min]

We use containerlab to orchestrate the VXR and SONiC virtual network topologies. 

For more information on containerlab see:

https://containerlab.dev/


## Contents
- Lab 1 Guide: Learn VXR Overview, Image Management, Configuration Management, and  [100 Min]
  - [Contents](#contents)
  - [Lab Objectives](#lab-objectives)
  - [Topology](#topology)
  - [VXR Overview](#vxr-overview)
  - [Image Management](#image-management)
  - [System Verification](#platform-and-sonic-software-verification)
  - [Basic SONiC Configuration](#basic-sonic-configuration)
  - [Configuration Management](#configuration-management)
  - [End of Lab 1](#end-of-lab-1)
  
## Lab Objectives
We will have achieved the following objectives upon completion of Lab 1:

* Access all devices in the lab
* Deployed the full network topology (XRd frontend, SONiC backend)
* Basic familiarity with containerlab
* Confirm IPv4 and IPv6 connectivity
* Familiarity with base SRv6 configuration 


## Topology

For Labs 1-6 you will be using a single topology as outlined below. We will have four SONiC routers running in a two tier fabric with host containers connected to each leaf. 

<img src="../drawings/topology-base-view.png" width="800" />


## VXR Overview

The Cisco 8000 Hardware Emulator portfolio (referred to as 8000e) provides one for one equivalent simulation of the 8xxx Series routers. The 8000e provides both accurate hardware profile and forwarding engine emulation. This enables 8000e to run the same productio IOS-XR images as hardware. Secondly, it can run third party Operating Systems such as SONIC which have been ported to the 8000 series routers. For this lab we will be using the VXR option to load the SONiC NOS

## Image Management

### SONiC Installation on Cisco 8000

### SONiC Upgrade and Downgrade Process

### Upgrade Mechanism: Warm Reboot, Express Boot and Fast Reboot

### Image Verification

### Reference Commands


## System Verification 

### Platform and SONiC Software Verification

### Container Verification

### Platform and Software Health Checks

## Basic SONiC Configuration

### Configuring Hostname

### Configuring Users

### Configuring Interface IPv4 and IPv6

### Configuring Loopback Interface

### Configuring VLAN 

## Configuration Management

### REDIS Database queries and verification of config
## End of Lab 1
Lab 1 is completed, please proceed to [Lab 2](https://github.com/cisco-asp-web/SONiC/blob/main/lab_2/lab_2-guide.md)
