create-vpc
=========

Overview
--------

Creates an Azure Virtual Network.

Role Variables
--------------

| Variable            | Description                                | Default Value        | Required   |
|---------------------|--------------------------------------------|----------------------|------------|
| rg_id               | Resource group name                        |                      | &#9745;   |
| existing_network_id | Existing network ID                        |                      | &#9745;   |
| new_vpc_name        | New VPC name                               |                      | &#9745;   |
| new_vpc_cidr        | New VPC CIDR block                         |                      | &#9745;   |
| client_id           | Azure client id                            |                      | &#9745;   |
| secret              | Azure secret                               |                      | &#9745;   |
| subscription_id     | Azure subscription ID                      |                      | &#9745;   |
| tenant              | Tenant ID for the Azure Active Directory   |                      | &#9745;   |
| cypher_auth         | cypher-shell authentication parameters     |                      | &#9745;   |
| new_vpc_cgid        | New VPC cloud_gate id                      |                      | &#9745;   |

Dependencies
------------

- `azure.azcollection`
- [requirements-azure.txt](https://github.com/opex-sa/cloud_gate_engine/blob/main/ansibleImageDocker/requirements-azure.txt)
- cypher-shell

Tasks
-----

- Create new VPC if it does not exist
- Store VPC details in Neo4j

Example Playbook
----------------


    - hosts: localhost
      connection: local
      gather_facts: False
      roles:
        - role: create-vpc
          vars:
            rg_id: "{{ rg }}"
            existing_network_id: "{{ existing_network_id }}"
            new_vpc_name: "{{ new_vpc_name }}"
            new_vpc_cidr: "{{ new_vpc_cidr }}"
            new_vpc_cgid: "{{ new_vpc_cgid }}"
            cypher_auth: "-u {{ neo4j_user }} -p {{ neo4j_password }} -a {{ neo4j_host }}"
            client_id: "{{ client_id }}"
            secret: "{{ secret }}"
            subscription_id: "{{ subscription_id }}"
            tenant: "{{ tenant }}"


Reference
---------

- [azure_rm_virtualnetwork](https://docs.ansible.com/ansible/latest/collections/azure/azcollection/azure_rm_virtualnetwork_module.html)