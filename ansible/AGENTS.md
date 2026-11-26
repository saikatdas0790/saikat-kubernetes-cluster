# Agent Guidelines for Ansible (this directory)

Ansible-specific rules. Repo-wide rules live in the root `AGENTS.md`; this file carries
only what is particular to working inside `ansible/`, and exists so those rules sit
next to the code they govern.

## Prefer collection modules over raw `uri` / `command` (Hard Rule)

**If a maintained collection module exists for the API or system being touched, use
it — never hand-roll the HTTP request.** Collection modules carry retries, rate-limit
handling, `check_mode` support, idempotency logic and error decoding that a raw `uri`
call does not, and their maintainers test them across versions. That is work we do not
have to redo, and cannot validate as thoroughly ourselves.

`community.hrobot` covers the Hetzner Robot API comprehensively — `server`,
`server_info`, `firewall`, `failover_ip`, `ssh_key`, `reverse_dns`, **`v_switch`**,
`reset`, `storagebox*`. Check before writing any Robot API call.

**Check availability first, and read the module's own docs for parameter names** — do
not assume the shape:

```sh
ls ansible/collections/ansible_collections/<namespace>/<collection>/plugins/modules/
sed -n '/^DOCUMENTATION/,/^EXAMPLES/p' ansible/collections/ansible_collections/community/hrobot/plugins/modules/v_switch.py
```

Two concrete traps found the hard way while writing the vSwitch role:

- **`hetzner_user` / `hetzner_password`**, NOT `api_user` / `api_key` (which is what
  the Namecheap role uses — different API, different names).
- **`servers:` is the list of server identifiers to ADD to the vSwitch.** An empty or
  omitted `servers` does not mean "no change" — it means *no servers added*, so a bare
  create and a create-with-membership are the same call. Omitting it on a
  `state: present` task is correct for creation and wrong for membership.

### Raw `uri` is legacy, not precedent

Roles that call an API with `ansible.builtin.uri` (e.g. `provision` against
`robot-ws.your-server.de`) predate the collection being vendored here. **They are not
a pattern to copy.** When you touch such a call, convert it in the same change.

Fall back to `uri`/`command` **only** when no module exists — and say so in a comment
at the task, naming what you checked, so the next reader does not have to re-derive it.

## Hand-rolled HTTP error handling is a smell

A raw `curl`/`uri` call needs its own status handling and body parsing, and getting
that wrong fails silently. Observed while adding a server to a Hetzner vSwitch by
hand (that role has since been removed, but the lesson is general):

```sh
curl -s -X POST .../vswitch/86224/server --data-urlencode "server=2863794"
# HTTP/2 201, content-length: 0
```

The call **succeeded** (201, empty body) but a `json.load` on the response body threw,
and the follow-up "did it work?" check reported `none` — because the membership takes
a moment to appear. Two false negatives in a row, from code that looked like it was
checking. A collection module returns a structured result and handles the empty-body
201 itself; this is exactly the class of bug the Hard Rule above exists to prevent.

## Duplicate YAML keys are silent — and untested in `.j2` templates

A YAML mapping with a key repeated within it keeps only the **last** occurrence. A second
`extraArgs:` block under kubeadm's `etcd` does not merge — it *replaces* the first, so the
dropped args vanish with no error anywhere. This happened: the etcd listen URLs never
reached a node, which is invisible on Hetzner (the public IP is on the interface) and
fatal on a NAT'd cloud VM.

`ansible-lint` already has `yaml[key-duplicates]` and it catches this in `.yml`/`.yaml`.
**It does not check `.j2` templates** (verified: the same duplicate key is a failure in a
`.yaml` and passes silently in a `.j2`). `ansible/manifests/kubeadm-cluster-configuration.yaml.j2`
is the file that matters here, so check it by hand when editing it — render it and parse
the result, or read the rendered output:

