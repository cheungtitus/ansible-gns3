## Overview
This playbook enables declaring a network topology in code and scaffold on GNS3.  This facilitates a consistent 
and repeatable set of network tests to be done with Continuous Integration.  

## Design Principles
* Declare all items (GNS3 project, network switches, links, console and SSH ports, etc) in code.
* Specification of device locations representative of physical topology such as data centers, racks, business units.
* Avoid manual placement by allowing declaration of exact placement of devices on GNS3 canvas.
* Allow specifying which device templates (ie switch image) to use for creating nodes.
* Setup base configurations using telnet through GNS3 followed by a detailed setup using SSH through the Cloud node and management switches (work in progress).
* Support the use of multiple remote GNS3 servers (feature to come) to run resource intensive nodes (ie NX-9000v).
* Specification of test cases to assert network functionality subjected to different failure scenarios (feature to come).

## Limitation
Only devices of the following type can be supported at the moment:
* qemu
* ethernet_switch
* cloud

## Configuration Approach
#### Telnet via GNS3
This is the first and only way to configure all devices after network is scaffolded.  This is similar having remote hands applying hostnames and management IPs to each switch after power up.
The `roles/ansible-template/tasks/initialize.yml` would apply this initial setting.

#### SSH via management network
This is representative to how the network will be setup or updated normally using SSH.  Most of the settings will be project specific and is not included in this repo.
It is important that the Ansible Controller sits physically on the same network as the ESXi server as going through additional firewall or NAT devices may cause issues for reaching to the management spines.

###### ESXi vSwitch Setting
:warning: Because the First Hop Router (shown on diagram below) sits behind the GNS3 eth0 interface (eth0 on diagram below), special settings need to be applied on the ESXi vSwitch and the Port Group the GNS3 VM is connected to:
- Allow promiscuous mode: ~~No~~ Yes
- Allow forged transmits: ~~No~~ Yes
- Allow MAC changes: No

```<!-- language: lang-none -->
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|                                   |---------------------------------------------------------------------------------------------------------------------------------------|  |
|                                   |                                                            ESXi                                                                       |  |
|                                   |                 |------------------------------------------------------------------------------------------------------------------|  |  |
|                                   |                 |                                          GNS3                                                                    |  |  |    
|                                   |                 |             |-------------------------------------------------------------------------------------------------|  |  |  |
|                                   |                 |             |                        GNS3 Project                                                             |  |  |  |
|                                   |                 |             |                                                                                                 |  |  |  |
|                                   |                 |             |                                    /--> Management Spine (Data Center 1) ---> Data Spine & Leaf |  |  |  |
|  WSL2 / Server                    |                 | eth0        |              First       Switch   /                                                             |  |  |  |
|   (Ansible   ----> Physical ----> | vSwitch  ---->  |  on   ----> | Cloud  ---->  Hop  ----> (Carrier ----> Management Spine (Data Center 2) ---> Data Spine & Leaf |  |  |  |
|   Controller)      Network        | (Special        | GNS3        | Node         Router      Network) \                                                             |  |  |  |
|                                   |  Setting        |             |                                    \--> Management Spine (Data Center N) ---> Data Spine & Leaf |  |  |  |
|                                   |  Required)      |             |-------------------------------------------------------------------------------------------------|  |  |  |
|                                   |                 |------------------------------------------------------------------------------------------------------------------|  |  |
|                                   |---------------------------------------------------------------------------------------------------------------------------------------|  |
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
```
## Example Usage
#### Configuration
##### Passwords
> roles/ansible-template/vars/credentials.yml

All sensitive data should go into this file and encrypted using Ansible vault.  At a minimum the following GNS3 credentials should be encrypted:
```txt
username: <GNS3 username>
password: <GNS3 password>
username_nodes: <switch username>
password_nodes: <switch password> 
```
##### Links
> roles/ansible-template/vars/main.yml

This is where all connections between devices (ie patching schedule) is to be declared.
```yaml
links:
# Patch cord 1
- a-end:
    device: cloud
    port: eth0
  b-end:
    device: switch
    port: Ethernet0
# Patch cord 2
- a-end:
    device: switch
    port: Ethernet1
  b-end:
    device: firsthoprouter
    port: Gi0/0
```
##### Base setting
> roles/ansible-template/tasks/initialize.yml

This is where the base settings are to be applied via telnet exposed via GNS3. Settings such as enabling SSH, hostname, no ip domain-lookup, logging synchronous, etc can be done here.

##### Inventory
> inventory-switches

