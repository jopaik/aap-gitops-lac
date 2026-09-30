# AAP 2.6 GitOps Infrastructure-as-Code Project

Declarative Ansible Automation Platform 2.6 project managing projects, inventories, credentials, and job templates via `infra.aap_configuration`. Includes a ping-checked Round-Robin dispatcher across multiple Gateway endpoints.

## Quickstart

```bash
# 1. Install Galaxy Requirements
ansible-galaxy collection install -r collections/requirements.yml

# 2. Sync Configuration to AAP Gateway Platforms
ansible-playbook playbooks/configure_aap.yml

# 3. Test Local Playbook Execution
ansible-playbook playbooks/demorun.yml

# 4. Dispatch Round-Robin Job Across Platforms
ansible-playbook playbooks/dispatch_round_robin.yml
