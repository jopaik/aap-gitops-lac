# AAP 2.6 Multi-Cluster GitOps Pipeline

A production-grade GitOps repository designed to declaratively configure Red Hat Ansible Automation Platform (AAP) 2.6 gateway clusters and dispatch workload jobs using GitHub Actions and native `ansible.controller` modules.

---

## 🏗️ Pipeline Architecture & Workflow

```text
+---------------------------------------------------------------------------------+
|                                 GITHUB REPOSITORY                               |
|                                                                                 |
|   vars/aap_config.yml     playbooks/configure_aap.yml     playbooks/demorun.yml |
|   vars/vault.yml          playbooks/dispatch_round_robin.yml                    |
+---------------------------------------------------------------------------------+
                                       |
                                       | Git Push to 'main'
                                       v
+---------------------------------------------------------------------------------+
|                           GITHUB ACTIONS WORKFLOW                               |
|                     Container: quay.io/jopaik/aap-runner:jp1                    |
|                                                                                 |
|  1. Generate .vault_pass from ANSIBLE_VAULT_PASSWORD Secret                    |
|  2. Execute playbooks/configure_aap.yml (IaC Cluster Provisioning)               |
|  3. Execute playbooks/dispatch_round_robin.yml (Workload Dispatch)              |
|  4. Clean up .vault_pass                                                        |
+---------------------------------------------------------------------------------+
           |                                                      |
           | Step 2: IaC Provisioning                             | Step 3: Job Launch
           | Syncs Organizations, Credentials,                    | Dispatches Job Template via
           | Projects, Inventories, Hosts & Templates             | GITHUB_RUN_NUMBER % 2
           v                                                      v
+----------------------------------+                   +----------------------------------+
|       AAP PLATFORM ALPHA         |                   |        AAP PLATFORM BETA         |
|  [https://control-nkdv8.apps](https://control-nkdv8.apps)...   |                   |  [https://control-6c7mh.apps](https://control-6c7mh.apps)...   |
|                                  |                   |                                  |
|  • Org: Default                  |                   |  • Org: Default                  |
|  • Credential: Production Key    |                   |  • Credential: Production Key    |
|  • Project: GitOps Application   |                   |  • Project: GitOps Application   |
|  • Inventory: Multi-DC           |                   |  • Inventory: Multi-DC           |
|  • Template: Deploy Workload Job |                   |  • Template: Deploy Workload Job |
+----------------------------------+                   +----------------------------------+
                 \                                                     /
                  \---> [ Executed Job Run on Selected Cluster ] <-----/


                  .
├── .github/
│   └── workflows/
│       └── gitops-pipeline.yml     # GitHub Actions workflow running in Quay execution container
├── playbooks/
│   ├── configure_aap.yml           # Entrypoint playbook iterating across all Gateway clusters
│   ├── configure_single_gateway.yml # Direct native ansible.controller module tasks
│   ├── demorun.yml                 # Target workload playbook executed on AAP execution nodes
│   └── dispatch_round_robin.yml    # Workload dispatch playbook targeting gateway instances
├── vars/
│   ├── aap_config.yml              # Declarative IaC variables (Orgs, Credentials, Projects, Inventories, Templates)
│   └── vault.yml                   # Encrypted secret credentials (Vault URLs, passwords, Machine credentials)
├── .vault_pass                     # Transient vault password file (generated and purged during pipeline execution)
└── README.md                       # Architecture diagram and repository documentation



# How It Works

The automated GitOps pipeline runs inside the custom container (`quay.io/jopaik/aap-runner:jp1`) and executes in two primary phases on every push or manual workflow dispatch on the `main` branch.

---

## Phase 1: Declarative IaC Infrastructure Provisioning
**Playbook Entrypoint:** `playbooks/configure_aap.yml` $\rightarrow$ `playbooks/configure_single_gateway.yml`

1. **Vault & Runner Initialization**
   * The pipeline reads the `ANSIBLE_VAULT_PASSWORD` GitHub secret and generates a temporary `.vault_pass` file (`chmod 600`).
   * `playbooks/configure_aap.yml` loads declarative definitions from `vars/aap_config.yml` and decrypted secrets from `vars/vault.yml`.

2. **Sequential Multi-Cluster Targeted Iteration**
   * The entrypoint playbook loops over all target gateway nodes in `aap_instances` (**AAP-Platform-Alpha** and **AAP-Platform-Beta**).

3. **Direct Native Synchronization**
   * `playbooks/configure_single_gateway.yml` invokes direct native `ansible.controller` modules synchronously on each cluster to establish state:
     * **Organizations**: Provisions enterprise management boundaries (e.g., `Default`).
     * **Credentials**: Configures Machine credentials using vaulted passwords/SSH keys for target hosts.
     * **Projects**: Syncs the Git repository (`https://github.com/jopaik/aap-gitops-lac.git`) with `wait: true` to ensure SCM repository cloning completes before downstream steps.
     * **Inventories, Hosts & Groups**: Sets up multi-datacenter inventory groups (`dc1`, `dc2`) and registers individual host endpoints with variables.
     * **Job Templates**: Links the synchronized Git project and inventory to create launchable execution templates (`Deploy Workload Job` pointing to `playbooks/demorun.yml`).

---

## Phase 2: Dynamic Round-Robin Workload Dispatch
**Playbook Entrypoint:** `playbooks/dispatch_round_robin.yml`

1. **GitHub Pipeline Context Evaluation**
   * The playbook reads the environment variable `GITHUB_RUN_NUMBER` generated automatically by GitHub Actions (e.g., Run #1, Run #2, Run #3...).

2. **Modulo Cluster Selection**
   * Target selection is calculated dynamically using modulo arithmetic across the array length of available instances:
     $$\text{Target Index} = \text{GITHUB\_RUN\_NUMBER} \pmod{\text{length}(\text{aap\_instances})}$$
   * **Run #1**: `1 % 2 = 1` $\rightarrow$ Targets **AAP-Platform-Beta**
   * **Run #2**: `2 % 2 = 0` $\rightarrow$ Targets **AAP-Platform-Alpha**
   * **Run #3**: `3 % 2 = 1` $\rightarrow$ Targets **AAP-Platform-Beta**

3. **Synchronous Job Launch**
   * The playbook calls `ansible.controller.job_launch` against the calculated target gateway cluster to execute `Deploy Workload Job`.
   * It waits (`wait: true`) until job execution finishes on the AAP platform and returns the final job status logs to the GitHub Actions runner.

4. **Transient Cleanup**
   * In a final step guaranteed by `if: always()`, the pipeline deletes `.vault_pass` from the runner context.