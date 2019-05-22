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
 |                                                     |   +------------------+    +------------------------+                             |             |
 |                                                     +-->| kuryr/libnetwork | -->|Docker network and port |<------------Binding---------+             |
 | +-------------+    +---------+    +-------------+   |   +------------------+    +------------------------+                              \            |
 | | Zun Request | -->| Zun API | -->| Zun Compute | --+                                                                                    +           |
 | +-------------+    +---------+    +-------------+   |   +--------------+    +--------------+                                             |           |
 |                                                     +-->| GlanceClient | -->| docker image |                                       +-----------+     |
 |                                                     |   +--------------+    +------+-------+                                       | Container |     |
 |                                                     |                              |                                               +-----------+     |
 |                                                     |   +------------+    +--------V--------+                                            |           |
 |                                                     +-->| Docker API | -->| docker instance | -------------------------------------------+           |
 |                                                         +------------+    +-----------------+                                                        |
 +------------------------------------------------------------------------------------------------------------------------------------------------------+
                                              Fig. 2 The multi-region container creation workflow.

Specifically, when a tenant attempts to create container, he/she needs to
send a request to Zun API. ………….


Container network connectivity in multi-region scenario
=======================================================

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

        Fig. 3 The network connectivity of containers in multi-region scenario.

As shown in Fig. 3, suppose that a load balancer is created in Region one,
and hence a listener, a pool, and two members in subnet1. When adding an
instance in Region Two to the pool as a member, the local neutron creates
the network in Region Two. Members that locate in different regions yet
reside in the same subnet form a shared VLAN/VxLAN network. As a result,
the Tricircle supports adding members that locates in different regions to
a pool.


Data Model Impact
-----------------

None

Dependencies
------------

None

Documentation Impact
--------------------

None

References
----------

None
