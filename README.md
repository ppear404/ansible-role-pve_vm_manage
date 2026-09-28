# PVE Dev Manager

Ansible automation for creating, snapshotting, and destroying a Proxmox-hosted Rocky Linux dev environment.

The environment is defined once in `vars/main.yml`. Deploy and destroy both use the same
`rocky_vms` list, so VMIDs, names, IP addresses, and gateways stay aligned across the full
lifecycle.

## Playbooks

- `deploy.yml` clones the configured VM list from a Proxmox template, applies cloud-init
  settings, creates a `deployed` snapshot, and starts each VM.
- `snap_running.yml` snapshots the configured VM list using the same VMIDs deployed by
  `deploy.yml`. When `rollback=true`, it rolls back each configured VM to `snapshot_name`
  and starts the VM afterward. It does not deploy or destroy VMs.
- `destroy.yml` stops and removes the configured VM list.

## Required Secrets

The Proxmox API credentials are expected in `inventory/vault.yml`.

GitLab CI requires one of these variables so it can decrypt the vault:

- `ANSIBLE_VAULT_PASSWORD`
- `ANSIBLE_VAULT_PASSWORD_FILE`

## Pipeline Variables

Run the pipeline manually, by API, trigger, or upstream pipeline. The pipeline uses these
variables:

| Variable | Required | Default | Description |
| --- | --- | --- | --- |
| `env_up` | No | `true` | `true` deploys the environment. `false` destroys it. |
| `snap` | No | `false` | `true` runs only the snapshot job. Deploy and destroy are skipped. |
| `rollback` | No | `false` | `true` runs only the snapshot job in rollback mode and starts VMs afterward. |
| `src_id` | Deploy only | none | Numeric VMID of the Proxmox template to clone. Required when `env_up=true`. |
| `ANSIBLE_PLAYBOOK_EXTRA_ARGS` | No | empty | Optional extra arguments appended to the `ansible-playbook` command. |

Examples:

```text
env_up=true
src_id=9000
```

```text
env_up=false
```

```text
snap=true
```

```text
rollback=true
```

`src_id` is intentionally not hardcoded in `vars/main.yml`; pass it when starting a deploy
pipeline so the same code can clone from different templates.

When `snap=true` or `rollback=true`, `env_up` and `src_id` are ignored by the deploy/destroy
job path. The pipeline validates and runs `snap_running.yml` instead.

## VM Inventory

Edit `vars/main.yml` to manage the target environment:

- `rocky_vms[].vmid` is the destination VMID used by both deploy and destroy.
- `rocky_vms[].new_name` is the VM name passed to Proxmox during clone.
- `rocky_vms[].hostname` is applied through cloud-init.
- `rocky_vms[].ip` and `rocky_vms[].gateway` define the VM network settings.

Shared Proxmox and cloud-init settings such as `pve_node`, `storage_target`,
`search_domain`, `nameservers`, and `lifecycle_tag` are also in `vars/main.yml`.
Snapshot settings such as `snapshot_name`, `snapshot_description`, and the default
`rollback` mode are defined there too. `snapshot_name` defaults to `running`; rollback uses
that same snapshot name.

## Local Usage

Deploy:

```bash
ansible-playbook -i localhost, --vault-password-file .vault-pass deploy.yml \
  --extra-vars "src_id=9000"
```

Destroy:

```bash
ansible-playbook -i localhost, --vault-password-file .vault-pass destroy.yml
```

Snapshot configured VMs:

```bash
ansible-playbook -i localhost, --vault-password-file .vault-pass snap_running.yml
```

Roll back configured VMs to `snapshot_name` and start them:

```bash
ansible-playbook -i localhost, --vault-password-file .vault-pass snap_running.yml \
  --extra-vars "rollback=true"
```

## CI Flow

The pipeline has two stages:

1. `validate` selects `snap_running.yml` when `snap=true` or `rollback=true`; otherwise it
   selects `deploy.yml` or `destroy.yml` from `env_up`. It then runs an Ansible syntax check.
2. `manage_vms` runs deploy or destroy when `snap=false` and `rollback=false`.
3. `snapshot_running_vms` runs `snap_running.yml` when `snap=true` or `rollback=true`.

The `proxmox-dev-vms` resource group serializes environment changes so deploy, destroy, and
snapshot jobs do not run against the same VM set at the same time.
