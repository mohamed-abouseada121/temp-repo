create-rg
=========

Overview
--------

It creates an Azure Resource Group.

Role Variables
--------------

| Variable            | Description                                | Default Value        | Required   |
|---------------------|--------------------------------------------|----------------------|------------|
| region              | Azure Region                    |         | &#9745;   |
| existing_rg         | Existing resource group name if any  |    ""      | &#x2611;   |
| new_rg              | new resource group name         |          | &#9745;   |
| rg_cgid             | resource group cloud_gate id    |        | &#9745;   |
| cypher_auth         | cypher-shell authentication parameters  |            | &#9745;   |
| client_id           | Azure client id         |        | &#9745;   |
| secret              | Azure secret              |       | &#9745;   |
| subscription_id     | Azure subscription ID                     |           | &#9745;   |
| tenant              | Tenant ID for the Azure Active Directory  |         | &#9745;   |


Role Outputs
------------

- rg

Dependencies
------------

- `azure.azcollection` (version 2.7.0)
- `cypher-shell` (version 5.22.0)


Tasks
-----

- Create a resource group if it does not exist
- Set the rg variable to be used in other roles
- Store the resource group details in neo4j


Reference
---------

[azure_rm_resourcegroup](https://docs.ansible.com/ansible/latest/collections/azure/azcollection/azure_rm_resourcegroup_module.html#ansible-collections-azure-azcollection-azure-rm-resourcegroup-module)
