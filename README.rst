.. Copyright 2018 Intel Corporation.
   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at
        http://www.apache.org/licenses/LICENSE-2.0
   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.

=================
OVN4NFVK8s Plugin
=================

Problem statement
-----------------

Networking applications are of three types - Management applications,
Control plane applications and  data plane applications. Management
and control plane applications are similar to Enterprise applications,
but data plane applications different in following aspects:

- Multiple virtual network interfaces
- Multiple IP addresses
- SRIOV networking support
- Programmable virtual switch (for service function chaining, to tap
  the traffic for visibility etc..)

Kubernetes (Simply K8S) is the most popular container orchestrator.
K8S is supported by GCE, AZURE and AWS and will be supported by
Akraino Edge stack that enable edge clouds.

K8S has being enhanced to support VM workload types, this helps
cloud providers that need to migrate legacy workloads to microservices
architecture. Cloud providers may continue to support VM workload
types for security reasons and hence there is need for VIM that
support both VMs and containers. Since same K8S instance can
orchestrate both VM and container workload types, same compute nodes
can be leveraged for both VMs and containers. Telco and CSPs are
seeing similar need to deploy networking applications as containers.

Since, both VMs and container workloads are used for networking
applications, there would be need for

- Sharing the networks across VMs and containers.
- Sharing the volumes across VMs and containers.

**Network Function Virtualization Requirements**

NFV workloads can be,

- Management plane workloads
- Control plane work loads
- User plane (data plane workloads)
- User plane workloads normally have
- Multiple interfaces, Multiple subnets, Multiple virtual networks
- NFV workloads typically have its own management network.
- Some data plane workloads require SR-IOV NIC support for data
  interfaces and virtual NIC for other interfaces (for performance
  reasons)
- Need for multiple CNIs.
- NFV workloads require dynamic creation of virtual networks. Dynamic
  configuration of subnets.

New Proposal
------------

A new plugin addressing the below requirements,

- For networking workloads as well typical application workloads
- Multi-interface support
- Multi-IP address support
- Dynamic creation of virtual networks
- Co-existing with SRIOV and other CNIs.
- Route management across virtual networks and external networks

**OVN Background**

OVN, the Open Virtual Network, is a system to support virtual network
abstraction. OVN complements the existing capabilities of OVS to add
native support for virtual network abstractions, such as virtual L2
and L3 overlays and security groups. Services such as DHCP are also
desirable features. Just like OVS, OVN’s design goal is to have a
production quality implementation that can operate at significant
scale. 

**K8S-OVN4NFV Plugin development**

Some code and ideas are being taken from ovn-kubernetes_ plugin
work that was done as part of OVN project.  Due to good number of
changes, it is a new plugin with its own code base.  This plugin
assumes that the first interface in a Pod is provided by some other
Plugin/CNI like Flannel or even OVN-Kubernetes and this plugin is
only responsible to add multiple interfaces based on the Pod
annotations. This plugin is currently tested to work with Multus as
CNI and Flannel as first interface.

Its functionality is divided into to following:

- Initialization:

  - Register itself as watcher to K8S API Server to receive POD events
    and service events.
  - Creates a distributed router
  - Creates gateway
  - Creates a logical switch to connect distributed router with
    Gateway.
  - Creates a subnet between distributed router & Gateway.
  - Assigns first two IP addresses of the subnet to router and
    Gateway.
  - Created router port and gateway port as part of assigning IP
    address and MAC addresses.

- Watcher:

  - Upon POD bring up event

    - Checks the annotations specific to OVN.
    - For each network on which POD is going to be brought up
    - Validates whether the logical switch is already present. If not,
      it is considered as error.
    - If IP address and MAC addresses are not static, it asks OVN to
      assign IP and MAC address.
    - Collects all IP addresses/MAC addresses assigned. Puts them as
      annotations (dynamic information) for that POD.

  - Upon POD deletion event

    - Returns the IP address and MAC address back to OVN pool.

- OVN CNI

This is present in every minion node. CNI is expected to be called
once for all OVN networks either Kubelet directly or via Multus.

  - Add:

    - Wait for annotations to be filled up by the watcher. From
      annotations, it knows set  of IP Address, MAC address and Routes
      to be added.
    - Using network APIs for each element in the set:
    - Creates veth pair.
    - Assigns the IP address and MAC address to one end of veth pair.
      Other end veth pair is assigned to br-int.
    - Creates routes based on the route list provided in annotations.

  - If isDefaultRoute is set in annotations, it creates default route
    using this veth.
  - Delete

    - Removes veth pair.
    - Removes routes.

**Figure**

.. code-block:: raw

    +-----------------+
    |                 |
    |                 |   Program OVN Switch
    |ovn4nfvk8s Plugin|                      +------------------+
    |                 +--------------------->|                  |
    |                 |                      | OVN Switch       |
    |                 |                      |                  |
    |                 |                      +------------------+
    +----+----------+-+
         ^          |
         |          |
         |On Event  |Annotate Pod
         |          |
         |          v
    +----+--------------+        +------------------+           +-----------+
    |                   |        |                  |           | Pod       |
    |  Kube API         +-------->  Kube Scheduler  +---------->|           |
    |                   |        |                  |           +--------+--+
    |                   |        +--------+---------+                    |
    +-------------------+                 |                              |
                                          |                              |
                                          |                              |Assign IP & MAC
                                 +--------v-----------+                  |
                                 |                    |                  |
                                 | ovn4nfvk8s-cni     |                  |
                                 |                    +------------------+
                                 +--------------------+


   Complete Architecture can be found in ovn-kubernetes documentation at github


**References**

.. _ovn-kubernetes: https://wiki.opnfv.org/display/OV/K8S+OVN+NFV+Plugin

**Authors/Contributors**

Addepalli, Srinivasa R <srinivasa.r.addepalli@intel.com>
Sood, Ritu <ritu.sood@intel.com>