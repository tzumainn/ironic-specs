..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================
Add Lessee to Ironic Nodes
==========================

https://storyboard.openstack.org/#!/story/2006506

This spec describes an update that allows nodes to specify a lessee:
someone who can provision the node and has limited node API access.

Problem description
===================

A person using Nova or Ironic's allocation API has access to every node
in Ironic's node inventory. There is no way to isolate specific nodes
for specific users.

Allowing nodes to specify a lessee would allow administrators of a data
center to assign use of specific nodes to specific users.

Proposed change
===============

We propose adding a ``lessee`` field to the node object. This field is
stored in the database and can be queried and manipulated through the REST
API.

Next, we propose updating Ironic's allocation API so that a user can only
be allocated a node if their project matches the project specifies in the
``lessee`` field. Usage of this functionality will be controlled by a
configuration switch: if off, then the behavior of the allocation API will
not change.

Lastly, we would like to update the REST API node controller so that policy
checks pass in ``lessee`` information. Then we would update the default
generated policy file as follows:

*  "is_node_lessee": "project_id:%(node.lessee)s"
*  "baremetal:node:get"*: "rule:is_admin or rule:is_node_lessee"
*  "baremetal:node:set_power_state": "rule:is_admin or rule:is_node_lessee" 

Alternatives
------------

Lessee information could be stored in ``extra``. However the changes to
the allocation API and policy checks means a dedicated field makes more
sense.

Data model impact
-----------------

A ``lessee`` field would be added to the nodes table as a ``VARCHAR(255)``
with a default value of ``null``.

State Machine Impact
--------------------

None

REST API impact
---------------

The node REST API will be updated to return the ``lessee`` field as part
of the node; and to allow a user to set and unset the ``lessee`` field.

Additionally the GET syntax for listing nodes will be updated to allow
matching against the ``lessee`` field.

A new microversion will guard POST/PATCH operations to the ``lessee`` field,
and ensure that the ``lessee`` is not returned if an insufficient microversion
is supplied.

Client (CLI) impact
-------------------

None

"ironic" CLI
~~~~~~~~~~~~

None

"openstack baremetal" CLI
~~~~~~~~~~~~~~~~~~~~~~~~~

The CLI will be updated to allow the ``lessee`` field to be set and unset
from the command line.


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

This change exposes functionality to additional users. However this access
is constrained through Oslo policy, and can be adjusted as an administrator
desires.

Other end user impact
---------------------

None

Scalability impact
------------------

None

Performance Impact
------------------

None: although node data needs to be retrieved in order to pass that
information into a policy check, the controller functions already fetch
that information.

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

* Add database field.
* Add object field.
* Update node controller.
* Update allocation API.
* Add REST API functionality and microversion.
* Update python-ironicclient.
* Add documentation.
* Write tests.

Dependencies
============

None

Testing
=======

Both unit tests and Tempest tests will be added.

Upgrades and Backwards Compatibility
====================================

The ``lessee`` field will be created as part of the upgrade process
with the default value specified in the database schema.

Documentation Impact
====================

We will include additional documentation describing the new configuration
switch as well as possible applications of using the ``node_lessee`` policy role.

References
==========

None
