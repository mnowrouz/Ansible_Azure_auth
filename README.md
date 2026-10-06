# CyberArk Secrets Manager SaaS – Ansible Integration Using Azure Managed Identity

## Overview

This proof of concept demonstrates how an Ansible control node hosted on an Azure VM can retrieve secrets from CyberArk Secrets Manager SaaS (Conjur) using the **Azure Authenticator** and the VM's **system-assigned Managed Identity**.

The design does not require a Conjur API key, Azure client secret, or manually managed JWT on the Ansible VM. Azure establishes the workload identity, CyberArk validates that identity through `authn-azure`, and Conjur authorization determines which synchronized secrets the workload can retrieve.

### Authentication flow

```text
Azure VM
┌──────────────────────────────────────────┐
│ ansible-conjur-demo                      │
│                                          │
│  Ansible                                 │
│     │                                    │
│     │ cyberark.conjur lookup             │
│     ▼                                    │
│  Azure Instance Metadata Service (IMDS)  │
│     │                                    │
│     │ Managed Identity token             │
└─────┼────────────────────────────────────┘
      │
      ▼
CyberArk Secrets Manager SaaS
      │
      │ authn-azure/ansible
      ▼
Conjur workload
host/data/ansible/azure-demo
      │
      │ read permission
      ▼
Synchronized CyberArk Vault secret
```

### Authorization model

```text
                         host/data/ansible/azure-demo
                                      │
                    ┌─────────────────┴─────────────────┐
                    │                                   │
                    ▼                                   ▼
   conjur/authn-azure/ansible/apps     .../delegation/consumers
                    │                                   │
                    ▼                                   ▼
          Authenticate to Conjur               Retrieve Secrets
```

The two grants serve different purposes:

- Membership in `conjur/authn-azure/ansible/apps` answers **"May this workload authenticate?"**
- Membership in the Safe's `delegation/consumers` group answers **"Which synchronized secrets may this workload retrieve after authentication?"**

---

## Repository Structure

```text
conjur-ansible-azure/
├── README.md
├── ansible/
│   ├── test-conjur.yml
│   └── test-conjur-secure.yml
└── policies/
    ├── authn-azure-ansible.yml
    ├── ansible-workload.yml
    ├── grant-authenticator.yml
    └── grant-safe-consumer.yml
```

---

## 1. Create the Azure VM

Create a Linux VM that will act as the Ansible control node.

Example lab configuration:

```text
Resource Group:       conjur-ansible-lab
VM Name:              ansible-conjur-demo
Image:                Ubuntu Server 24.04 LTS
VM Size:              Standard_B2s
Authentication:       SSH public key
```

Allow SSH only from appropriate management networks where possible. The VM also requires outbound HTTPS connectivity to the CyberArk Secrets Manager SaaS tenant.

---

## 2. Enable the System-Assigned Managed Identity

In the Azure portal, navigate to:

```text
Virtual Machines
  → ansible-conjur-demo
    → Security
      → Identity
        → System assigned
```

Set **Status** to **On** and save the configuration.

No application registration or client secret is required for this POC.

Record the **Object (principal) ID** displayed on this page.

### Azure identifiers used by the integration

| Value | Azure Portal Location |
| --- | --- |
| Tenant ID | Microsoft Entra ID → Overview |
| Subscription ID | Subscriptions → selected subscription |
| Resource Group | Virtual Machine → Overview |
| Managed Identity Object ID | Virtual Machine → Security → Identity → System assigned |

The examples in this repository use placeholders for tenant-specific IDs. Do not publish credentials or secret values in the repository.

---

## 3. Verify the Managed Identity

SSH to the Azure VM and request a token from Azure Instance Metadata Service (IMDS):

```bash
curl -s \
  -H "Metadata: true" \
  "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https%3A%2F%2Fmanagement.azure.com%2F"
```

A successful response contains an `access_token`.

Do not copy the token into documentation or source control.

This test establishes that the VM can obtain a token using its system-assigned Managed Identity before CyberArk is introduced into the flow.

---

## 4. Install Ansible

On Ubuntu:

```bash
sudo apt update
sudo apt install -y python3-pip python3-venv git curl
```

Create a Python virtual environment:

```bash
python3 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate
```

