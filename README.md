# Active-Active AAP Multi-Cluster with GitOps Pipeline

A friendly, production-grade GitOps pipeline for Active-Active multi-cluster AAP setups! ⚡ Uses GitHub Actions and ansible.controller modules to manage declarative platform configuration on main and handle workload dispatch on deploy.

---

## 🏗️ Pipeline Architecture & Workflow

```text
+---------------------------------------------------------------------------------+
|                                 GITHUB REPOSITORY                               |
|                                                                                 |
|   vars/aap_config.yml     playbooks/configure_aap.yml     playbooks/demorun.yml |
|   vars/vault.yml          playbooks/dispatch_round_robin.yml                    |
+---------------------------------------------------------------------------------+
                     /                               \
    Git Push to 'main'                             Git Push to 'deploy'
                   /                                   \
                  v                                     v
+-----------------------------------+   +-----------------------------------+
|  WORKFLOW: gitops-main.yml        |   |  WORKFLOW: gitops-deploy.yml      |
|  Container: aap-runner:jp1        |   |  Container: aap-runner:jp1        |
|                                   |   |                                   |
|  1. Generate .vault_pass          |   |  1. Generate .vault_pass          |
|  2. Execute configure_aap.yml     |   |  2. Execute dispatch_round_robin  |
|  3. Clean up .vault_pass          |   |  3. Clean up .vault_pass          |
+-----------------------------------+   +-----------------------------------+
                  |                                       |
                  | Phase 1: IaC                          | Phase 2: Job Launch
                  | Syncs Orgs, Credentials,              | Dispatches Job Template via
                  | Projects, Inventories, Templates      | GITHUB_RUN_NUMBER % 2
                  v                                       v
+----------------------------------+     +----------------------------------+
|       AAP PLATFORM ALPHA         |     |        AAP PLATFORM BETA         |
|                                  |.    |                                  |
|                                  |     |                                  |
|  • Org: Default                  |     |  • Org: Default                  |
|  • Credential: Production Key    |     |  • Credential: Production Key    |
|  • Project: GitOps Application   |     |  • Project: GitOps Application   |
|  • Inventory: Multi-DC           |     |  • Inventory: Multi-DC           |
|  • Template: Deploy Workload Job |     |  • Template: Deploy Workload Job |
+----------------------------------+     +----------------------------------+
                 \                                       /
                  \---> [ Executed Job Run on Selected Cluster ] <------/

# 📁 Repository Structure & Workflow Guide

This document outlines the repository layout and execution workflow for 
the active-active multi-cluster Red Hat Ansible Automation Platform (AAP) GitOps pipeline.

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
│   ├── configure_single_gateway.yml # Direct native ansible.controller module tasks
│   ├── demorun.yml                 # Target workload playbook executed on AAP execution nodes
│   └── dispatch_round_robin.yml    # Workload dispatch playbook targeting gateway instances
├── vars/
│   ├── aap_config.yml              # Declarative IaC variables (Orgs, Credentials, Projects, Inventories, Templates)
│   └── vault.yml                   # Encrypted secret credentials (Vault URLs, passwords, Machine credentials)
├── .vault_pass                     # Transient vault password file (generated and purged during pipeline execution)
└── README.md                       # High-level architecture and pipeline documentation

# 🚀 How It Works

The automated GitOps pipeline decouples platform infrastructure management from workload execution across two dedicated Git branches: **`main`** (Platform IaC) and **`deploy`** (Workload Dispatch).

---

## Phase 1: Declarative IaC Infrastructure Provisioning (`main` Branch)
**Workflow File:** `.github/workflows/gitops-main.yml`  
**Playbook Entrypoint:** `playbooks/configure_aap.yml` $\rightarrow$ `playbooks/configure_single_gateway.yml`

1. **Trigger & Environment Setup**
   * Pushing to `main` (or running `workflow_dispatch`) spawns a runner inside `quay.io/jopaik/aap-runner:jp1`.
   * The pipeline fetches `ANSIBLE_VAULT_PASSWORD` from GitHub Secrets, generates a transient `.vault_pass` file, and applies strict file permissions (`chmod 600`).

2. **Multi-Cluster Loop**
   * `playbooks/configure_aap.yml` loads declarative platform settings from `vars/aap_config.yml` and decrypted credentials from `vars/vault.yml`.
   * It iterates sequentially across all target gateway nodes in `aap_instances` (**AAP-Platform-Alpha** and **AAP-Platform-Beta**).

3. **Synchronous Native Provisioning**
   * `playbooks/configure_single_gateway.yml` uses direct native `ansible.controller` modules to establish state on each AAP cluster:
     * **Organizations**: Sets up enterprise boundaries (e.g., `Default`).
     * **Credentials**: Provisions Machine credentials using vaulted SSH passwords or private keys.
     * **Projects**: Synchronizes the Git repository (`https://github.com/jopaik/aap-gitops-lac.git`) tracking the `deploy` branch. Setting `wait: true` enforces synchronous cloning before downstream resource binding.
     * **Inventories, Hosts & Groups**: Builds multi-datacenter inventory groups (`dc1`, `dc2`) and populates host endpoints with host variables.
     * **Job Templates**: Binds the synchronized Git project and inventory to launchable template definitions (`Deploy Workload Job` pointing to `playbooks/demorun.yml`).

---

## Phase 2: Dynamic Round-Robin Workload Dispatch (`deploy` Branch)
**Workflow File:** `.github/workflows/gitops-deploy.yml`  
**Playbook Entrypoint:** `playbooks/dispatch_round_robin.yml`

1. **Trigger & Context Evaluation**
   * Pushing to `deploy` (or running `workflow_dispatch`) triggers the workload dispatch pipeline.
   * The job retrieves `GITHUB_RUN_NUMBER` provided automatically by the active GitHub Actions execution context (Run #1, Run #2, Run #3...).

2. **Modulo Target Selection**
   * Target selection alternates dynamically between clusters using integer modulo arithmetic:
     $$\text{Target Index} = \text{GITHUB\_RUN\_NUMBER} \pmod{\text{length}(\text{aap\_instances})}$$
   * **Run #1**: $1 \pmod 2 = 1 \longrightarrow$ Targets **AAP-Platform-Beta**
   * **Run #2**: $2 \pmod 2 = 0 \longrightarrow$ Targets **AAP-Platform-Alpha**
   * **Run #3**: $3 \pmod 2 = 1 \longrightarrow$ Targets **AAP-Platform-Beta**

3. **Synchronous Execution & Logging**
   * Calls `ansible.controller.job_launch` against the calculated target cluster to run `Deploy Workload Job`.
   * Sets `wait: true` to poll execution status until completion and streams output back to the runner logs.

4. **Transient Cleanup**
   * A final step enforced by `if: always()` permanently deletes `.vault_pass` from the container workspace.e AAP platform and returns the final job status logs to the GitHub Actions runner.

4. **Transient Cleanup**
   * In a final step guaranteed by `if: always()`, the pipeline deletes `.vault_pass` from the runner context.