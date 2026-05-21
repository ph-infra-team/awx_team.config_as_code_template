# AWX Team Configuration as Code Template

Enterprise Ansible collection for onboarding and maintaining team-owned AWX or Red Hat Ansible Automation Platform resources as code.

Collection identity:

```yaml
namespace: awx_team
name: config_as_code_template
version: 1.0.0
```

This collection provides local roles and an example controller playbook that teams can use to declaratively create and update AWX/AAP notification templates, credentials, projects, inventories, inventory sources, schedules, job templates, and workflow job templates.

## Scope

This repository is a reusable template collection for team onboarding. It is not the central platform configuration repository and should not own global controller administration unless explicitly adopted for that purpose.

Managed resources:

- Notification templates
- Source control and other controller credentials
- SCM projects
- Inventories
- SCM inventory sources and immediate source syncs
- Schedules
- Job templates
- Workflow job templates
- Workflow nodes, approvals, success paths, failure paths, and always paths

Out of scope:

- Controller installation
- Organization and team creation, unless handled by a separate platform onboarding workflow
- Execution environment builds
- Automation Hub administration
- EDA administration
- Destructive resource deletion

## Repository Layout

```text
awx_team.config_as_code_template/
|-- README.md
|-- galaxy.yml
|-- meta/
|   `-- runtime.yml
|-- playbooks/
|   `-- controller_config.yml
|-- docs/
|   `-- examples/
|       `-- team_vars.yml
|-- roles/
|   |-- awx_notification_template/
|   |-- awx_credential/
|   |-- awx_project/
|   |-- awx_inventory/
|   |-- awx_inventory_source/
|   |-- awx_schedule/
|   |-- awx_job_template/
|   `-- awx_workflow_job_templates/
`-- plugins/
    `-- README.md
