
# Federated Multi-Cluster Red Hat Ansible Automation Platform with GitOps

A quick demo of GitOps pipeline for Federated Multi-cluster Multi-region Red Hat Ansible Automation Platform (AAP) setups. Uses GitHub Actions and `ansible.controller` modules to manage declarative platform configuration on `main` and handle workload dispatch on `deploy`.

---

## 🏗️ Pipeline Architecture & Workflow

```text
+---------------------------------------------------------------------------------+
|                                GITHUB REPOSITORY                                |
|                                                                                 |
| vars/aap_config.yml      playbooks/configure_aap.yml    playbooks/demorun.yml  |
| vars/vault.yml           playbooks/dispatch_round_robin.yml                     |
| vars/deploy_job.yml                                                             |
+---------------------------------------------------------------------------------+
           /                                                       \
   Git Push to 'main'                                      Git Push to 'deploy'
         /                                                           \
        v                                                             v
+-----------------------------------+               +-----------------------------------+
| WORKFLOW: gitops-main.yml         |               | WORKFLOW: gitops-deploy.yml       |
| Container: aap-runner:jp1         |               | Container: aap-runner:jp1         |
|                                   |               |                                   |
| 1. Generate .vault_pass           |               | 1. Generate .vault_pass           |
| 2. Execute configure_aap.yml      |               | 2. Execute dispatch_round_robin   |
| 3. Clean up .vault_pass           |               | 3. Clean up .vault_pass           |
+-----------------------------------+               +-----------------------------------+
                 |                                                    |
           Phase 1: IaC                                         Phase 2: Job Launch
    Syncs Orgs, Credentials,                             Dispatches Job Template via
   Inventories, Groups, Hosts,                           Label / Instance Name Match or
        Projects, Templates                                GITHUB_RUN_NUMBER % 2
                 v                                                    v
+----------------------------------+                 +----------------------------------+
| AAP PLATFORM DC1                 |                 | AAP PLATFORM DC2                 |
|                                  |                 |                                  |
| • Labels: [dc1, production]      |                 | • Labels: [dc2, non-production]  |
| • Org: Default                   |                 | • Org: Default                   |
| • Credential: Production SSH     |                 | • Credential: Production SSH     |
| • Project: GitOps Application    |                 | • Project: GitOps Application    |
| • Inventory: Multi-DC Production |                 | • Inventory: Multi-DC Production |
| • Groups: dc1, dc2, prod, non-prod|                | • Groups: dc1, dc2, prod, non-prod|
| • Template: Deploy Workload Job  |                 | • Template: Deploy Workload Job  |
+----------------------------------+                 +----------------------------------+
                 \                                                   /
                  \---> [ Executed Job Run on Selected Cluster ] <--/

```

---

## 📁 Repository Structure

```text
.
├── .github/
│   └── workflows/
│       ├── gitops-main.yml         # IaC pipeline triggered on pushes to 'main'
│       └── gitops-deploy.yml       # Workload dispatch pipeline triggered on pushes to 'deploy'
├── playbooks/
│   ├── configure_aap.yml           # Entrypoint playbook iterating across all Gateway clusters
│   ├── configure_single_gateway.yml# Native ansible.controller module tasks executed per gateway
│   ├── demorun.yml                 # Target workload playbook executed on AAP execution nodes
│   └── dispatch_round_robin.yml    # Workload dispatch playbook with dynamic target selection
├── vars/
│   ├── aap_config.yml              # Declarative IaC variables (Orgs, Credentials, Groups, Hosts, Templates)
│   ├── deploy_job.yml              # Runtime dispatch parameters (target_aap, deploy_job_name, deploy_job_limit)
│   └── vault.yml                   # Encrypted secret credentials (Vault URLs, passwords, SSH keys)
├── .vault_pass                     # Transient vault password file (generated and purged during pipeline)
└── README.md                       # High-level architecture and pipeline documentation

```

---

## ⚙️ Workload Dispatch Configuration (`vars/deploy_job.yml`)

The workload execution parameters are controlled declaratively via `vars/deploy_job.yml`:

```yaml
---
# Default Target Selection ('auto', 'dc1', 'dc2', 'production', 'non-production', or Instance Name)
target_aap: "auto"

# Default Job Template to Launch
deploy_job_name: "Deploy Workload Job"

# Default Execution Limit Filter
deploy_job_limit: "dc1"

```

### Runtime Override Priority

All dispatch variables support runtime command-line and workflow overrides via `extra-vars`:

```bash
# Example 1: Dispatch to DC1 targeting specific host
ansible-playbook playbooks/dispatch_round_robin.yml \
  --vault-password-file .vault_pass \
  -e "target_aap_override=dc1" \
  -e "deploy_job_limit_override=dc1_host_1"

# Example 2: Dispatch a secondary job template to non-production environment
ansible-playbook playbooks/dispatch_round_robin.yml \
  --vault-password-file .vault_pass \
  -e "target_aap_override=non-production" \
  -e "deploy_job_name_override=Deploy Workload Job 2" \
  -e "deploy_job_limit_override=non-production"

```

---

## 🚀 How It Works

The automated GitOps pipeline decouples platform infrastructure management from workload execution across two dedicated Git branches: **`main`** (Platform IaC) and **`deploy`** (Workload Dispatch).

### Phase 1: Declarative IaC Infrastructure Provisioning (`main` Branch)

