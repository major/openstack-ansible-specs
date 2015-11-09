Create Certificate Authority
############################
:date: 2015-11-09 00:00
:tags: certificates, ssl, tls, security, encryption

This spec aims to create a certificate authority (CA) for openstack-ansible
deployments. That certificate authority would be used to issue certificates
for various servers and clients throughout the OpenStack environment. The CA
would be provisioned early in the bootstrap process so that certificates for
individual services can be issued from the CA.

Blueprint - Create Certificate Authority:

* *TBD*


Problem description
===================

Deployers currently have two options to deploy SSL certificates for various
services

1. Generate ad-hoc self-signed certificates within various openstack-ansible
   roles and automatically deploy those certificates within the OpenStack
   environment.
2. Acquire certificates from another trusted authority and provide the paths
   to those certificates when running the openstack-ansible playbooks.

The ad-hoc self-signed certificates are the default, but they present several
problems:

1. The certificates are useful for encryption, but they cannot be verified
   against a trusted CA.
2. Self-signed certificate generation is done differently in various roles at
   different times. There is quite a bit of duplicated code within the
   openstack-ansible repository for creating self-signed certificates.
3. Some services (rsyslog, for example) require certificates that are issued
   from a trusted CA.


Proposed change
===============

An Ansible role will be created that will create a CA and issue certs. Any
roles which use SSL/TLS functionality can utilize the CA role to issue
certificates for any services within that role.

Deployers would have the option to disable this process for any service if they
want to provide their own certificates from an external CA.

Configuration options should be added for:

* Subject matter: Deployers must be able to set common SSL certificate
  content, such as the common name, organization name, or locality data.
* Duration: Deployers must be able to set the duration for certificates issued
  from the CA as well as the CA itself. *Sensible defaults should be used in
  the absence of deployer configuration.*
* Forced regeneration: An option must exist that allows a deployer to force a
  new certificate and key to be generated, and then signed by the CA.
* Opt-out: Deployers must have the ability to opt-out of certificates generated
  by the CA for any service deployed by openstack-ansible. This would require
  deployers to generate their own certificates from another external CA.

There also must be functionality that will allow deployers to gracefully
recover from the loss of the CA whether due to security compromises or complete
data loss.  Deployers must be able to recover from this failure gracefully.

Once the CA role is functional, the self-signed certificate generation tasks
and variables could be removed from various openstack-ansible roles.

Alternatives
------------

The current self-signed certficate generation mechanisms that exist in the
roles could be used.

Another alternative could be the use of OpenStack's `Anchor`_ project. However,
that project issues ephemeral certificates (duration of 24 hours or less), and
this can be cumbersome to manage in a large OpenStack environment. Certificates
with a longer duration are preferred for stability.

.. _Anchor: https://wiki.openstack.org/wiki/Security/Projects/Anchor

Playbook/Role impact
--------------------

Playbooks and roles would need to call upon the new CA role for generating
certificates rather than using the existing self-signed certificate tasks that
exist in various roles today. The CA role would offer a standard method for
issuing certificates with a predictable process and predictable variable names
for configuration.


Upgrade impact
--------------

No containers will be removed or added with this change.  However, the issuance
of SSL certificates will change from being done ad-hoc in each individual role
will be replaced with centralized certificate issuance within the CA role
itself.

If a deployer already has certificates deployed from a previous version, they
could opt-out of deploying new certificates from the new CA role, or they could
allow all of their certificates to be re-issued from the new CA.  This will be
configurable in the CA role.


Security impact
---------------

The new CA role will increase security on the system since all certificates
can be verified against the CA.  It ensures that SSL certificates are all
generated the same way, by the same Ansible tasks, from the same CA.

In addition, more services can be secured with SSL/TLS encryption, such as
rsyslog log transport. The rsyslog daemon is one of the services which
**requires** a trusted CA before encrypted connections can be made.

However, creating a centralized CA does create an additional item to protect
within the OpenStack environment.  The CA, and its related files, will be
stored so that only the root user can access it. Hardware security modules
(HSM's) are out of scope at the moment, but they could be added later for
additional security.

Deployers who operate OpenStack infrastructure in high security environments
are **strongly urged** to maintain an carefully maintained, highly secure CA
outside of their OpenStack deployment.


Performance impact
------------------

The deployment impact should be a slight extension of overall deployment time,
since the CA will need to be generated. The time difference will be only a few
seconds.

Once the environment is fully deployed, there should be no performance
difference in OpenStack services that have a self-signed certificate or a
certificate issued from the centralized CA.


End user impact
---------------

End users will notice a new certificate from a new CA and they may receive
certificate warnings from their browser or scripts *if the deployer chooses*
to replace the existing self-signed certificates with ones issued from the new
CA.

Deployers could present end users with the trusted CA certificate for the
OpenStack environment and users could add that certificate to their web
browser or utilize it with their scripts that query OpenStack API endpoints.
This would reduce the number of times that end users see browser warnings or
errors from scripts.


Deployer impact
---------------

Deployers would need to choose from two options:

1. Issue certificates from the new CA. *(default)*
2. Provide their own certificates and keys.

Deployers who choose to use the default will still have certificates applied to
any services that require them.  However, those services will have certificates
issued from the new CA rather than ad-hoc self-signed certificates from various
roles.  Deployers will still have the option to set the subject matter within
the certificate as they did before, and they will be able to set the
certificate duration as well.

Deployers with external CA's will still issue certificates from that CA, copy
them to the deployment server, and deploy them as user-provided certificates
as they do today.

Developer impact
----------------

Developers will need to utilize the new CA role when creating SSL certificates
for various services.  There will be a standard mechanism for importing the
role within other roles (similar to how we currently install pip in each
container) and it will be easy for developers to copy and paste into their
roles.


Dependencies
------------

There are no known dependencies at this time.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  `Major Hayden`_ (*mhayden* on Freenode IRC)

.. _Major Hayden: https://launchpad.net/~rackerhacker


Work items
----------

The work items would include:

1. Create the CA role.
2. Write documentation for configuring the CA role and how to implement it
   within other roles.
3. Convert existing roles to use the new CA role instead of issuing ad-hoc
   self-signed certificate in each individual role.


Testing
=======

Once the CA role is ready, it can be tested in the standard gate check jobs.
One added benefit is that we can test the CA role along with the user-provided
certificate logic in each individual role at the same time.

No additional hardware or resources are required.


Documentation impact
====================

Some documentation will be required:

1. How to configure the CA role and the certificates generated from it.
2. How to utilize the CA role within other roles which require SSL certficates
   for the services they are deploying.
3. Removal and/or adjustment of documentation for each individual role as the
   ad-hoc self-signed certificate functionality is removed.


References
==========

[openstack-dev] [openstack-ansible][security] Creating a CA for
openstack-ansible deployments?

* http://lists.openstack.org/pipermail/openstack-dev/2015-October/077877.html

