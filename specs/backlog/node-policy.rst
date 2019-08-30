..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Adding Policy Targets for Ironic Nodes
======================================

Include the URL of your StoryBoard story (which should have an `rfe`` tag):

https://storyboard.openstack.org/#!/story/XXXXXXX

This spec describes an update to the Ironic node controller to provide
additional information for the policy checks. It is intended to accommodate
the following people:

* Owner: A node Owner who has full Ironic API access to a node.
* User: A node User who has restricted Ironic API access to a node.


Problem description
===================

Right now Ironic is not multi-tenant; anyone with API access to one node
has access to all nodes. There are a variety of use cases where this is
inconvenient:

* An Ironic deployment consists of nodes from multiple hardware Owners.
  Each Owner needs to be able to manage their own nodes through the Ironic
  API, while being unable to manage others' nodes.

* An OpenStack operator wishes to grant specific Users the capability of
  performing limited Ironic API actions on a node - for example, allowing
  a User to set a node's power state while preventing them from setting
  node properties.


Proposed change
===============

The Ironic API already has a way of controlling access to the REST API: a
policy engine[0] that is used throughout the REST API. However, when the node
controller uses the policy engine[1], it does so without passing in any
information about the node being checked. Other services such as Nova[2] and
Barbican[3] pass in additional information, and we propose doing the same. In
summary, we would like to:

* Add a ``users`` field to the ``nodes`` table that is interpreted as a list of
  project_ids that have User access to the node
* Update the node controller so that policy checks also pass in information
  about the node, including ``owner`` information and whether the person
  making the API request has an entry in the ``users`` list
* Update Ironic's default generated policy file to include ``is_node_owner``
  and ``is_node_user`` rules, along with updated rules for ``node:*`` actions
  (Nova does the same[4]). For example:

   *  "is_node_owner": "project_id:%(node.owner)s"
   *  "is_node_user": "True:%(node.is_user)s"
   *  "baremetal:node:update": "rule:is_admin or rule:is_node_owner"
   *  "baremetal:node:set_power_state": "rule:is_admin or rule:is_node_owner or rule:is_node_user"

This update is enough to control access for most API functions - except for list
functions. For those, we propose updating the node list query to filter the result
for non-admins.

Alternatives
------------

One alternative is to perform these checks at the database API level. However
that requires a new code pattern to be used on a large number of individual
functions.

Data model impact
-----------------

A ``users`` field would be added to the nodes table as a ``VARCHAR(255)``
and will have a default value of ``null`` in the database.

State Machine Impact
--------------------

None

REST API impact
---------------

An ``users`` field will be returned as part of the node. The API shall be
updated to allow a user to set and unset the value via the API.

Additionally the GET syntax for the nodes list will be updated to allow a
list of matching nodes to be returned.

POST/PATCH operations to the field will be guarded by a new microversion.
The field shall not be returned unless the sufficient microversion is supplied
for GET operations.

Client (CLI) impact
-------------------

None

"ironic" CLI
~~~~~~~~~~~~

None

"openstack baremetal" CLI
~~~~~~~~~~~~~~~~~~~~~~~~~

A corresponding change will be implemented to allow a user to set/unset a
node's ``users`` value from the command line.

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

This change exposes functionality to additional users. However this access
is constrained through Oslo policy, and can be adjusted as an adminstrator
desires.

Other end user impact
---------------------

None

Scalability impact
------------------

None

Performance Impact
------------------

Checking ``users`` membership when making an API request has a small
impact.

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

I don't wanna know...

Work Items
----------

* Add database and object field.
* Update REST API.
* Update CLI.
* Update node controller.
* Add documentation.
* Write tests

Dependencies
============

None

Testing
=======

Both unit tests and Tempest tests will be added.

Upgrades and Backwards Compatibility
====================================

The ``users`` field will be created as part of the upgrade process with
a default value in the database schema.

Documentation Impact
====================

Additional documentation describing the possible applications of
using the ``node_owner`` and ``node_user`` policy roles will be added.

The REST API documentation will be updated.

References
==========

* [0] https://github.com/openstack/ironic/blob/master/ironic/common/policy.py
* [1] https://github.com/openstack/ironic/blob/master/ironic/api/controllers/v1/node.py#L225
  Example of a current policy check. Note the use of ``cdict``; it is being passed in as both
  the ``target`` and the ``creds``.
* [2] https://github.com/openstack/nova/blob/master/nova/api/openstack/compute/servers.py#L648-L652
  Example of Nova creating a ``target`` dictionary.
* [3] https://github.com/openstack/barbican/blob/stable/rocky/barbican/api/controllers/__init__.py#L59-L72
  Example of Barbican creating a ``target`` dictionary.
* [4] https://github.com/openstack/nova/blob/master/nova/policies/base.py#L27-L30
  Example of Nova defaulting a rule that uses information from a ``target`` dictionary.
