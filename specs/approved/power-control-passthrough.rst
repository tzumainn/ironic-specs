..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Power Control Passthrough
=========================

In certain situations, a non-admin user with access to an Ironic node
may need access to power credentials. For example, a lessee may lease
an Ironic node with the intent of using non-Ironic provisioning tools.

Ironic possesses this power credential information. However, if the
non-admin's use of the node is only temporary, releasing that
information is unwise.

This spec describes a solution that allows Ironic to act as a power
control intermediary while using a Redfish emulator as a proxy. This
ensures that the solution is compatible with existing provisioning
tools while allowing Ironic to maintain access control.

Problem description
===================

Ironic can be configured to allow lessees to gain temporary API access
to leased nodes. That Ironic API access allows the lessee to perform
a variety of actions, including provisioning. However there is another
class of action for which the lessee would need power control
credentials. This is information that Ironic has; however, releasing
that information would be unwise, as that would allow the lessee to
control the node's power state even after the lease has expired.

Proposed change
===============

We propose using a single proxy to act as a power control intermediary.
sushy-tools [0]_ allows for the creation of a configurable Redfish
emulator that can be used with a variety of drivers, including libvirt
and Nova drivers. We propose creating a new Ironic driver that implements
the following methods by calling the matching Ironic API:

* `get_power_state`
* `set_power_state`
* `get_boot_device`
* `set_boot_device`

These methods will use the Ironic node's UUID as the identifier for the
Ironic API call. The Redfish emulator must be configured to access the
Ironic API using admin credentials.

For authentication, we propose including an Ironic configuration option
such that, whenever the node's owner or lessee is updated, random
`emulated_bmc_username` and `emulated_bmc_password` values are
created and stored in the Ironic node's `properties` field. These values
can be viewed by the non-administrative user, and used as the Redfish
username or password. When an API request is sent to the Redfish emulator
using the Ironic driver, the emulator will make an Ironic API call to
retrieve the Ironic node's `emulated_bmc` values. If they match the
passed in username and password, then the Ironic driver will make the
requested API call using the equivalent Ironic API.

The `emulated_bmc` values can also be directly set by an administrator
or anyone else with the appropriate API access, such as a node owner.

Note that the Ironic driver will not allow users to retrieve all systems;
thus, there is no need to scope any such call.

Alternatives
------------

VirtualBMC [1]_ is an alternative to sushy-tools, using IPMI instead of
Redfish. However, Redfish is the preferred power control driver.

Data model impact
-----------------

None

State Machine Impact
--------------------

None

REST API impact
---------------

None

Client (CLI) impact
-------------------

None

"ironic" CLI
~~~~~~~~~~~~

None

"openstack baremetal" CLI
~~~~~~~~~~~~~~~~~~~~~~~~~

None

RPC API impact
--------------

None

Driver API impact
-----------------

None

Nova driver impact
------------------

None

Ramdisk impact
--------------

None

Security impact
---------------

The Redfish emulator requires the use of Ironic admin credentials.

This solution also grants additional access to the power control
API. However, that access is fundamentally gated by the same
mechanisms as the Ironic API.

Note that this functionality can be shut off by simply not setting
any `emulated_bmc` parameters.

Other end user impact
---------------------

None

Scalability impact
------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignees:
* tzumainn - tzumainn@redhat.com
* larsks - lars@redhat.com

Work Items
----------

* Create Ironic driver for sushy-tools
* Add optional ability to create randomized `emulated_bmc`
  username and password when a node changes owner or lessee.
* Write tests.
* Write documentation detailing usage.

Dependencies
============

None

Testing
=======

We will add unit tests and Tempest tests.

Upgrades and Backwards Compatibility
====================================

N/A

Documentation Impact
====================

Usage documentation will be created.

References
==========

.. [0] https://opendev.org/openstack/sushy-tools
.. [1] https://opendev.org/openstack/virtualbmc

