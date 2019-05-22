=============================================
Container Management in Multi-Region Scenario
=============================================

Background
==========

Currently, multi-region container management is not supported in the Tricircle.
This spec is to describe how container management will be implemented
in the Tricircle multi-region scenario. Now openstack provides many components
for container services such as zun,kuyr,kuryr-libnetwork. Zun is a component that
provides container management service in openstack, it provides a unified OpenStack API
for launching and managing containers, supporting docker container technology.
Kuryr is an component that interfaces a container network to a neutron network.
Kuryr-libnetwork is a kuryr plugin running under the libnetwork framework and provides
network services for containers. Zun integrates with keystone, neutron,
and glance to implement container management. Keystone provides identity authentication
for containers, neutron provides network for containers, and glance provides images for containers.
These openstack services work together to accomplish the multi-region container management.

Overall Implementation
======================

The Tricircle is designed in a Central_Neutron-Local_Neutron fashion, where all the local neutrons are
managed by the central neutron. As a result, in order to adapt the Central_Neutron-Local_Neutron design and
the container network requirements and image requirements, we plan to deploy zun, kuryr,kuryr-libnetwork and
raw docker engine as follows. ::

 +--------------------------------------------------+                     +--------------------------------------------------+
 |                                                  |    Central Region   |                                                  |
 |        +--------+                             +---------------------------+                             +--------+        |
 |  +-----| Glance |            Zun Request      |          Keystone         |      Zun Request            | Glance |-----+  |
 |  |     +--------+            +---------+      +---------------------------+      +---------+            +--------+     |  |
 |  |                                |           |       Central Neutron     |           |                                |  |
 |  |  +---------------+             |           +-------^-----------^-------+           |             +---------------+  |  |
 |  |  |   Zun  API    |<------------+              |    |           |    |              +------------>|   Zun  API    |  |  |
 |  |  +---------------+        +---------------+   |    |           |    |   +---------------+        +---------------+  |  |
 |  |  |               |        |               |   |    |           |    |   |               |        |               |  |  |
 |  +--+  Zun Compute  +--------+ Docker Engine |   |    |           |    |   | Docker Engine +--------+  Zun Compute  +--+  |
 |     |               |        |               |   |    |           |    |   |               |        |               |     |
 |     +-------+-------+        +-------+-------+   |    |           |    |   +-------+-------+        +-------+-------+     |
 |             |                        |           |    |           |    |           |                        |             |
 |             |                        |           |    |           |    |           |                        |             |
 |     +-------+-------+        +-------+-------+   |    |           |    |   +-------+-------+        +-------+-------+     |
 |     |               |        |               |   |    |           |    |   |               |        |               |     |
 |     | Local Neutron +--------+     Kuryr     |   |    |           |    |   |     Kuryr     <--------> Local Neutron |     |
 |     |               |        |  libnetwork   |   |    |           |    |   |  libnetwork   |        |               |     |
 |     +-------+-------+        +---------------+   |    |           |    |   +---------------+        +-------+-------+     |
 |             |                                    |    |           |    |                                    |             |
 |             +------------------------------------×----+           +----×------------------------------------+             |
 |                                                  |                     |                                                  |
 +--------------------------------------------------+                     +--------------------------------------------------+
                      Region One                                                               Region Two

                                Fig. 1 The multi-region container management architecture.

As showned in the Fig. 1 above, in Tricircle, each region has already installed
a local neutron. In order to accomplish container management in Tricircle,
admins need to configure and install zun,docker,kuryr and kuryr-libnetwork.
Under the Central_Neutron-Local_Neutron scenario, we plan to let zun employ
the central neutron in Central Region to manage networking resources, meanwhile
still employ docker engine in its own region to manage docker container and docker network.
Then, use kuryr/kuryr-libnetwork to connect the container network to the neutron network.
Hence, the workflow of container creation in Tricircle can be described as follows. ::

 +------------------------------------------------------------------------------------------------------------------------------------------------------+
 |                                                         +---------------+    +---------------+    +-----------------+    +-------------------------+ |
 |                                                     +-->| NeutronClient | -->| Local Neutron | -->| Central Neutron | -->|Neutron network and port | |
 |                                                     |   +---------------+    +---------------+    +-----------------+    +-------------^-----------+ |
 |                                                     |                                                                                  |             |
 | +-------------+    +---------+    +-------------+   |   +------------------+    +------------------------+                             |             |
 | | Zun Request | -->| Zun API | -->| Zun Compute | --+-->| kuryr/libnetwork | -->|Docker network and port |<------------Binding---------+             |
 | +-------------+    +---------+    +-------------+   |   +------------------+    +------------------------+                              \            |
 |                                                     |                                                                                    +           |
 |                                                     |   +--------------+    +--------------+                                             |           |
 |                                                     +-->| GlanceClient | -->| docker image |                                       +-----------+     |
 |                                                     |   +--------------+    +------+-------+                                       | Container |     |
 |                                                     |                              |                                               +-----------+     |
 |                                                     |   +------------+    +--------V--------+                                            |           |
 |                                                     +-->| Docker API | -->| docker instance | -------------------------------------------+           |
 |                                                         +------------+    +-----------------+                                                        |
 +------------------------------------------------------------------------------------------------------------------------------------------------------+
                                              Fig. 2 The multi-region container creation workflow.

