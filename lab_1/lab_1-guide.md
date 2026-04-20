# Lab 1 Guide: Learn Cisco 8000 Emulator, Image Management, Configuration Management [100 Min]

## Contents
- Lab 1 Guide: Learn VXR Overview, Image Management, Configuration Management, and  [100 Min]
  - [Contents](#contents)
  - [Lab Objectives](#lab-objectives)
  - [Topology](#topology)
  - [Cisco 8000 Emulator Overview](#cisco-8000-emulator-overview)
  - [Image Management](#image-management)
  - [System Verification](#platform-and-sonic-software-verification)
  - [Configuration Management](#configuration-management)
  - [Basic SONiC Configuration](#basic-sonic-configuration)
  - [Configuring SONiC Leaf-03](#configuring-sonic-leaf-03)
  - [End of Lab 1](#end-of-lab-1)
  


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

## Configuration Management

### Managing Configurations
Configuration state in SONiC is saved in two separate locations. For persistant configuration between reloads configuration files are used. The main configuration is found at */etc/sonic/config_db.json*. The second configuration file in this lab is for the FRR routing stack and it's configuration is found at */etc/sonic/frr/bgpd.conf*. 

When the router boots it loads the configuration from these two files into the redis database. The redis database is the running configuration of the router where the various services read or write state information into the redis database.


<img src="../drawings/redis-diagram.png" width="800" />

#### Loading configuration from JSON file

The command *config load* is used to load a configuration following the JSON schema. This command loads the configuration from the input file (if user specifies this optional filename, it will use that input file. Otherwise, it will use the default */etc/sonic/config_db.json* file as the input file) into CONFIG_DB. The configuration present in the input file overwrites the already running configuration. This command does not flush the config DB before loading the new configuration (i.e., if the configuration present in the input file is same as the current running configuration, nothing happens) If the config present in the input file is not present in running configuration, it will be added. If the config present in the input file differs (when key matches) from that of the running configuration, it will be modified as per the new values for those keys.

- Usage:
```
config load [-y|--yes] [<filename>]
```
- Example:
```
cisco@spine-01:~$ sudo config load
Load config from the file /etc/sonic/config_db.json? [y/N]: y
Running command: /usr/local/bin/sonic-cfggen -j /etc/sonic/config_db.json --write-to-db
```

#### Reloading configuration

This command is used to clear current configuration and import new configurationn from the input file or from */etc/sonic/config_db.json*. This command shall stop all services before clearing the configuration and it then restarts those services.

The command *config reload* restarts various services/containers running in the device and it takes some time to complete the command.
> **NOTE**
> If the user had logged in using SSH, users might get disconnected depending upon the new management IP address. Users need to reconnect their SSH sessions.

- Usage:
```
config reload [-y|--yes] [-l|--load-sysinfo] [<filename>] [-n|--no-service-restart] [-f|--force]
```
- Example:
```
cisco@spine-01~$ sudo config reload
Clear current config and reload config from the file /etc/sonic/config_db.json? [y/N]: y
Running command: systemctl stop dhcp_relay
Running command: systemctl stop swss
Running command: systemctl stop snmp
Warning: Stopping snmp.service, but it can still be activated by:
  snmp.timer
Running command: systemctl stop lldp
Running command: systemctl stop pmon
Running command: systemctl stop bgp
Running command: systemctl stop teamd
Running command: /usr/local/bin/sonic-cfggen -H -k Force10-Z9100-C32 --write-to-db
Running command: /usr/local/bin/sonic-cfggen -j /etc/sonic/config_db.json --write-to-db
Running command: systemctl restart hostname-config
Running command: systemctl restart interfaces-config
Timeout, server 10.11.162.42 not responding.
```

#### Saving Configuration to a File for Persistence

The command *config save* is used to save the config DB configuration into the user-specified filename or into the default /etc/sonic/config_db.json. This saves the current redis database CONFIG_DB int the configuration specified by the user. This is analogous in Cisco IOS as the command *copy run start*. 

Saved files can be transferred to remote machines for debugging. If users wants to load the configuration from this new file at any point of time, they can use "config load" command and provide this newly generated file as input. 

- Usage:
```
config save [-y|--yes] [<filename>]
```
- Example (Save configuration to /etc/sonic/config_db.json):

```
cisco@spine-01:~$ sudo config save -y
```

- Example (Save configuration to a specified file):
```
cisco@spine-01:~$ sudo config save -y /etc/sonic/config2.json
```

#### Edit Configuration Through CLI

Configuration management is also possible through the SONiC CLI. From the SONiC command prompt enter *config* and the command syntax needed. 
```
cisco@spine-01:~$ config -?
Usage: config [OPTIONS] COMMAND [ARGS]...

  SONiC command line - 'config' command
```

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

## Basic SONiC Configuration
The following configuration snippets are examples of basic router configuration using CLI all hyperscalers perform configuration management through the use of config_db changes which are then loaded into Redis database. The CLI commands here are linux commands and not executed through the *vtysh* shell.

- [Configuring Hostname](#configuring-hostname)
- [Configuring Users](#configuring-users)
- [Configuring Interface IPv4 and IPv6](#configuring-interface-ipv4-ipv6)
- [Configuring Loopback Interface](#configuring-loopback-interface)
- [Configure VLAN](#configure-vlan)
- [Configuring Port Channels](#configuring-port-channels)
  
### Configuring Hostname
```
sudo config hostname HOSTNAME ​
```
### Configuring Users
Creates the home directory (/home/username) and sets up default settings​
```
sudo adduser username​
```
Set the created user with superuser admin rights
```
sudo usermod -aG sudo username
```

### Configuring Interface IPv4 and IPv6
```
sudo config interface ip add INTERFACE_NAME INTERFACE_IPv4/MASK ​
sudo config interface ip add INTERFACE_NAME INTERFACE_IPv6/MASK
```
### Configuring Loopback Interface
```
sudo config interface ip add Loopback0 INTERFACE_IPv4/MASK
sudo config interface ip add Loopback0 INTERFACE_IPv6/MASK
```

### Configuring VRF 
```
sudo config vrf add VRF_NAME​
sudo config interface vrf bind INTERFACE_NAME VRF_NAME​
sudo config vrf add_vrf_vni_map VRF_NAME VNI_NUMBER (???)
```
### Configuring VLAN
```
sudo config vlan add VLAN_NUMBER​
sudo config interface ip add VLAN_NUMBER VLAN_IP​
sudo config vlan member add VLAN_NUMBER INTERFACE
```
### Configuring Port Channels
1. Create Port Channels to *sonic-rtr-spine-1* and *sonic-rtr-spine-2*
   ```
   sudo config portchannel add PortChannel1
   ```
2.  Configure Port Channel interface members
    ```
    sudo config portchannel member add PortChannel1 Ethernet0
    sudo config portchannel member add PortChannel1 Ethernet4
    ```
3. Configure Port Channel IPs
   ```
   sudo config interface ip add PortChannel1 INTERFACE_IPv4/MASK
   ```

## Configuring SONiC Leaf-03

### REDIS Database queries and verification of config


## End of Lab 1
Lab 1 is completed, please proceed to [Lab 2](https://github.com/cisco-asp-web/SONiC/blob/main/lab_2/lab_2-guide.md)