Install Ansible:

```bash
pip install ansible
```

Verify the installation:

```bash
ansible --version
```

The validated lab used:

```text
Ansible Core: 2.21.5
Python:       3.12.3
```

---

## 5. Install the CyberArk Conjur Collection

Install the collection:

```bash
ansible-galaxy collection install cyberark.conjur
```

Verify it:

```bash
ansible-galaxy collection list | grep cyberark
```

The validated POC used:

```text
cyberark.conjur 1.3.12
```

You can inspect the lookup plugin installed on the VM with:

```bash
ansible-doc -t lookup cyberark.conjur.conjur_variable
```

---

## 6. Create the Azure Authenticator in Secrets Manager SaaS

In the Secrets Manager SaaS UI, open the **Authenticators** page and create an **Azure Authenticator**.

Use the service ID:

```text
ansible
```

This creates the authenticator:

```text
authn-azure/ansible
```

### Find the Azure Tenant ID

In the Azure portal, navigate to:

```text
Microsoft Entra ID
  → Overview
    → Tenant ID
```

Construct the Provider URI using the Tenant ID:

```text
https://sts.windows.net/<TENANT-ID>/
```

Enter this value as the authenticator's **Provider URI**.

After creating the authenticator, check its status in Secrets Manager SaaS and confirm that validation succeeds.

### Equivalent authenticator policy

`policies/authn-azure-ansible.yml`

```yaml
---
- !policy
  id: ansible
  body:
    - !webservice

    - !variable
      id: provider-uri

    - !group
      id: apps
      annotations:
        description: >-
          Group of hosts permitted to authenticate using
          the authn-azure/ansible authenticator

    - !permit
      role: !group apps
      privilege:
        - read
        - authenticate
      resource: !webservice

    - !webservice
      id: status
      annotations:
        description: >-
          Status service used to verify that the Azure
          authenticator is configured correctly

    - !group
      id: operators
      annotations:
        description: >-
          Group of users permitted to check the status
          of the authenticator

    - !permit
      role: !group operators
      privilege:
        - read
      resource: !webservice status
```

Load this policy under:

```text
conjur/authn-azure
```

The resulting structure is:

```text
conjur/authn-azure/ansible
├── webservice
├── provider-uri
├── apps
├── status
└── operators
```

The `provider-uri` variable value is then set to:

```text
https://sts.windows.net/<TENANT-ID>/
```

---

## 7. Synchronize a CyberArk Vault Secret

For this POC, a CyberArk Vault account was synchronized into Secrets Manager SaaS before the workload was created.

The synchronization generated variables similar to:

```text
data/vault/conjur-ansible-demo/Database-MySQL-TEST-administrator/username

data/vault/conjur-ansible-demo/Database-MySQL-TEST-administrator/password
```

The actual username or password values should never be stored in this repository.

The generated Safe authorization structure includes a consumer group that can be used to grant workloads access to the synchronized secrets.

---

## 8. Create the Ansible Workload

Create a branch for the Ansible workload:

```text
data/ansible
```

Create the workload:

```text
data/ansible/azure-demo
```

Configure the Azure authentication annotations using values from the Azure VM:

```text
authn-azure/ansible/resource-group
authn-azure/ansible/subscription-id
authn-azure/ansible/system-assigned-identity
```

For this lab, the resource group is:

```text
conjur-ansible-lab
```

Use your own Azure Subscription ID and Managed Identity Object ID for the remaining values.

### Equivalent workload policy

`policies/ansible-workload.yml`

```yaml
---
- !host
  id: azure-demo
  annotations:
    authn-azure/ansible/resource-group: conjur-ansible-lab
    authn-azure/ansible/subscription-id: <AZURE-SUBSCRIPTION-ID>
    authn-azure/ansible/system-assigned-identity: <AZURE-MANAGED-IDENTITY-OBJECT-ID>
```

Load this policy under:

```text
data/ansible
```

The resulting host identity is:

```text
host/data/ansible/azure-demo
```

---

## 9. Grant the Workload Access to the Azure Authenticator

The workload must be a member of the authenticator's `apps` group.

### `policies/grant-authenticator.yml`