Specifically, when a tenant attempts to create a load balancer, he/she needs to
send a request to the local neutron-lbaas service. The service plugin of
neutron-lbaas then prepares for creating the load balancer, including
creating port via local plugin, inserting the info of the port into the
database, and so on. Next the service plugin triggers the creating function
of the corresponding driver of Octavia, i.e.,
Octavia.network.drivers.neutron.AllowedAddressPairsDriver to create the
amphora. During the creation, Octavia employs the central neutron to
complete a series of operations, for instance, allocating VIP, plugging
in VIP, updating databases. Given that the main features of managing
networking resource are implemented, we hence need to adapt the mechanism
of Octavia and neutron-lbaas by improving the functionalities of the local
and central plugins.

Considering the Tricircle is dedicated to enabling networking automation
across Neutrons, the implementation can be divided as two parts,
i.e., LBaaS members in one OpenStack instance, and LBaaS members in
multiple OpenStack instances.

LBaaS members in single region
==============================

For LBaaS in one region, after installing octavia, cloud tenants should
build a management network and two security groups for amphorae manually
in the central neutron. Next, tenants need to create an interface for health
management. Then, tenants need to configure the newly created networking
resources for octavia and let octavia employ central neutron to create
resources. Finally, tenants can create load balancers, listeners, pools,
and members in the local neutron. In this case, all the members of a
loadbalancer are in one region, regardless of whether the members reside
in the same subnet or not.

LBaaS members in multiple regions
=================================

1. members in the same subnet yet locating in different regions
---------------------------------------------------------------
As shown below. ::

  +------------------------+   +-------------------+   +------------------------+
  |    net1                |   |                   |   |               net1     |
  | +---------+--------------------------+-------------------------+----------+ |
  |           |            |   |         |         |   |           |            |
  |           |            |   |         |         |   |           |            |
  |     +-----+------+     |   |         |         |   |     +-----+------+     |
  |     | Container1 |     |   |    +----+----+    |   |     | Container2 |     |
  |     +------------+     |   |    |         |    |   |     +------------+     |
  |                        |   |    |  Router |    |   |                        |
  |     +-----+------+     |   |    |         |    |   |     +-----+------+     |
  |     | Container3 |     |   |    +----+----+    |   |     | Container4 |     |
  |     +-----+------+     |   |         |         |   |     +-----+------+     |
  |           |            |   |         |         |   |           |            |
  |           |            |   |         |         |   |           |            |
  | +---------+--------------------------+-------------------------+----------+ |
  |    net2                |   |                   |   |               net2     |
  |                        |   |                   |   |                        |
  | +--------------------+ |   | +---------------+ |   | +--------------------+ |
  | |   Local  Neutron   | |   | |Central Neutron| |   | |   Local  Neutron   | |
  | +--------------------+ |   | +---------------+ |   | +--------------------+ |
  +------------------------+   +-------------------+   +------------------------+
         Region One               Central Region              Region Two

  Fig. 2 The scenario of balancing load across instances of one subnet which
  reside in different regions.

As shown in Fig. 1, suppose that a load balancer is created in Region one,
and hence a listener, a pool, and two members in subnet1. When adding an
instance in Region Two to the pool as a member, the local neutron creates
the network in Region Two. Members that locate in different regions yet
reside in the same subnet form a shared VLAN/VxLAN network. As a result,
the Tricircle supports adding members that locates in different regions to
a pool.

2. members residing in different subnets and regions
----------------------------------------------------
As shown below. ::

  +---------------------------------------+  +-----------------------+
  | +-----------------------------------+ |  |                       |
  | |            Amphora                | |  |                       |
  | |                                   | |  |                       |
  | | +---------+  +------+ +---------+ | |  |                       |
  | +-+ subnet2 +--+ mgmt +-+ subnet1 +-+ |  |                       |
  |   +---------+  +------+ +---------+   |  |                       |
  |                                       |  |                       |
  | +----------------------------------+  |  | +-------------------+ |
  | |                                  |  |  | |                   | |
  | |   +---------+      +---------+   |  |  | |    +---------+    | |
  | |   | member1 |      | member2 |   |  |  | |    | member3 |    | |
  | |   +---------+      +---------+   |  |  | |    +---------+    | |
  | |                                  |  |  | |                   | |
  | +----------------------------------+  |  | +-------------------+ |
  |           network1(subnet1)           |  |    network2(subnet2)  |
  +---------------------------------------+  +-----------------------+
                 Region One                         Region Two
  Fig. 2. The scenario of balancing load across instances of different subnets
  which reside in different regions as well.

As show in Fig. 2, supposing that a load balancer is created in region one, as
well as a listener, a pool, and two members in subnet1. When adding an instance
of subnet2 located in region two, the local neutron-lbaas queries the central
neutron whether subnet2 exist or not. If subnet2 exists, the local
neutron-lbaas employ octavia to plug a port of subnet2 to the amphora. This
triggers cross-region vxlan networking process, then the amphora can reach
the members. As a result, the LBaaS in multiple regions works.

Please note that LBaaS in multiple regions should not be applied to the local
network case. When adding a member in a local network which resides in other
regions, neutron-lbaas use 'get_subnet' will fail and returns "network not
located in current region"

Data Model Impact
-----------------

None

Dependencies
------------

None

Documentation Impact
--------------------

Configuration guide needs to be updated to introduce the configuration of
Octavia, local neutron, and central neutron.

References
----------

None
