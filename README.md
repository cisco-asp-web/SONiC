# Welcome to the SONiC Labs at Cisco

### Description: This repository contains lab guides, router configs, setup scripts, and other code for running through the various SONiC labs.

SONiC is an open source network operating system based on Linux that runs on switches/routers from multiple vendors and ASICs. SONiC offers a full-suite of network functionality, like BGP, that has been production-hardened in the data centers of some of the largest cloud-service providers. That said, of this writing (October 2023) SONiC may not be the best choice for all network operators. One of the purposes of this lab series is to give network engineers and operators hands on exposure to SONiC so they may get a sense for what's available today, and what challenges they might face should they pursue a real SONiC deployment.

Though it is easy to run SONiC in a virtualized lab environment, this lab brings the ability to run SONiC on a platform that emulates the Cisco 8000 hardware. This emulator allows data-plane features such as ACLs, hardware counters, and debug tools. 

The lab software stack is built off the SONiC master build with Cisco specific platform drivers for the Cisco 8000 hardware.

## Contents
* Lab Topology (TBD)
* Lab 1 - Intro to Cisco 8000 Emulator, Image Management, Config Management [LINK](https://github.com/cisco-asp-web/SONiC/blob/main/lab_1/lab_1-guide.md)
* Lab 2 - SONiC and FRR, Building a BGP Fabric [LINK](https://github.com/cisco-asp-web/SONiC/blob/main/lab_2/lab_2-guide.md)
* Lab 3 - SONiC Automation on Cisco 8000 [LINK](https://github.com/cisco-asp-web/SONiC/blob/main/lab_3/lab_3-guide.md)
* Lab 4 - SONiC ACL, CoPP, and Packet Mirroring [LINK](https://github.com/cisco-asp-web/SONiC/blob/main/lab_4/lab_4-guide.md)
* Lab 5 - SONiC and FRR, Building a VXLAN EVPN Fabric [LINK](https://github.com/cisco-asp-web/SONiC/blob/main/lab_5/lab_5-guide.md)
* Lab 6 - SONiC Building an AI Backend Network on Cisco 8000 [LINK](https://github.com/cisco-asp-web/SONiC/blob/main/lab_6/lab_6-guide.md)

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