* **Workflow File:** `.github/workflows/gitops-main.yml`
* **Playbook Entrypoint:** `playbooks/configure_aap.yml` $\rightarrow$ `playbooks/configure_single_gateway.yml`

1. **Trigger & Environment Setup**:
* Pushing to `main` (or running `workflow_dispatch`) spawns a runner inside `quay.io/jopaik/aap-runner:jp1`.
* The pipeline fetches `ANSIBLE_VAULT_PASSWORD` from GitHub Secrets, generates a transient `.vault_pass` file, and applies strict file permissions (`chmod 600`).


2. **Multi-Cluster Loop**:
* `playbooks/configure_aap.yml` loads decrypted credentials from `vars/vault.yml` **first**, followed by declarative platform settings in `vars/aap_config.yml`.
* It iterates sequentially across all target gateway instances defined in `aap_instances` using `loop_var: current_aap` (**AAP-Platform-DC1** and **AAP-Platform-DC2**).


3. **Synchronous Native Provisioning**:
* `playbooks/configure_single_gateway.yml` uses direct native `ansible.controller` modules to establish state on each gateway:
* **Organizations**: Sets up enterprise boundaries (e.g., `Default`).
* **Credentials**: Provisions Machine credentials using vaulted SSH passwords or private keys.
* **Inventories, Groups & Hosts**: Builds `Multi-DC Production Inventory`, creates inventory groups (`dc1`, `dc2`, `production`, `non-production`) via `ansible.controller.group`, and binds host endpoints with host variables.
* **Projects & SCM Sync**: Clones/updates the tracking Git repository (`https://github.com/jopaik/aap-gitops-lac.git`). An explicit `ansible.controller.project_update` step with `wait: true` enforces synchronous SCM synchronization before template definition.
* **Job Templates**: Binds the synchronized Git project and inventory to launchable template definitions (`Deploy Workload Job` pointing to `playbooks/demorun.yml`) with `ask_limit_on_launch: true` enabled.





---

### Phase 2: Targeted & Round-Robin Workload Dispatch (`deploy` Branch)

* **Workflow File:** `.github/workflows/gitops-deploy.yml`
* **Playbook Entrypoint:** `playbooks/dispatch_round_robin.yml`

1. **Trigger & Variable Resolution**:
* Pushing to `deploy` (or running `workflow_dispatch`) triggers the workload dispatch pipeline.
* Dispatch variables (`deploy_job_name`, `deploy_job_limit`, `target_aap`) are loaded from `vars/deploy_job.yml` and can be overridden dynamically using workflow inputs or `extra-vars` (`target_aap_override`, `deploy_job_name_override`, `deploy_job_limit_override`).


2. **Flexible Target Resolution**:
* **Explicit Label/Name Selection**: If `target_aap` is set to a label (`dc1`, `dc2`, `production`, `non-production`) or instance name (`AAP-Platform-DC1`), the playbook inspects `item.labels` and `item.name` across `aap_instances` to target that specific cluster.
* **Auto Round-Robin Fallback**: If `target_aap` is set to `auto` (or omitted), target selection alternates dynamically using integer modulo arithmetic on the active execution context:

$$\text{Target Index} = \text{GITHUB\_RUN\_NUMBER} \pmod{\text{length}(\text{aap\_instances})}$$




3. **Synchronous Execution, Limit Overrides & Diagnostics**:
* Calls `ansible.controller.job_launch` against the resolved target cluster to launch `deploy_job_name` with `limit: {{ deploy_job_limit }}`.
* Sets `wait: true` to poll execution status until completion.
* **API Diagnostic Capture**: If the job fails on the target AAP controller, `ignore_errors: true` allows the playbook to immediately fetch the job stdout directly via AAP REST API (`/api/v2/jobs/<id>/stdout/?format=txt`) and print the underlying runner output before exiting.


4. **Transient Cleanup**:
* A final step enforced by `if: always()` permanently deletes `.vault_pass` from the container workspace.

---
## Key Use Cases

* **Zero-Downtime Operations:** Maintain continuous execution for business-critical automation tasks spanning multiple clusters across multiple geographic regions.

> **Note:** Not all workloads require federated multi-cluster topologies; a single-cluster featuring HA/DR provides a production-ready solution.

* **Seamless Operating System Updates:** Divert current workloads to DC2, permitting maintenance or platform OS upgrades on AAP nodes in DC1 without service interruption.

* **Major Platform Migrations:** Operate distinct AAP version releases side by side across environments, enabling controlled, phased migration of job workloads.

---

## 🔮 Roadmap & Future Enhancements

### 🛡 Reliability & Health

* [ ] **Pre-Flight Cluster Health Checks**: Add optional validation tasks to query gateway API endpoints (`/api/v2/ping/`) before triggering IaC changes or workload runs.

### ⚙ Automation & Lifecycle Management

* [ ] **Automated Workflow Cleanups**: Execute scheduled or triggered repository actions via `gh` CLI / API to purge failed or cancelled workflow run artifacts.
* [ ] **Dynamic Extra-Vars & Prompt Overrides**: Expand `vars/deploy_job.yml` to pass structured `extra_vars` payloads dynamically into job template launches.

### 📍 Multi-DC & Location Intelligence

* [ ] **Geographic & Location-Aware Inventories**: Enhance inventory group mappings to dynamically route jobs to regional execution nodes based on datacenter proximity and workload priority.

```

```