```yaml
---
# Grants the Azure-hosted Ansible workload permission to use
# the authn-azure/ansible authenticator.
#
# Load this policy at the root policy branch.

- !grant
  role: !group conjur/authn-azure/ansible/apps
  members:
    - !host data/ansible/azure-demo
```

The relationship is:

```text
host/data/ansible/azure-demo
          │
          ▼
group/conjur/authn-azure/ansible/apps
          │
          ▼
webservice/conjur/authn-azure/ansible
      read + authenticate
```

---

## 10. Grant the Workload Access to the Synchronized Safe

Grant the workload membership in the synchronized Safe's consumer group.

### `policies/grant-safe-consumer.yml`

```yaml
---
# Grants the Azure-hosted Ansible workload access to secrets
# synchronized from the conjur-ansible-demo CyberArk Safe.
#
# Load this policy at the root policy branch.

- !grant
  role: !group data/vault/conjur-ansible-demo/delegation/consumers
  members:
    - !host data/ansible/azure-demo
```

The resulting authorization flow is:

```text
host/data/ansible/azure-demo
          │
          ▼
data/vault/conjur-ansible-demo/delegation/consumers
          │
          ▼
Synchronized Safe variables
          │
          ├── username
          └── password
```

At this point, the host has both of the required relationships:

```text
Azure Managed Identity
        │
        ▼
host/data/ansible/azure-demo
        │
        ├── conjur/authn-azure/ansible/apps
        │        └── permits authentication
        │
        └── data/vault/.../delegation/consumers
                 └── permits secret retrieval
```

---

## 11. Configure the Ansible Environment

Activate the virtual environment:

```bash
source ~/ansible-venv/bin/activate
```

Set the Secrets Manager SaaS connection and Azure Authenticator configuration:

```bash
export CONJUR_ACCOUNT="conjur"
export CONJUR_APPLIANCE_URL="https://<TENANT>.secretsmgr.cyberark.cloud/api"
export CONJUR_AUTHN_TYPE="azure"
export CONJUR_AUTHN_SERVICE_ID="ansible"
export CONJUR_AUTHN_LOGIN="host/data/ansible/azure-demo"
```

No Conjur API key, Azure client secret, or manually supplied JWT is configured.

For a system-assigned Managed Identity, `AZURE_CLIENT_ID` is not required for this POC.

### Verify the environment variables

Before running the playbook, verify the configuration:

```bash
env | grep -E '^(CONJUR|AZURE_)' | sort
```

Expected output is similar to:

```text
CONJUR_ACCOUNT=conjur
CONJUR_APPLIANCE_URL=https://<TENANT>.secretsmgr.cyberark.cloud/api
CONJUR_AUTHN_LOGIN=host/data/ansible/azure-demo
CONJUR_AUTHN_SERVICE_ID=ansible
CONJUR_AUTHN_TYPE=azure
```

There should be no requirement for values such as:

```text
CONJUR_AUTHN_API_KEY
CONJUR_AUTHN_JWT_TOKEN
AZURE_CLIENT_ID
```

in this system-assigned Managed Identity configuration.

---

## 12. Test Secret Retrieval – Visible Username

The first playbook provides an easy visual demonstration that Ansible successfully retrieved a synchronized value from CyberArk.

### `ansible/test-conjur.yml`

```yaml
---
- name: Test CyberArk Secrets Manager Azure authentication
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Retrieve username from Secrets Manager
      ansible.builtin.debug:
        msg: >-
          {{ lookup(
            'cyberark.conjur.conjur_variable',
            'data/vault/conjur-ansible-demo/Database-MySQL-TEST-administrator/username'
          ) }}
```

Run it:

```bash
ansible-playbook ansible/test-conjur.yml
```

A successful test resembles:

```text
TASK [Retrieve username from Secrets Manager]
ok: [localhost] => {
    "msg": "administrator"
}
```

This demonstrates that Ansible authenticated through the Azure Managed Identity and retrieved the synchronized CyberArk value.

Warnings about no inventory being parsed are expected in this simple localhost POC and do not indicate an authentication failure.

---

## 13. Test Secret Retrieval Without Printing the Password

For customer demonstrations, use a second playbook to prove that the password was retrieved without exposing it in the Ansible output.

### `ansible/test-conjur-secure.yml`

