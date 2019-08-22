..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Policy Engine for Ironic Node API
======================================

Include the URL of your StoryBoard story (which should have an `rfe`` tag):

https://storyboard.openstack.org/#!/story/XXXXXXX

This spec describes a custom node policy engine that provides fine-grained
access control to nodes. It is designed to accommodate the following people:

* Owner: A node Owner who has full Ironic API access to a node.
* User: A node User who has restricted Ironic API access to a node.
  These restrictions are controlled by the node Owner.


Problem description
===================

Right now Ironic is not multi-tenant; anyone with API access to one node
has access to all nodes. There are a variety of use cases where this is
inconvenient:

* An Ironic deployment consists of nodes from multiple hardware Owners.
  Each Owner needs to be able to manage their own nodes through the Ironic
  API, while being unable to manage others' nodes.

* An OpenStack operator wishes to grant specific Users the capability of
  performing specific Ironic API actions on a node (for example, setting
  the node's power state) while preventing others (setting node properties).



Proposed change
===============

The Ironic API already has a way of controlling access to the REST API:
https://github.com/openstack/ironic/blob/master/ironic/common/policy.py defines
a policy engine that is used throughout the REST API. We propose creating an
extended version of this policy engine for use solely by the Ironic node API.
This policy engine still relies upon the Oslo policy file, but also permits the
following access:

* Allows an Owner of a node to pass any policy check if they own the node, as
  determined by the node's ``owner`` field.

* Allows a User of a node to pass a policy check if they have the appropriate
  entry in a mapping table. This mapping table maps a User with a node and a
  Policy Set - a group of policies, analogous to a security group and its rules.

This alternative policy engine is enough to control access for most API
functions - except for list functions. For those, we propose updating the
node list query to filter the result for non-admins.

Note that this change does not alter the default behavior of Ironic. If a node
has no defined owner, and contains no entry in the User/Policy Set mapping
table, then the new node policy engine will act exactly the same as the current
policy engine.

Nevertheless, we can add a switch in the configuration that controls whether
the Ironic node API uses the new policy engine or the current policy engine.

Alternatives
------------

One alternative is to perform these checks at the database API level. However
that requires a new code pattern to be used on a large number of individual
functions. Using an alternate policy engine isolates those changes in a single
place, and allows the use of one or the other to be a simple as instantiating
the policy engine specified in the configuration.

Data model impact
-----------------

This change requires the creation of three new tables::

    CREATE TABLE policy_sets (
        id INT(11) NOT NULL AUTO_INCREMENT,
        name VARCHAR(255) CHARACTER SET utf8 NOT NULL,
        uuid VARCHAR(36) NOT NULL,
        project_id VARCHAR(36) NOT NULL,
        PRIMARY KEY (id),
        UNIQUE KEY `uniq_policy_sets0uuid` (`uuid`),
    )

    CREATE TABLE policy_set_policies (
        set_id INT(11),
        name VARCHAR(255) CHARACTER SET utf8 NOT NULL,
        FOREIGN KEY (set_id)
          REFERENCES policy_sets(id)
          ON DELETE CASCADE,
    )

    CREATE TABLE node_policy_sets (
        node_uuid VARCHAR(36) NOT NULL,
        set_uuid VARCHAR(36) NOT NULL,
        project_id VARCHAR(36) NOT NULL,
        FOREIGN KEY (node_uuid)
          REFERENCES nodes(uuid)
          ON DELETE CASCADE,
        FOREIGN KEY (set_uuid)
          REFERENCES policy_sets(uuid)
          ON DELETE CASCADE,
    )

Note that the ``name`` column in ``policy_set_policies`` needs to correspond
to a policy rule defined by the current policy engine.

State Machine Impact
--------------------

None

REST API impact
---------------

New REST API endpoints will be added for policy_sets and node_policy_sets.
These will support the standard CRUD operations.

These operations will be accompanied by a new microversion. If a lower
microversion is specified, then these operations will be unavailable.

Client (CLI) impact
-------------------

None

"ironic" CLI
~~~~~~~~~~~~

None

"openstack baremetal" CLI
~~~~~~~~~~~~~~~~~~~~~~~~~

A corresponding change will be implemented to allow a user to perform
these operations from the command line.

RPC API impact
--------------

None

Driver API impact
-----------------

None

Nova driver impact
------------------

???

Ramdisk impact
--------------

None

Security impact
---------------

This change exposes administrative functionality to additional users. However
this access is constrained on a node-by-node basis, and this access must be
specifically granted by an admin or Owner. Furthermore, the node policy
engine can be disabled through a configuration setting.

Other end user impact
---------------------

Admins and Owners will need to learn what policies exist in order for
them to manage Policy Sets effectively.

Scalability impact
------------------

None

Performance Impact
------------------

The node policy engine could have a small performance impact when using
Ironic's node API.

Other deployer impact
---------------------

None

Developer impact
----------------

Since policy names are referenced in ``policy_set_policies``, there could
be consequences if the names are ever changed.

Implementation
==============

Assignee(s)
-----------

I don't wanna know...

Work Items
----------

* Create new node policy engine that allows access for Owners
* Add DB tables and objects for Policy Sets
* Update node policy engine to accommodate Policy Sets
* Add API for Policy Sets
* Extend CLI to support above API
* Write tests

Dependencies
============

None

Testing
=======

Both unit tests and Tempest tests will be added.

Upgrades and Backwards Compatibility
====================================

The new APIs will be hidden behind a new API version, and the
new node policy engine will function exactly as the current policy
engine as long as the node's ``owner`` field is empty and no Policy
Sets are created. As an added precaution, usage of the new node policy
engine will require a configuration switch to be turned on.

Documentation Impact
====================

Additional documentation describing the new node policy engine
will be added.

The updated API and CLI will required updated documentation.

References
==========

None
