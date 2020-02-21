OCP Test
=========

Role to configure a ocp 4 cluster on AWS 

Role Variables
--------------

To specify a version to download the following variables need to be specified
The role would download the oc client and the openshift-installer
ocp_version: 4.3.1
oc_checksum: 9938b9894b50a717a94c2baa6eceeabc23678f059852b76d67f8c1d520db2fd6
oc_install_checksum: 3a01b5e64be2a7ac510dc8b55e88d7d541a2930174faec8b9224f97eb69f6817

The configuration of the installer would use the following variables:

For AWS conection:
aws_key: AWSAWSAWSAWSAWS
aws_secret: SecretSecretSecret

For  cluster configuration:
base_domain: mycluster.example.com
pull_secret: {"auths":{"cloud.openshift.com":{"auth":"foo","email":"user@example.com"},"quay.io":{"auth":"foo","email":"user@example.com"},"registry.connect.redhat.com":{"auth":"foo","email":"user@example.com"},"registry.redhat.io":{"auth":"foo","email":"user@example.com"}}}

variables for executing openshift commands locally and remotely:

kubeconfig: ~/{{ cluster_name }}/auth/kubeconfig
local_kubeconfig: kubeconfig_{{ cluster_name }}
oc_exec: ./oc

Dependencies
------------

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - { role: username.rolename, x: 42 }

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
