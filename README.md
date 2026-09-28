# pve-vmdeploy

An Ansible role for cloning template VMs on Proxmox VE, configuring Windows VM
network settings, creating baseline snapshots, and managing VM lifecycle operations.

## Current behavior

The default entry point, `tasks/main.yml`, selects deployment or destruction using
`env_up`. You must supply this variable; the role does not define a default.

| Operation | Windows VMs (`windows_vms`) | Linux VMs (`nix_vms`) |
| --- | --- | --- |
| Clone template (full clone, raw format) | From `win_src_id` | From `nix_src_id` |
| Update cloud-init/network settings | Yes | No |
| Create `deployed` baseline snapshot | Yes | Yes |
| Start after deployment | Yes | No |
| Stop and destroy when `env_up=false` | Yes | No |

Deployment validates both template IDs, clones the VMs, updates Windows settings,
creates baseline snapshots, and then starts the Windows VMs. Windows networking
uses `net0` with VirtIO on `vmbr0` and the configured static IP and gateway.
Destruction attempts to force-stop Windows VMs (ignoring stop errors), then removes
them with `purge: true`.

Snapshot and rollback tasks are available separately through `tasks/snap_running.yml`.
They still iterate over `rocky_vms`, which has no default. See the snapshot example
below to select their targets explicitly. Setting `rollback` alone does not select
these tasks through the default role entry point.

## Requirements

- Ansible with the `community.proxmox` collection and its module dependencies
  installed in the environment executing the role.
- Access to the Proxmox API using a token with permissions for the requested VM,
  storage, and snapshot operations.
- Existing source templates and target storage supporting the requested clones and
  snapshots. Windows templates must support the cloud-init settings used here.
- The role installed as `pve-vmdeploy` in your Ansible roles path.

Install the collection with:

```bash
ansible-galaxy collection install community.proxmox
```

## Configuration

Defaults are in `defaults/main.yml`; `vars/main.yml` is currently empty. Override
settings in your calling playbook, inventory, or extra variables. Replace the
bundled environment values and credential placeholders before use.

| Variable | Default | Purpose |
| --- | --- | --- |
| `env_up` | Not defined; required for the default entry point | `true` deploys; `false` destroys Windows VMs. |
| `pve_node` | `pve1` | Proxmox node used by VM tasks. |
| `storage_target` | `vm-pool01` | Destination storage for full clones. |
| `win_src_id` | `1002` | Numeric Windows template VMID. |
| `nix_src_id` | `9022` | Numeric Linux template VMID. |
| `windows_vms` | Two example VMs | Windows VM definitions. |
| `nix_vms` | Two example VMs | Linux VM definitions. |
| `lifecycle_tag` | `mytag` | Tag applied during cloning and Windows updates. |
| `search_domain` | `example.com` | Windows cloud-init search domain. |
| `nameservers` | `[10.1.154.112]` | Windows cloud-init DNS servers. |
| `api_host`, `api_user`, `api_token_id`, `api_token_secret` | Placeholders | Proxmox API connection and token credentials. |
| `snapshot_name` | `running` | Snapshot name for the separate snapshot/rollback tasks. |
| `snapshot_description` | `Running VM snapshot` | Description for snapshots created by those tasks. |
| `rollback` | `false` | Create a snapshot when false; roll back and start targets when true. |
| `rocky_vms` | Not defined | Target list required by the separate snapshot/rollback tasks. |

Each VM entry supplies `vmid` (destination ID), `new_name` (clone name), `ip` (CIDR),
and `gateway`. Windows updates use `hostname` as the Proxmox VM name, falling back
to `new_name` if it is omitted. Linux networking fields are not currently applied.
Use unique destination VMIDs and replace the example names and addresses.

Both source template IDs must be numeric even if one VM list is empty. Set a VM
list to `[]` to omit that group.

The defaults also contain `node`, `ciuser`, and `cipassword`, but the current tasks
do not use them. All API tasks currently set `validate_certs: false`.

## Deploy or destroy

Create a calling playbook such as `manage.yml`:

```yaml
---
- name: Manage Proxmox VMs
  hosts: localhost
  connection: local
  gather_facts: false
  vars_files:
    - inventory/vault.yml
  roles:
    - role: pve-vmdeploy
      vars:
        pve_node: pve1
        storage_target: vm-pool01
        win_src_id: 1002
        nix_src_id: 9022
        search_domain: example.com
        nameservers:
          - 10.1.154.112
        windows_vms:
          - vmid: 9011
            new_name: dev-dc1
            hostname: dc1
            ip: 10.1.154.141/16
            gateway: 10.1.0.1
        nix_vms: []
```

Create `inventory/vault.yml` yourself and encrypt it with Ansible Vault. Supply
`api_host`, `api_user`, `api_token_id`, and `api_token_secret` there. The role does
not automatically load a vault file.

Deploy:

```bash
ansible-playbook -i localhost, manage.yml --ask-vault-pass \
  --extra-vars '{"env_up": true}'
```

Stop and permanently remove the configured Windows VMs:

```bash
ansible-playbook -i localhost, manage.yml --ask-vault-pass \
  --extra-vars '{"env_up": false}'
```

## Snapshot or roll back

Use `include_role` with `tasks_from` to invoke the snapshot tasks directly. For
example, create `snapshot.yml`:

```yaml
---
- name: Snapshot selected Proxmox VMs
  hosts: localhost
  connection: local
  gather_facts: false
  vars_files:
    - inventory/vault.yml
  tasks:
    - name: Manage snapshots
      ansible.builtin.include_role:
        name: pve-vmdeploy
        tasks_from: snap_running
      vars:
        rocky_vms:
          - vmid: 9011
            new_name: dev-dc1
        snapshot_name: running
```

Select VMIDs matching your deployed environment. These tasks act on every listed
VM; they do not discover or filter VMs by running state. Snapshot targets only
need `vmid` and `new_name`.

Create snapshots:

```bash
ansible-playbook -i localhost, snapshot.yml --ask-vault-pass
```

Roll back to an existing snapshot and start each selected VM afterward:

```bash
ansible-playbook -i localhost, snapshot.yml --ask-vault-pass \
  --extra-vars '{"rollback": true, "snapshot_name": "running"}'
```

The deployment baseline is always named `deployed`; `snapshot_name` only controls
the separate snapshot/rollback tasks. Pass `snapshot_name=deployed` to roll back
to that baseline.

This repository contains role task files, not standalone lifecycle playbooks or a
GitLab CI pipeline. Run the calling playbooks above rather than invoking files
under `tasks/` with `ansible-playbook`.