```

| Path | Purpose |
| --- | --- |
| `galaxy.yml` | Collection metadata, version, repository, license, and dependency on `awx.awx >= 24.0.0`. |
| `meta/runtime.yml` | Ansible runtime requirement. Currently requires Ansible `>=2.14`. |
| `playbooks/controller_config.yml` | Example playbook that loads `team_vars.yml`, loads `vaults/{{ env }}.yml`, installs collections, and applies roles. |
| `docs/examples/team_vars.yml` | Example team input file showing supported `controller_*_all` variable structures. |
| `roles/*` | Resource-specific AWX/AAP automation roles. |

## Architecture

Teams provide declarative YAML lists in `team_vars.yml`. Secrets are loaded from `vaults/{{ env }}.yml`. The example playbook then applies each role in dependency order.

Execution order in `playbooks/controller_config.yml`:

1. Load `team_vars.yml`.
2. Load environment vault from `vaults/{{ env }}.yml`.
3. Install required collections from `collections/requirements.yml`.
4. Create notification templates.
5. Create credentials.
6. Create projects.
7. Create inventories.
8. Create inventory sources and sync them.
9. Create schedules.
10. Create job templates.
11. Create workflow job templates and workflow nodes.
12. Send an SMTP failure notification if the managed block fails.

The roles use `awx.awx` modules and connect to the controller through these variables:

```yaml
CONTROLLER_HOST:
CONTROLLER_USERNAME:
CONTROLLER_PASSWORD:
CONTROLLER_VERIFY_SSL:
```

These values must come from vault or another approved secret source in production.

## Prerequisites

Execution host or automation runner requirements:

- Ansible Core `>=2.14`
- `ansible-galaxy`
- Collection dependency `awx.awx >=24.0.0`
- Network access to the AWX/AAP controller
- Network access to SCM repositories used by project definitions
- Controller permissions to manage resources in target organizations
- Vault password or approved secret identity for the target environment

Install the collection dependency:

```bash
ansible-galaxy collection install awx.awx
```

Build and install this collection locally from the repository root:

```bash
ansible-galaxy collection build
ansible-galaxy collection install awx_team-config_as_code_template-1.0.0.tar.gz --force
```

## Consumer Repository Pattern

A team repository using this collection should include:

```text
team_automation/
|-- setup_awx.yml
|-- team_vars.yml
|-- hosts
|-- collections/
|   `-- requirements.yml
`-- vaults/
    |-- dev.yml
    |-- test.yml
    `-- prod.yml
```

Example `collections/requirements.yml`:

```yaml
---
collections:
  - name: awx.awx
    version: ">=24.0.0"
  - name: git+http://gitlab.midhtech.local/infra_team/awx_team.config_as_code_template.git
    type: git
    version: main
```

Use CI-injected credentials or approved Git credential helpers for private collection sources. Do not commit personal access tokens into `collections/requirements.yml`.

## Variable Contract

### Required Controller Connection Variables

Store these in `vaults/{{ env }}.yml`:

```yaml
CONTROLLER_HOST: "https://controller.example.com"
CONTROLLER_USERNAME: "service-account"
CONTROLLER_PASSWORD: "vaulted-password"
CONTROLLER_VERIFY_SSL: true
```

### Runtime Selectors

Define the target environment and organization naming values in `team_vars.yml`:

```yaml
env: dev
platform_owner_org: PLATFORM
config_as_code_org: config_as_code
```

### Notification Templates

Variable:

```yaml
controller_notifications_all:
```

Required fields:

- `name`
- `type`
- `organization`
- `config`

Use vault variables for SMTP passwords and recipients.

### Credentials

Variable:

```yaml
controller_credentials_all:
```

Required fields:

- `name`
- `organization`
- `credential_type`
- `inputs`

Credential `inputs` must not contain plaintext secrets in production.

### Projects

Variable:

```yaml
controller_projects_all:
```

Required fields:

- `scm_url`
- `organization`

Common fields:

- `name`
- `scm_type`
- `scm_branch`
- `credential`
- `update_project`

### Inventories

Variable:

```yaml
controller_inventories_all:
```

Common fields:

- `name`
- `organization`
- `description`

### Inventory Sources

Variable:

```yaml
controller_inventory_sources_all:
```

Required fields:

- `name`
- `inventory`
- `organization`
- `source`
- `source_project`
- `source_path`
- `credential`

The role creates the inventory source and immediately runs `awx.awx.inventory_source_update` with `wait: true`.

### Schedules

Variable:

```yaml
controller_schedules_all:
```

Common fields:

- `name`
- `description`
- `unified_job_template`
- `rrule`

### Job Templates

Variable:

```yaml
controller_job_templates_all:
```

Required fields:

- `name`
- `project`
- `playbook`
- `inventory`
- `execution_environment`

Common optional fields:

- `description`
- `job_type`
- `credentials`
- `labels`
- `limit`
- `verbosity`
- `extra_vars`
- `forks`
- `job_tags`
- `skip_tags`
- `timeout`
- `ask_variables_on_launch`
- `ask_inventory_on_launch`
- `ask_scm_branch_on_launch`

### Workflow Job Templates

Variable:

```yaml
controller_workflow_job_templates_all:
```

Required fields:

- `name`
- `organization`
- `inventory`
- `nodes`

Supported node fields:

- `identifier`
- `unified_job_template`
- `approval_node`
- `success_nodes`
- `failure_nodes`
- `always_nodes`
- `all_parents_must_converge`

Workflow creation happens in two phases:

1. Create workflow templates and nodes without dependencies.
2. Link node dependencies with success, failure, and always paths.

## Running the Example Playbook

The example playbook expects `team_vars.yml`, `vaults/{{ env }}.yml`, and `collections/requirements.yml` to exist relative to the execution directory.

Run from a consumer repository:

```bash
ansible-playbook -i hosts setup_awx.yml -e env=dev --ask-vault-pass
```

Run the collection example directly after preparing the expected files:

```bash
ansible-playbook -i localhost, -c local playbooks/controller_config.yml -e env=dev --ask-vault-pass
```

Run syntax validation:

```bash
ansible-playbook -i localhost, -c local playbooks/controller_config.yml --syntax-check --ask-vault-pass
```

List tasks:

```bash
ansible-playbook -i localhost, -c local playbooks/controller_config.yml --list-tasks --ask-vault-pass
```

List tags:

```bash
ansible-playbook -i localhost, -c local playbooks/controller_config.yml --list-tags --ask-vault-pass
```

## Tags

Roles expose tags for targeted execution:

| Tag | Resource type |
| --- | --- |
| `notifications` | Notification templates |
| `credentials` | Credentials |
| `projects` | SCM projects |
| `inventories` | Inventories |
| `inventory_sources` | Inventory sources |
| `schedules` | Schedules |
| `job_templates` | Job templates |
| `workflow_job_templates` | Workflow templates |
| `workflow_job_template_nodes` | Workflow nodes and dependency links |

Example:

```bash
ansible-playbook -i hosts setup_awx.yml -e env=dev --ask-vault-pass --tags projects
```

## Secret Management

Create a vault file:

```bash
ansible-vault create vaults/dev.yml
```

Edit a vault file:

```bash
ansible-vault edit vaults/dev.yml
```

View a vault file:

```bash
ansible-vault view vaults/dev.yml
```

Production vault requirements:

- Controller credentials must be vaulted.
- SCM usernames, passwords, deploy tokens, and PATs must be vaulted.
- SMTP usernames, app passwords, senders, and recipients must be vaulted.
- Do not copy plaintext values from `docs/examples/team_vars.yml` into production files.
- Prefer service accounts over personal accounts.
- Rotate secrets on a defined enterprise schedule.

## Enterprise Onboarding Workflow

1. Platform team creates or approves the target AWX/AAP organization and base team.
2. Team creates a consumer repository from this template pattern.
3. Team defines resources in `team_vars.yml`.
4. Team stores environment secrets in `vaults/<env>.yml`.
5. Team runs syntax checks and a dev deployment.
6. Platform team reviews permissions, credentials, schedules, and workflow paths.
7. Team promotes the same reviewed configuration through `test` and `prod`.
8. Execution evidence is attached to the change record.

## Governance

### Change Control

- All resource changes must go through merge requests.
- Production changes require platform-owner approval.
- Workflow, credential, schedule, and inventory source changes require extra review.
- Collection version changes must be intentional and documented.

### Security

- Use least-privilege controller service accounts.
- Scope SCM credentials to required repositories only.
- Avoid shared admin credentials for team automation.
- Keep notification recipients approved for the environment.
- Use TLS certificate validation in production where enterprise PKI supports it.

### Idempotency

Roles are designed to converge resources to the desired state. Repeated runs should be safe when definitions are stable.

Before production execution:

- Review the Git diff.
- Validate YAML.
- Run in `dev`.
- Confirm project syncs and inventory source syncs succeed.
- Confirm schedules and workflow paths are intentional.

### Naming Standards

Recommended names:

- Credential: `<org> <system>`
- Project: `<org> <application> Project`
- Inventory: `<org> <application> Inventory`
- Inventory source: `<org> <application> Inventory Source`
- Job template: `<org> <application> <action>`
- Workflow: `<org> <business process> Workflow`

## Troubleshooting

### Controller Authentication Failed

Check:

- `CONTROLLER_HOST`
- `CONTROLLER_USERNAME`
- `CONTROLLER_PASSWORD`
- `CONTROLLER_VERIFY_SSL`
- Service account permissions in the target organization

### Collection Not Found

Check:

- `awx.awx` is installed.
- `awx_team.config_as_code_template` is installed.
- `collections/requirements.yml` points to the correct repository and version.
- Private GitLab credentials are available to `ansible-galaxy`.

### Variable Validation Failed

Each role asserts required fields before creating resources. Review the failed item in the task output and update the matching `controller_*_all` list.

### Project Sync Failed

Check:

- SCM URL is reachable from the controller.
- SCM credential has repository access.
- Branch exists.
- Project path contains the referenced playbooks or inventory files.

### Inventory Source Sync Failed

Check:

- `source_project` exists and is synced.
- `source_path` exists in the project repository.
- Inventory file syntax is valid.
- Source credential has access.

### Workflow Nodes Did Not Link

Check:

- Node identifiers are unique within the workflow.
- `success_nodes`, `failure_nodes`, and `always_nodes` reference existing identifiers.
- Approval nodes use the `approval_node` structure.

## Maintenance Notes

- Keep example values generic and non-sensitive.
- Keep new role behavior documented in this README and the role README.
- Keep collection metadata in `galaxy.yml` current before publishing.
- Increment the collection version for released changes.
- Preserve Ansible `>=2.14` compatibility unless a version upgrade is approved.