```yaml
---
- name: Test CyberArk Secrets Manager secret retrieval
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Retrieve password from Secrets Manager
      ansible.builtin.set_fact:
        conjur_password: >-
          {{ lookup(
            'cyberark.conjur.conjur_variable',
            'data/vault/conjur-ansible-demo/Database-MySQL-TEST-administrator/password'
          ) }}
      no_log: true

    - name: Confirm credential retrieval
      ansible.builtin.debug:
        msg: "SUCCESS: Credential was retrieved from CyberArk Secrets Manager."
      when:
        - conjur_password is defined
        - conjur_password | length > 0
```

Run it:

```bash
ansible-playbook ansible/test-conjur-secure.yml
```

The secret retrieval task is censored by `no_log: true`, while the following task provides immediate confirmation that a non-empty credential was returned.

Expected confirmation:

```text
TASK [Confirm credential retrieval]
ok: [localhost] => {
    "msg": "SUCCESS: Credential was retrieved from CyberArk Secrets Manager."
}
```

This provides immediate proof during a customer demonstration without displaying the password or waiting for audit events to appear.

---

## 14. End-to-End Flow

Once configured, the complete flow is:

```text
Ansible playbook
      │
      ▼
cyberark.conjur lookup plugin
      │
      │ CONJUR_AUTHN_TYPE=azure
      ▼
Azure IMDS
169.254.169.254
      │
      │ System-assigned Managed Identity token
      ▼
CyberArk Secrets Manager SaaS
      │
      ▼
authn-azure/ansible
      │
      │ validates Azure identity attributes
      ▼
host/data/ansible/azure-demo
      │
      │ member of Safe consumers group
      ▼
data/vault/conjur-ansible-demo/
Database-MySQL-TEST-administrator/
      │
      ├── username
      └── password
      │
      ▼
Ansible task
```

The VM stores no Conjur API key and no Azure client secret. The Azure platform provides the initial workload identity.

---

## Troubleshooting

### Certificate content or certificate file is missing

Do **not** make certificate loading a prerequisite unless the lookup plugin reports a certificate error.

If the initial playbook fails with an error similar to:

```text
Both certificate content and certificate file are invalid or missing.
Please provide a valid certificate.
```

configure an appropriate certificate file for the Secrets Manager SaaS endpoint and expose it to the collection:

```bash
export CONJUR_CERT_FILE="$HOME/conjur.pem"
```

For lab troubleshooting, the endpoint certificate can be inspected with OpenSSL. In production, use the appropriate trusted CA chain according to organizational certificate-management requirements.

After configuring the certificate, verify the environment again:

```bash
env | grep -E '^(CONJUR|AZURE_)' | sort
```

The output should now also contain:

```text
CONJUR_CERT_FILE=/home/azureuser/conjur.pem
```

Retry the playbook:

```bash
ansible-playbook ansible/test-conjur.yml
```

### Azure IMDS does not return a token

Verify that the VM's system-assigned Managed Identity is enabled:

```text
Virtual Machine
  → Security
    → Identity
      → System assigned
```

Then retry the IMDS request from the VM.

### Authenticator status fails

Verify that the Provider URI uses the Azure Tenant ID and includes the trailing slash:

```text
https://sts.windows.net/<TENANT-ID>/
```

The Tenant ID is available at:

```text
Microsoft Entra ID → Overview → Tenant ID
```

### Authentication succeeds but secret retrieval fails

Authentication and secret authorization are separate.

Verify that the workload is a member of both:

```text
conjur/authn-azure/ansible/apps
```

and:

```text
data/vault/conjur-ansible-demo/delegation/consumers
```

The first permits authentication. The second permits access to the synchronized Safe secrets.

---

## Security Characteristics Demonstrated by the POC

```text
Long-lived Conjur API key on VM       ✗
Azure application client secret       ✗
Manually supplied JWT                 ✗
System-assigned Managed Identity      ✓
Azure workload identity validation    ✓
Conjur authentication authorization   ✓
Safe-level secret authorization       ✓
Secret retrieval through Ansible      ✓
```

The key benefit of this design is that the Ansible control node does not need to hold a separate bootstrap credential simply to authenticate to Secrets Manager SaaS. Azure establishes the workload identity, while CyberArk controls authentication and authorization to the synchronized secrets.