```sh
./.venv/bin/python3 -c "
import yaml,sys; from jinja2 import Environment,FileSystemLoader
e=Environment(loader=FileSystemLoader('ansible/manifests'),trim_blocks=True,lstrip_blocks=True)
yaml.safe_load(e.get_template('kubeadm-cluster-configuration.yaml.j2').render(
    kubernetes_version='1.36.4', control_plane_endpoint='x:6443', pod_network_cidr='10.244.0.0/16',
    service_node_port_range='30000-32767', etcd_heartbeat_interval_milliseconds=250,
    etcd_election_timeout_milliseconds=2500, api_server_cert_sans=['x']))
print('renders as valid YAML')"
```

Note `yaml.safe_load` cannot detect the duplicate itself — it drops the key while parsing.
Only `yaml.compose_all` (which walks the node graph) or ansible-lint's own check sees it.

## Secrets and vault

- Ansible vault is the source for cluster-ops secrets; roles consume `vault_*` vars.
- Use the mise tasks (`mise run ansible-vault-view -- <key>`) rather than piping
  `ansible-vault view` through `grep` — see the root `AGENTS.md`.
- Never edit a SOPS file outside `sops`; a stray edit breaks the MAC.

## Running things

Vast GPU rentals are not Kubernetes nodes. Declare their physical machine IDs and
model names in `inventory-vast/hosts.yml`, then run `mise run vast-inference-deploy`.
The [official Vast CLI](https://docs.vast.ai/cli/reference/ssh-url) resolves a
single active rental and its direct SSH endpoint. It uses the age-encrypted mise
key on the controller only; do not duplicate that key in Ansible Vault. SSH
automatically trusts and records the first host key per rental instance ID and
rejects later changes. The first connection does not independently verify the
server's identity. The CUDA base image's
[Supervisor](https://github.com/vast-ai/base-image) manages the loopback-only
[vLLM runtime](https://docs.vllm.ai/en/stable/getting_started/installation/gpu/).
Deployment checks for the latest stable runtime; model loads use the publisher's
default revision. Neither is polled by the running service. This workflow never
rents or destroys machines.

Azure firewall reconciliation uses `azure.azcollection.azure_rm_securitygroup`
with `auth_source: cli` and an explicit subscription. Read the existing NSG first,
preserve unlisted rules (the module's default), and verify after reconciliation.
Do not run node provisioning merely to update an NSG. Run
`mise run azure-ansible-setup` to install the pinned collection's own SDK
requirements in `.venv-azure`; its SDK versions must not alter the root Python
environment or the independently managed Azure CLI. The module's
[requirements](https://docs.ansible.com/ansible/latest/collections/azure/azcollection/azure_rm_securitygroup_module.html)
come from the selected collection, not a hand-maintained subset.

Use the mise tasks, never `ansible-playbook` / `ansible-lint` / `ansible-vault`
directly (the root `AGENTS.md` lists them). They ensure the `.venv` is used, the
vault password file is found via `ansible.cfg`, and runs are reproducible.

Run playbooks in the foreground. Lint before PRs: `mise run ansible-lint`.

## Waiting

Short poll loops (≤10s sleep) when waiting for something to converge. Never a long
`sleep` — poll, re-check, or stop and let the operator reinitiate.

## systemd units: no native module exists — validate with `systemd-analyze verify`

**There is no Ansible module that CREATES a systemd unit file.** `ansible.builtin.systemd_service`
only *controls* units that already exist (`state`, `enabled`, `masked`, `daemon_reload`), and no
vendored community module fills the gap — `community.general` has `systemd_info` (read facts),
`ini_file` (generic INI editing), `runit`, `openwrt_init`; none writes a unit. So a unit file is
necessarily written by `template` (or `copy`). Checked against the collection sources in
`ansible/collections/` and the module docs, not assumed.

**Because we therefore cannot get type safety from the module, get it from systemd itself.**
Follow the unit-file task with a `systemd-analyze verify` task:

```yaml
- name: Validate the unit file with systemd's own parser
  ansible.builtin.command:
    cmd: systemd-analyze verify {{ unit_path }}
  register: unit_verify
  changed_when: false
  # ! rc alone is not enough — `verify` also reports problems in UNRELATED system units.
  failed_when: >-
    unit_verify.rc != 0
    or (unit_verify.stderr ~ unit_verify.stdout)
       | regex_search('myunit\.service')
```

Why this matters, and why the second condition is not optional: **systemd silently IGNORES
directives it does not recognise.** A typo'd directive name or a removed option does not fail the
unit — it is dropped, the service starts, and the setting simply never takes effect. That is the
same silent-failure class as a dropped `extraArgs:` key. `systemd-analyze verify` fails loudly
instead.

The `regex_search` is load-bearing, measured on Ubuntu 26.04: `verify` prints warnings for other
units in the image (`xfs_scrub_all.service: Support for option CPUAccounting= has been removed`)
and returns 0 for them, so exit code alone is fine — but scoping to our unit name keeps the check
meaningful once a shipped unit in the image genuinely errors.

## Roles and playbooks

- Playbooks in `ansible/playbooks/operations/` are thin single-play wrappers: a `roles:` list or minimal `include_role` for pre/post task files. No conditional logic, loops, or non-role modules in playbooks.
- Roles are atomic, independently executable, and contain only role-specific conditions. Do not call `include_role` inside a role or hide orchestration there.
- A playbook orchestrates the ordered sequence of roles. Add new capabilities as roles to an existing playbook; ask before creating a structurally distinct playbook with a different host target or lifecycle stage.
- Reboot handling belongs only in `base-system`, using `/var/run/reboot-required` and `ansible.builtin.reboot`. Playbooks orchestrate across nodes.
- Multi-node operations are serial: complete and verify one node before starting the next. Use `include_tasks` loops for serial work within a role.
- Error handling belongs in roles: use `fail`, `assert`, appropriate `changed_when`, and explicit failure checks. Playbooks contain orchestration only.
- Run `mise run ansible-lint` before Ansible changes and before review. Playbooks and roles must pass.

The root `AGENTS.md` retains the shared immutable-infrastructure, production-safety, and per-component verification requirements.

### Current lifecycle entry points and limits

Run from the repository root with
`mise run ansible-playbook -- ansible/playbooks/operations/<operation>.yml -e target_host=<node>`.
Use `init-control-plane.yml` for bootstrap, `add-node.yml` for joining either node role,
`remove-node.yml` for retirement, and `upgrade-node.yml` for the existing upgrade path.
Bootstrap runs provisioning and base setup too; it is not merely an idempotent
`kubeadm init` wrapper to rerun against an established control plane.

**Pass an explicit target.** Add and upgrade default to all `nodes`, serially;
bootstrap defaults to `control_plane`. Removal's initial Beszel cleanup play also
defaults to all `nodes`, before the localhost removal role validates `target_host`.
Do not rely on a later validation task to protect an earlier play.

**Existing upgrade behavior is not the immutable provisioning policy.** The active
roles are `upgrade-control-plane` and `upgrade-worker`, not `node-upgrade`. They run
in-place package upgrades and handle their own reboots. Workers retain their Node
registration; control planes reset and rejoin when rebooting. The playbook runs in
inventory order, with no separate control-plane-first phase or inter-node pauses.
These are implementation limitations, not exceptions authorizing new in-place
maintenance or reboot logic outside `base-system`.

**Serial execution and confirmation do not prove quorum safety.** The control-plane
upgrade check counts Kubernetes Ready nodes, including the target; removal has no
pre-removal live-quorum assertion. Before disrupting a voter, verify healthy
surviving etcd voters against the quorum required for that membership. Do not infer
etcd health from Node readiness or assume the roles enforce every safety requirement.

### Retiring temporary operations

Remove fleet-backfill entry points once their configuration belongs to the normal
node lifecycle; retain the roles used by that lifecycle. Do not delete an operation
solely because its header says "one-time" or has a removal TODO: check migration
completion conditions and whether it is the only caller of another capability.
Bootstrap, declarative configuration generation, and read-only diagnostics remain
valid entry points in day-2 management.

RustFS disk preparation runs in `add-node.yml`. RustFS pool membership and local
PV/PVC declarations remain GitOps-managed: `remove-node.yml` refuses a node with
bound RustFS local storage, so decommission its pool and retire its PVCs before
node removal. The node lifecycle does not automatically resize the RustFS cluster.

Beszel agents register through `add-node.yml` and are removed from the hub through
`remove-node.yml`. The historical installation/authentication backfill entry point
is retired; hub authentication setup is not a per-node lifecycle action. Its role
remains available for a separately reviewed hub recovery or configuration change.

Beszel's host-side systemd agent configuration is not expressible by Kubernetes
resources. Use `update-beszel-agents.yml` for approved runtime configuration
changes through the existing agent role, serially and with an explicit target.
Flush restart handlers before checking the running process's HUB_URL, keep its
credential-bearing environment out of logs, and verify hub metrics before moving
to the next node or retiring the previous endpoint. This does not authorize node
provisioning, binary upgrades, or unrelated host changes.

## Inventory: one node list, derived role groups

Every node is declared **once** in `ansible/inventory/00-hosts.yml` under a single
`nodes:` group. `control_plane` and `worker_nodes` are **derived** from each host's
`node_role` by `ansible/inventory/10-role-groups.yml` (the `constructed` inventory
plugin), so the two views cannot drift from the list — which a hand-maintained second
list could not guarantee.

**One operation per lifecycle stage, not one per node kind.** `add-node.yml` and
`upgrade-node.yml` each handle both control planes and workers; the host's `node_role`
decides which roles run. This replaced `add-worker`/`add-control-plane` and
`upgrade-worker`/`upgrade-control-plane`, which had already drifted apart (different
`storage-longhorn` placement; only the control-plane path ran `node_provider_address`,
so a cloud *worker* would have joined advertising an address kubelet cannot use).

Attributes, and what each is actually for:

| Attribute | Set on | Means |
|---|---|---|
| `node_role` | every host (workers inherit `worker` from group vars) | control-plane vs worker — drives the derived groups |
| `stacked_control_plane` | every control plane | CP that **also runs workloads** (taint removed). NOT the same fact as `node_role` |
| `provider` | every host | `hetzner` (Robot API + installimage) vs a cloud VM (image is already ready) |

**Traps, each measured rather than assumed:**

- **`node_role` omission is silent for a control plane.** A CP without it inherits
  `worker`, drops out of `control_plane`, and is then left out of the API server endpoint
  records and skipped by the control-plane path — while the node reports Ready. New
  control planes must set both `node_role: control-plane` and `stacked_control_plane: true`.
- **`stacked_control_plane` is not the control-plane marker.** A dedicated CP is a control
  plane *without* it. Use `node_role` to answer "is this a control plane".
- **The plugin's filename is load-bearing.** Ansible loads inventory sources in filename
  order and a plugin only sees hosts from sources loaded *before* it, so the file is named
  `10-` to sort after `00-hosts.yml`. Renamed below that, every derived group comes out
  empty and nothing errors — 0 control planes, 0 workers, playbooks that "succeed" having
  done nothing. `constructed` is also not enabled by default; `inventory_enabled` in
  `ansible.cfg` lists it.
- **A play's `hosts:` cannot be templated from `hostvars`.** It works at runtime but fails
  `--list-hosts` (`'hostvars' is undefined`), which is why the role views exist as groups
  rather than as a `hosts:` expression computed from host attributes. A `when:` can filter
  tasks but cannot select hosts for a play or produce a set for a cluster-wide aggregate
  (`groups['control_plane'][0]` as the kubectl delegate, `loop: groups['worker_nodes']`,
  the etcd quorum count).

### Inventory changes within an operation

Normal `remove-node.yml` runs `node-remove`, then `node-inventory-remove`, then
`pdns-zones`. Inventory deletion is automated, not a manual post-removal step.
The cleanup-only path skips inventory removal and DNS generation.

After changing inventory on disk, use `ansible.builtin.meta: refresh_inventory` and
assert the resulting in-memory host set before rendering downstream artifacts.
A successful file edit alone does not prove `groups` and `hostvars` are current.
Constructed groups disappear when their last matching host is removed; use
`groups['worker_nodes'] | default([])` where an empty group is valid.

Add and normal removal regenerate DNS declarations locally; they do not deploy
them. Commit and push the resulting inventory and generated DNS changes, then
verify authoritative DNS convergence through the root GitOps workflow. Registrar
glue changes and returning provider hardware are separate operations.
