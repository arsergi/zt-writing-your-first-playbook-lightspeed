# Ansible Automation Coding Assistant Lab

A zero-touch RHDP lab where students use the Red Hat Ansible VS Code extension's Automation Coding Assistant to generate, understand, run, and refactor Ansible playbooks and roles — building real multi-tier infrastructure along the way.

## Overview

Students deploy a multi-tier web and database stack across three RHEL 9.6 managed nodes using AI-generated Ansible automation:

- **Web tier** (node1, node2) — Apache httpd with a Jinja2-rendered HTML status page, viewable in dedicated browser tabs
- **Database tier** (node3) — MariaDB server
- **All nodes** — User creation and a dynamic MOTD template with role-based conditionals

The Automation Coding Assistant (backed by LiteMaaS / gpt-oss-120b) generates a complete 7-task playbook from a single prescriptive prompt, and later converts it into a reusable Ansible role.

## Modules

| # | Module | What Students Do |
|---|--------|------------------|
| 01 | Meet Your Automation Coding Assistant | Explore the pre-configured inventory and verify the Ansible extension |
| 02 | Generate a Comprehensive Playbook | Use the coding assistant to generate `system_setup.yml` (7 tasks, 5 variables, handler) |
| 03 | Understand Your Playbook | Walk through conditionals, templates, variables, and the handler |
| 04 | Run and Verify the Playbook | Execute with `ansible-navigator`, verify the status page in browser tabs, check MariaDB and MOTD |
| 05 | Convert Your Playbook to a Role | Use Generate Role to refactor into a role with tasks, handlers, and vars |
| 06 | Wrap-Up and Next Steps | Summary of what was built and where to go next |

## Infrastructure

| Component | Detail |
|-----------|--------|
| **Platform** | AgnosticV / Babylon CNV on RHDP, Showroom content delivery |
| **Control VM** | `devtools-ansible` image — runs code-server (VS Code) on port 8080 + wetty terminal |
| **Managed nodes** | 3 x `rhel-9.6` — node1, node2 (web), node3 (database) |
| **LLM backend** | LiteMaaS endpoint with `gpt-oss-120b` (120B params), configured via `rhcustom` provider |
| **Extension** | Patched Ansible VS Code extension v26.6.0 (bundled vsix, `ms-python.vscode-python-envs` dep removed for code-server compatibility) |

### Student-Facing Tabs (Showroom UI)

| Tab | Purpose |
|-----|---------|
| **VS Code** | Browser-based VS Code on the control VM — edit playbooks, use the coding assistant |
| **Control** | Wetty terminal on the same control VM — run `ansible-navigator` |
| **node1 Web** | HTTP status page deployed by the playbook (visible after Module 04) |
| **node2 Web** | Same status page rendered with node2's Ansible facts |

Both VS Code and the terminal share the same filesystem (`/home/rhel/ansible-files/`) on the control VM — no file-sync mechanism needed.

## Repository Structure

```
config/
  instances.yaml          # 4 VM definitions (control + 3 nodes, all with services/routes)
  networks.yaml           # Single default pod network
  firewall.yaml           # Kubernetes NetworkPolicy rules (egress + ingress incl. port 80)
  secrets.yaml            # Vault-encrypted LiteMaaS API key (vault ID: ansiblebu_vault)

content/
  antora.yml              # Antora component descriptor
  modules/ROOT/pages/     # 6 AsciiDoc lab modules
  modules/ROOT/assets/    # Screenshots
  supplemental-ui/        # Dark theme CSS

setup-automation/
  main.yml                # Orchestrator: inventory, /etc/hosts, setup scripts, SSH keys, vsix
  setup-control.sh        # Control VM: workspace files, templates, Lightspeed config, code-server
  setup-node{1,2}.sh      # Satellite registration + firewalld HTTP
  setup-node3.sh          # Satellite registration only
  ansible-26.6.0.vsix     # Patched extension (rhcustom support, code-server compatible)
  patch_prompts.py        # Patches extension.js for lint rules, multi-file roles, outline fix

runtime-automation/
  main.yml                # Dispatcher for per-module setup/solve/validate
  inventory               # Maps controller/web/database to VMs
  01-playbook-inventory/  # Module 01 automation
  02-generate-comprehensive-playbook/
  03-understand-the-playbook/
  04-playbook-run-it/
  05-generate-roles/
  06-wrap-up/

ansible-files/            # Reference copies of the student workspace
  inventory               # Web group (node1, node2) + database group (node3)
  ansible.cfg             # Points to inventory, SSH timeout
  ansible-navigator.yml   # EE config with --network=host
  system_setup.yml        # Reference playbook (7 tasks)
  templates/motd.j2       # Dynamic MOTD with role-based conditional
  templates/index.html.j2 # Dark-themed HTML status page showing Ansible facts

ui-config.yml             # Showroom tab layout and module definitions
site.yml                  # Antora site config
utilities/health-check.sh # Post-provision connectivity check
```

## Provisioning Flow

1. **Showroom pod** runs `setup-automation/main.yml`
2. Dynamic inventory is built from RHDP environment variables
3. Ansible gathers node IPs via facts and populates `/etc/hosts` on the control VM
4. Per-VM setup scripts run (control: workspace + code-server + extension; nodes: Satellite registration)
5. SSH keypair generated on the showroom pod and distributed to all VMs
6. Vault-encrypted LiteMaaS API key is loaded and passed to the control VM setup
7. Patched extension vsix is installed; `patch_prompts.py` customizes LLM prompt behavior

All file creation uses inline heredocs — no network downloads during provisioning.

## Key Design Decisions

- **Single-VM for edit + run**: VS Code and terminal share one filesystem on the control VM, eliminating file-sync complexity
- **Services/routes on all VMs**: Forces CNV to place every VM on the same 10.0.2.x subnet for reliable cross-node SSH
- **Ansible facts for /etc/hosts**: More reliable than CNV DNS (`getent hosts` returns duplicates); populated before setup scripts run
- **Prescriptive LLM prompts**: Module 02's prompt specifies exact variable names, conditionals, and task order so the generated playbook passes string-match validation
- **Bundled patched vsix**: The `rhcustom` provider (v26.3.4+) requires an extension dependency incompatible with code-server 1.99.3 — the bundled vsix removes it
- **No external dependencies at provision time**: Everything is either baked into the VM image or generated inline
