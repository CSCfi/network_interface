[![Build Status](https://travis-ci.org/CSCfi/network_interface.svg?branch=master)](https://travis-ci.org/CSCfi/network_interface)

network_interface
=================

_WARNING: This role can be dangerous to use. If you lose network connectivity
to your target host by incorrectly configuring your networking, you may be
unable to recover without physical access to the machine._

This roles enables users to configure various network components on target
machines. The role can be used to configure:

- Ethernet interfaces
- Bridge interfaces
- Bonded interfaces
- VLAN tagged interfaces
- Network routes
- Bonding Kernel Module parameters

Requirements
------------

This role requires Ansible 1.4 or higher, and platform requirements are listed
in the metadata file.

Role Variables
--------------

The variables that can be passed to this role and a brief description about
them are as follows:

    # The list of ethernet interfaces to be added to the system
    network_ether_interfaces: []

    # The list of bridge interfaces to be added to the system
    network_bridge_interfaces: []

    # The list of bonded interfaces to be added to the system
    network_bond_interfaces: []

    # The list of vlan interfaces to be added to the system
    network_vlan_interfaces: []

Note: The values for the list are listed in the examples below.

Examples
--------

1) Configure eth1 and eth2 on a host with a static IP and a dhcp IP. Also
define static routes and a gateway.
```
    - hosts: myhost
      roles:
        - role: network
          network_ether_interfaces:
            - device: eth1
              bootproto: static
              address: 192.168.10.18
              netmask: 255.255.255.0
              gateway: 192.168.10.1
              route:
                - network: 192.168.200.0
                  netmask: 255.255.255.0
                  gateway: 192.168.10.1
                - network: 192.168.100.0
                  netmask: 255.255.255.0
                  gateway: 192.168.10.1
            - device: eth2
              bootproto: dhcp
```

2) Configure a bridge interface with multiple NIcs added to the bridge.
Also set bridging_opts, an example in case you need multicasting.
```
    - hosts: myhost
      roles:
        - role: network
          network_bridge_interfaces:
            - device: br1
              type: bridge
              address: 192.168.10.10
              netmask: 255.255.255.0
              gateway: 192.168.10.1
              bootproto: static
              stp: "on"
              bridging_opts: "multicast_snooping=1 multicast_querier=1"
          network_ether_interfaces:
            - device: eth1
              bootproto: none
              onboot: yes
              bridge: br1
```

Note: Routes can also be added for this interface in the same way routes are
added for ethernet interfaces.

3) Configure a bond-ext interface with an "active-backup" slave configuration.
Also set max_bonds=0 parameter to the bonding kernel module. Preventing
an empty bond0 from being created.
```
    - hosts: myhost
      roles:
        - role: network
          network_extra_bonding_module_options: "max_bonds=0"
          network_bond_interfaces:
            - device: bond-ext
              address: 192.168.10.128
              netmask: 255.255.255.0
              bootproto: static
              bond_mode: active-backup
              bond_miimon: 100
              bond_slaves: [eth1, eth2]
              route:
                - network: 192.168.222.0
                  netmask: 255.255.255.0
                  gateway: 192.168.10.1
```

4) Configure a bonded interface with "802.3ad" as the bonding mode and IP
address obtained via DHCP.
```
    - hosts: myhost
      roles:
        - role: network
          network_bond_interfaces:
            - device: bond0
              bootproto: dhcp
              bond_mode: 802.3ad
              bond_miimon: 100
              bond_slaves: [eth1, eth2]
```

5) Configure a VLAN interface with the vlan tag 2 for an ethernet interface
and set nozeroconf to True (no 169.254.0.0/16 link local address).

```
    - hosts: myhost
      roles:
        - role: network
          network_ether_interfaces:
            - device: eth1
              bootproto: static
              nozeroconf: True
              address: 192.168.10.18
              netmask: 255.255.255.0
              gateway: 192.168.10.1
          network_vlan_interfaces:
	          - device: eth1.2
	            bootproto: static
	            address: 192.168.20.18
	            netmask: 255.255.255.0
```
6) Configure Bond interface over vlan tagged 10001 interfaces with extra route.
important part is define that slaves for the bond are vlan slaves
and bond carrier detection is not using physical interface link status. This feature is extremely usable whenever one needs better utilization of physical interfaces. two physical interfaces can support 2 or more different bonded interfaces  that have different tagged vlans. This means that for example public and private networks ( 2 different tagged vlans) can be separated  to different physical interfaces for communication and in failure state, fail over to available interface. 

```
- hosts: myhost
  roles:
    - role: network
      network_extra_bonding_module_options: "use_carrier=0"
      network_bond_interfaces:
        - device: bond-ext
          address: 192.168.10.128
          netmask: 255.255.255.0
          bootproto: static
          bond_mode: active-backup
          onboot: "yes"
          nm_controlled: "no"
          mtu: 9000
          bond_miimon: 500
          bond_slaves: [eth1.10001, eth2.10001]
          route:
            - network: 192.168.222.0
              netmask: 255.255.255.0
              gateway: 192.168.10.1

```

7) All the above examples show how to configure a single host, The below
example shows how to define your network configurations for all your machines.

Assume your host inventory is as follows:

### /etc/ansible/hosts

    [dc1]
    host1
    host2

Describe your network configuration for each host in host vars:

### host_vars/host1

    network_ether_interfaces:
           - device: eth1
             bootproto: static
             nozeroconf: True
             address: 192.168.10.18
             netmask: 255.255.255.0
             gateway: 192.168.10.1
             route:
              - network: 192.168.200.0
                netmask: 255.255.255.0
                gateway: 192.168.10.1
    network_bond_interfaces:
            - device: bond0
              bootproto: dhcp
              bond_mode: 802.3ad
              bond_miimon: 100
              bond_slaves: [eth2, eth3]

### host_vars/host2

    network_ether_interfaces:
           - device: eth0
             bootproto: static
             address: 192.168.10.18
             netmask: 255.255.255.0
             gateway: 192.168.10.1

Create a playbook which applies this role to all hosts as shown below, and run
the playbook. All the servers should have their network interfaces configured
and routed updated.

    - hosts: all
      roles:
        - role: network

Note: Ansible needs network connectivity throughout the playbook process, you
may need to have a control interface that you do *not* modify using this
method so that Ansible has a stable connection to configure the target
systems.


Dependencies
------------

None

License
-------

BSD

Author Information
------------------

Benno Joy