This is where we declare our network topology.
```yaml
all:
  vars:
    project_template:
      name: ansible-template
      path: /opt/gns3/projects/ansible-template
  children:
    gns3:
      hosts:
        # GNS3 master server
        master:
          ip: 20.0.1.1
          port: 3080
        # Enable connectivity between devices in GNS3 and physical network using SSH
        cloud:
          template: Cloud
          node_type: cloud
          symbol: :/symbols/cloud.svg
          properties:
            interfaces:
            - name: docker0
              special: true
              type: ethernet
            - name: eth0
              special: false
              type: ethernet
            - name: lo
              special: true
              type: ethernet
            - name: virbr0
              special: true
              type: ethernet
            - name: virbr0-nic
              special: true
              type: ethernet
            ports_mapping:
            - interface: eth0
              name: eth0
              port_number: 0
              type: ethernet
            remote_console_host: ''
            remote_console_http_path: "/"
            remote_console_port: 23
            remote_console_type: none
          position:
            x: -109
            y: 209
            z: 1
        # Gateway for connecting from physical network into GNS3 using SSH
        firsthoprouter:
          console:
            ip: 20.0.1.1
            port: 5001
          ssh: 20.0.1.13
          template: Cisco IOSvL2 15.2(20170321:233949)
          node_type: qemu
          symbol: :/symbols/multilayer_switch.svg
          port_name_format: Gi{1}/{0}
          properties:
            adapter_type: e1000
            adapters: 16
            qemu_path: "/usr/bin/qemu-system-x86_64"
          position:
            x: -55
            y: 10
            z: 1
        # Simulate L2 carrier network servicing management connectivity between data centers
        switch:
          console:
            ip: 20.0.1.1
            port: 5002
          ssh: 20.0.1.14
          template: Ethernet switch
          node_type: ethernet_switch
          symbol: :/symbols/cloud.svg
          port_name_format: Ethernet{0}
          properties:
            ports_mapping:
            - name: Ethernet0
              port_number: 0
              type: access
              vlan: 1
            - name: Ethernet1
              port_number: 1
              type: access
              vlan: 1
            - name: Ethernet2
              port_number: 2
              type: access
              vlan: 1
            - name: Ethernet3
              port_number: 3
              type: access
              vlan: 1
            - name: Ethernet4
              port_number: 4
              type: access
              vlan: 1
          position:
            x: -65
            y: 100
            z: 1
    # All settings belonging to a data center goes here
    datacenter1:
      children:
        # Switches belonging to a rack goes here
        rack1000:
          # Common settings for the rack goes here
          vars:
            unit: shared
            node_type: qemu
            symbol: :/symbols/multilayer_switch.svg
            port_name_format: Gi{1}/{0}
            properties:
              adapter_type: e1000
              adapters: 16
              qemu_path: "/usr/bin/qemu-system-x86_64"
          hosts:
            # Management network spine 1
            DC1R1000MS1:
              console:
                ip: 20.0.1.1
                port: 5011
              ssh: 10.0.1.4
              template: Cisco IOSvL2 15.2(20170321:233949)
              type: mgmt-spine-switches
              position:
                x: -144
                y: -89
                z: 1
            # Management network spine 2
            DC1R1000MS2:
              console:
                ip: 20.0.1.1
                port: 5022
              ssh: 10.0.1.5
              template: Cisco IOSvL2 15.2(20170321:233949)
              type: mgmt-spine-switches
              position:
                x: 44
                y: -89
                z: 1
            # Data network spine 1
            DC1R1000DS1:
              console:
                ip: 20.0.1.1
                port: 5013
              ssh: 10.0.1.6
              template: Cisco IOSvL2 15.2(20170321:233949)
              type: data-spine-switches
              position:
                x: -144
                y: -247
                z: 1
            # Data network spine 2
            DC1R1000DS2:
              console:
                ip: 20.0.1.1
                port: 5014
              ssh: 10.0.1.7
              template: Cisco IOSvL2 15.2(20170321:233949)
              type: data-spine-switches
              position:
                x: 44
                y: -247
                z: 1
        # Switches belonging to another rack goes here
        rack1001:
          # Common settings for the rack goes here
          vars:
            unit: businessunit1
            node_type: qemu
            symbol: :/symbols/multilayer_switch.svg
            port_name_format: Gi{1}/{0}
            properties:
              adapter_type: e1000
              adapters: 16
              qemu_path: "/usr/bin/qemu-system-x86_64"
          hosts:
            # Top of rack management leaf switch
            DC1R1001ML1:
              console:
                ip: 20.0.1.1
                port: 5111
              ssh: 10.1.1.4
              template: Cisco IOSvL2 15.2(20170321:233949)
              type: mgmt-leaf-switches
            # Top of rack data leaf switch 1
            DC1R1001DL1:
              console:
                ip: 20.0.1.1
                port: 5112
              ssh: 10.1.1.5
              template: Cisco IOSvL2 15.2(20170321:233949)
              type: data-leaf-switches
            # Top of rack data leaf switch 2
            DC1R1001DL2:
              console:
                ip: 20.0.1.1
                port: 5113
              ssh: 10.1.1.6
              template: Cisco IOSvL2 15.2(20170321:233949)
              type: data-leaf-switches
```

## Run Playbooks
#### Encrypt credentials
Encrypt credentials.yml
Follow official guide at https://docs.ansible.com/ansible/latest/user_guide/vault.html

##### Playbook
Scaffold topology on GNS3.
```bash
ansible-playbook -i inventory-switches network.yaml --ask-vault
```

Apply base configurations using telnet via GNS3 and using SSH via Cloud node.
```bash
Details to come.
```

##### Topology
View topology.
```bash
ansible-inventory -i inventory-switches --graph
@all:
  |--@datacenter1:
  |  |--@rack1000:
  |  |  |--DC1R1000DS1
  |  |  |--DC1R1000DS2
  |  |  |--DC1R1000MS1
  |  |  |--DC1R1000MS2
  |  |--@rack1001:
  |  |  |--DC1R1001DL1
  |  |  |--DC1R1001DL2
  |  |  |--DC1R1001ML1
  |--@datacenter2:
  |  |--@rack2000:
  |  |  |--DC2R2000DS1
  |  |  |--DC2R2000DS2
  |  |  |--DC2R2000MS1
  |  |  |--DC2R2000MS2
  |  |--@rack2001:
  |  |  |--DC2R2001DL1
  |  |  |--DC2R2001DL2
  |  |  |--DC2R2001ML1
  |--@gns3:
  |  |--cloud
  |  |--firsthoprouter
  |  |--master
  |  |--switch
  |--@ungrouped:
```

# License
Apache License 2.0
