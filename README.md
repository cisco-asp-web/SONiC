# Welcome to the SONiC Labs at Cisco

### Description: This repository contains lab guides, router configs, setup scripts, and other code for running through the various SONiC labs.

SONiC is an open source network operating system based on Linux that runs on switches/routers from multiple vendors and ASICs. SONiC offers a full-suite of network functionality, like BGP, that has been production-hardened in the data centers of some of the largest cloud-service providers. That said, SONiC may not be the best choice for all network operators. One of the purposes of this lab series is to give network engineers and operators hands on exposure to SONiC so they may get a sense for what's available today, and what challenges they might face should they pursue a real SONiC deployment.

Though it is easy to run SONiC in a virtualized lab environment (sonic-vs, containerlab, etc.), this lab leverages the Cisco 8000 hardware emulator, which supports data-plane features such as ACLs, hardware counters, and debug tools. 


## Contents
* Docker 101/202 Guide - A quick primer on Docker concepts for this lab [LINK](./docker_101_202.md)

### SONiC Workshop #1
* Lab Topology (TBD)
* Lab 0 - Connecting to your lab pod and accessing SONiC nodes [LINK](./lab_0/lab_0_Getting_started.md)
* Lab 1 - Intro to Cisco 8000 Emulator, Image Management, Config Management [LINK](./lab_1/lab_1-guide.md)
* Lab 2 - SONiC and FRR, Building a BGP Fabric [LINK](./lab_2/lab_2-guide.md)
* Lab 3 - SONiC Automation on Cisco 8000 [LINK](./lab_3/lab_3-guide.md)
* Lab 4 - SONiC ACL, CoPP, and Packet Mirroring [LINK](./lab_4/lab_4-guide.md)

### SONiC Workshop #2
* Lab 5 - SONiC and FRR, Building a VXLAN EVPN Fabric [LINK](./lab_5/lab_5-guide.md)
* Lab 6 - SONiC Building an AI Backend Network on Cisco 8000 [LINK](./lab_6/lab_6-guide.md)
* Lab 7 - SONiC and troubleshooting optical transceivers 
* Lab 8 - SONiC Advanced Health Checks and Troubleshooting
  
## Github Repository Overview
Each of the labs is designed to be completed in the order presented.

### Root Directory

| File Name      | Description                                                          |
|:---------------|:---------------------------------------------------------------------|
| infrastructure | Base configurations for the non-router containers and VMs in the lab |
| drawings       | Lab diagrams folder                                                  |
| lab_1 -> lab_5 | Individual lab folders                                               |


### Individual Lab Directories
Within each lab directory you should see several files of importance:
(X = lab #)

| File Name                | Description                                                  |
|:-------------------------|:-------------------------------------------------------------|
| configs                  | Folder containing the configuration for the completed lab X  |
| lab_X-guide.md           | User guide for this lab                                      |

**Please proceed to Lab_0 to get started**: [LINK](./lab_0/lab_0_Getting_started.md)