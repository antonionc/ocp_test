# ocp_test
Setup and configuration of openshift 4 in AWS

Simple playbook that installs and configures a new cluster in aws using openshift install into a server that would be used as bastion or adminitrative server

The simplest way to use is:

`shell
$ ansible-playbook -i inventories all.yaml -e cluster_name=mycluster
`

## Configuration

Each playbook invokes one of more roles (defined in this repo) with the proper configuration

The main configuration is contained in the`group_vars/all`file:

```ansible
# Choose your version from https://mirror.openshift.com/pub/openshift-v4/clients/ocp/
es_channel: 4.3
clo_channel: 4.3
ocp_version: 4.3.1
oc_checksum: 9938b9894b50a717a94c2baa6eceeabc23678f059852b76d67f8c1d520db2fd6
oc_install_checksum: 3a01b5e64be2a7ac510dc8b55e88d7d541a2930174faec8b9224f97eb69f6817
aws_key: AWSAWSAWSAWSAWSAWSAWSAWS
aws_secret: SecretSecretSecretSecret
base_domain: sample.example.com
pull_secret: {"auths":{"cloud.openshift.com":{"auth":"foo","email":"user@example.com"},"quay.io":{"auth":"foo","email":"user@example.com"},"registry.connect.redhat.com":{"auth":"foo","email":"user@example.com"},"registry.redhat.io":{"auth":"foo","email":"user@example.com"}}}
kubeconfig: ~/{{ cluster_name }}/auth/kubeconfig
local_kubeconfig: kubeconfig_{{ cluster_name }}
oc_exec: ./oc
```

## Main usage

The main usage is with the `all.yaml` playbook, that setups and configures different parts of the OCP 4 installation.


```yaml
---
- import_playbook: install.yaml 
- import_playbook: configure.yaml 
- import_playbook: add_le.yaml
# Increase the size of the workers
- import_playbook: mset.yaml 
- import_playbook: add_rhocs.yaml
- import_playbook: infra.yaml
- import_playbook: clo.yaml 
- import_playbook: clustermon.yaml 
#- import_playbook: argocd.yaml 
```

This will install a new cluster (or modify an existing cluster with that name installed using a previous execution of the playbook), and configure it with:

- A set of users and passwords defined in `configure.yaml` with the definition of the admin role and its members
- A let's encrypt certificate for the wildcard (*.apps) configured as the ingress controller default certificate.
- Resize the workers to be bigger
- Adds new nodes with role rhoc, installs OCS, and confiugres a starting OCS cluster on the new nodes when they are ready
- Adds new nodes with role infra
- Installs and configures the cluster logging operators
- Configures the cluster monitoring


Configuration parameters for each role can be found in the `defaults/main.yaml` of each role

The configuration of the playbooks included in `all.yaml`are:

### instal.yaml (ocp_test)

This playbook installs a new cluster names `cluster_name`of the version specified: `ocp_version` 


### configure.yaml (ocp_auth role)

#### htpasswd
Configuration of users for the htpasswd Identity Provider

Its content is a dictionary with  usernames as keys and its password as value.

To avoid having a predictible password you can user the `password` lookup filter to generate a password and store its content in a file

``` yaml
   htpasswd:
    - admin: "{{ lookup('password', './admin_passwordfile length=24 ') }}"
    - developer: R3dH4t!!
    - dev2: "{{ lookup('password', './dev2_passwordfile length=24 ') }}"
```

#### admin_users

List of users to be added to the admin group (with cluster-admin rights)

``` yaml
   admin_users:
    - admin
```


### add_le.yaml (ocp_le)

This playbook request a certificate for the wildcard of *.apps.cluster_name.base_domain to Let's encrypt and configures it as the ingress controller default certificate.

Make sure to provide a valid email in the `acme_email`variable

### mset.yaml (ocp_mset role)

Configuration of machinesets. In this invocation it changes the size of workers to m4.xlarge.

Note that when you change the machinesets, OCP does not recreate the machines, so we invoek it twice:
The first one to scale to two and mark the delete_policy to Oldest

```yaml
    - replicas: 2
      delete_policy: Oldest
      role: worker
      instance_type: m4.xlarge
```

, and then we scale back to one

```yaml
    - replicas: 2
      delete_policy: Oldest
      role: worker
      instance_type: m4.xlarge
```


### add_rhocs.yaml (ocp_infra, ocp_mset, ocp_rhocs)

This playbook will generate a new machine config for role `rhocs`, then would create a machineset using that machine config, and then would install rhocs and configure to use the newly created machines

```yaml
  vars:
    rhocs_role: rhocs
    msets:
    - replicas: 1
      role: "{{ rhocs_role }}"
      instance_type: m4.4xlarge
    mset_user_data_secret: "{{rhocs_role}}-user-data"
    move_ingress: "false"
    infra_role: "{{rhocs_role}}"
```


### infra.yaml (ocp_infra, ocp_mset)

This playbook will generate a new machine config for role `infra, then it would create a new machine set using that machine config and move the ingress controller to run on on the newly created machines.
```yaml
  vars:
    msets:
    - replicas: 2
      role: infra
      instance_type: m4.2xlarge
    mset_user_data_secret: "infra-user-data"
    move_ingress: "true"
```
### clo.yaml (ocp_clo)

This playbook will install and configure the cluster logging operator on the cluster

supported varibles:
- *es_channel* Channel to subscribe the ElasticSearch operator to (if not specified it would use the defaultChannel on the elasticsearch-operator packageManifest
- *clo_channel* Channel to subscribe the Cluster Logging operator to (if not specified it would use the defaultChannel on the cluster-logging packageManifest
- *clo_node_selector* node selector to use for elasticsearchm kibana, and curator
- *clo_node_selector_annotation* Annotation format of the node selector used in the openshift-logging  openshift-operators-redhat  namespaces
- *clo_nodes* number of elastic search data nodes to use (defaults to 1)
- *clo_redundancy* redundancy policy to use for ElasticSearch indices, defaults to 'ZeroRedundancy'

```yaml
  vars:
   clo_nodes: 3
   clo_redundancy: SingleRedundancy
   clo_node_selector: "node-role.kubernetes.io/infra: ''"
```

### clustermon.yaml (ocp_clustermon)

This playbook configures the already installed cluster monitoring stack, by adding/updating the cluster-monitoring-config ConfigMap

supported variables:
- *clmon_node_selector* node selector to use for the relocatable components of the cluster monitoring
  - prometheus
  - alertmanager
  - prometheus operator
  - kubestatemetrics
  - grafana
  - telemeterclient
  - prometheus adapter

```yaml
  vars:
   clmon_node_selector: "node-role.kubernetes.io/infra: ''"
```

### argocd.yaml (ocp_argocd) (WIP)

This playbook installs argocd on the cluster

```yaml
  vars:
    argocd_version: v1.3.0
```

### add_crw (ocp_crw) (WIP)

Adds Code Ready Workspaces to the existing cluster
