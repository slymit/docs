.. _minio-authenticate-using-ad-ldap-generic:

================================================================
Configure MinIO for Authentication using Active Directory / LDAP
================================================================

.. default-domain:: minio

.. contents:: Table of Contents
   :local:
   :depth: 2

Overview
--------

MinIO supports using an Active Directory / LDAP Connect for external management of user
identities. The procedure on this page provides instructions for:

- Configuring a MinIO cluster for an external AD/LDAP provider.
- Logging into the cluster using the MinIO Console and AD/LDAP credentials.
- Using the MinIO ``AssumeRoleWithLDAPIdentity`` Security Token Service (STS)
  API to generate temporary credentials for use by applications.

This procedure is generic for AD/LDAP services. Defer to the documentation for
the AD/LDAP provider of your choice for specific instructions or procedures on
configuration of user identities.

Prerequisites
-------------

Active Directory / LDAP Compatible IDentity Provider
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This procedure assumes an existing Active Directory or LDAP service.
Instructions on configuring AD/LDAP are out of scope for this procedure.

MinIO requires a read-only service account with which it
:ref:`binds <minio-external-identity-management-ad-ldap-lookup-bind>` to perform
authenticated user and group queries.

MinIO Cluster
~~~~~~~~~~~~~

This procedure assumes an existing MinIO cluster running the 
:minio-git:`latest stable MinIO version <minio/releases/latest>`. 
This procedure *may* work as expected for older versions of MinIO.

Install and Configure ``mc`` with Access to the MinIO Cluster
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This procedure uses :mc:`mc` for performing operations on the
MinIO cluster. Install ``mc`` on a machine with network access to the cluster.
See the ``mc`` :ref:`Installation Quickstart <mc-install>` for instructions on
downloading and installing ``mc``.

This procedure assumes a configured :mc:`alias <mc alias>` for the MinIO
cluster. 

.. _minio-external-identity-management-ad-ldap-configure:

Procedure
---------

1) Set the Active Directory / LDAP Configuration Settings
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can configure the AD/LDAP provider using either
environment variables *or* server runtime configuration settings. Both
methods require starting/restarting the MinIO deployment to apply changes. The
following tabs provide a quick reference of all required and optional
environment variables and configuration settings respectively:

.. tabs::

   .. tab:: Environment Variables

      MinIO supports specifying the AD/LDAP provider
      settings using :ref:`environment variables
      <minio-server-envvar-external-identity-management-ad-ldap>`. The 
      :mc:`minio server` process applies the specified settings on its next
      startup. For distributed deployments, specify these settings across all
      nodes in the deployment using the *same* values consistently.

      The following example code sets *all* environment variables related to
      configuring an AD/LDAP provider for external
      identity management. The minimum *required* variable are:
      
      - :envvar:`MINIO_IDENTITY_LDAP_SERVER_ADDR`
      - :envvar:`MINIO_IDENTITY_LDAP_LOOKUP_BIND_DN`
      - :envvar:`MINIO_IDENTITY_LDAP_LOOKUP_BIND_PASSWORD`

      .. code-block:: shell
         :class: copyable


      For complete documentation on these variables, see
      :ref:`minio-server-envvar-external-identity-management-ad-ldap`

   .. tab:: Configuration Settings

      MinIO supports specifying the AD/LDAP provider
      settings using :mc-conf:`configuration settings <identity_ldap>`. The 
      :mc:`minio server` process applies the specified settings on its next
      startup. For distributed deployments, the :mc-cmd:`mc admin config`
      command applies the configuration to all nodes in the deployment.

      The following example code sets *all* configuration settings related to
      configuring an AD/LDAP provider for external
      identity management. The minimum *required* setting are:
      
      - :mc-conf:`identity_ldap server_addr <identity_ldap.server_addr>`

      - :mc-conf:`identity_ldap lookup_bind_dn <identity_ldap.lookup_bind_dn>`

      - :mc-conf:`identity_ldap lookup_bind_password <identity_ldap.lookup_bind_password>`

      .. code-block:: shell
         :class: copyable

         mc admin config set ALIAS/ identity_ldap \

      For more complete documentation on these settings, see
      :mc-conf:`identity_ldap`.

2) Restart the MinIO Deployment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You must restart the MinIO deployment to apply the configuration changes. 
Use the :mc-cmd:`mc admin service restart` command to restart the deployment.

.. important::

   MinIO restarts *all* :mc:`minio server` processes associated to the 
   deployment at the same time. Applications may experience a brief period of 
   downtime during the restart process. 

   Consider scheduling the restart during a maintenance period to minimize
   interruption of services.

.. code-block:: shell
   :class: copyable

   mc admin service restart ALIAS

Replace ``ALIAS`` with the :mc:`alias <mc-alias>` of the deployment to 
restart.

3) Use the MinIO Console to Log In with AD/LDAP Credentials
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The MinIO Console supports the full workflow of authenticating to the
AD/LDAP provider, generating temporary credentials using
the MinIO :ref:`minio-sts-assumerolewithldapidentity` Security Token Service
(STS) endpoint, and logging the user into the MinIO deployment.

Starting in RELEASE, the MinIO Console is embedded in the MinIO server.
You can access the Console by opening the root URL for the MinIO cluster.
For example, ``https://minio.example.net:9001``.

From the Console, click :guilabel:`BUTTON` to begin the Active Directory / LDAP
authentication flow.

Once logged in, you can perform any action for which the authenticated
user is :ref:`authorized 
<minio-external-identity-management-ad-ldap-access-control>`. 

You can also create :ref:`service accounts <minio-idp-service-account>` for
supporting applications which must perform operations on MinIO. Service accounts
are long-lived credentials which inherit their privileges from the parent user.
The parent user can further restrict those privileges while creating the service
account. 

4) Generate S3-Compatible Temporary Credentials using AD/LDAP Credentials
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO requires clients authenticate using :s3-api:`AWS Signature Version 4
protocol <sig-v4-authenticating-requests.html>` with support for the deprecated
Signature Version 2 protocol. Specifically, clients must present a valid access
key and secret key to access any S3 or MinIO administrative API, such as
``PUT``, ``GET``, and ``DELETE`` operations.

Applications can generate temporary access credentials as-needed using the
:ref:`minio-sts-assumerolewithldapidentity` Security Token Service (STS) API
endpoint and AD/LDAP user credentials. MinIO provides an example Go application
:minio-git:`ldap.go <minio/blob/master/docs/sts/ldap.go>` with an example of
managing this workflow.

.. code-block:: shell

   POST https://minio.example.net/Action=AssumeRoleWithLDAPIDentity
   &LDAPUsername=USERNAME
   &LDAPPassword=PASSWORD
   &Version=2011-06-15
   &Policy={}

- Replace the ``LDAPUsername`` with the username of the AD/LDAP user.

- Replace the ``LDAPPassword`` with the password of the AD/LDAP user.

- Replace the ``Policy`` with an inline URL-encoded JSON 
  :ref:`policy <minio-policy>` that further restricts the permissions associated
  to the temporary credentials. Omit to use the  
  :ref:`policy whose name matches <minio-external-identity-management-ad-ldap-access-control>`
  the Distinguished Name (DN) of the AD/LDAP user. 

The API response consists of an XML document containing the
access key, secret key, session token, and expiration date. Applications
can use the access key and secret key to access and perform operations on
MinIO.

See the :ref:`minio-sts-assumerolewithldapidentity` for reference documentation.