# Ansible node setup

Provisions a new Linux node for the Docker Swarm infrastructure in this
repository:

- installs Docker CE on a supported RHEL-family distribution
- initializes a new Swarm or joins an existing one
- applies Swarm node labels
- restricts Swarm ports to a configured private interface and opens published
  ports separately in `firewalld` when it is active
- optionally deploys or updates the existing
  [compose.portainer.yml](../compose.portainer.yml) when `portainer_deploy` is
  true

The playbook does not deploy the infra or monitoring stacks, and it does not
render `.env` or create Docker secrets. It is intended for new nodes: the Docker
role renders `/etc/docker/daemon.json` from `docker_engine_daemon_options` and
replaces the existing file rather than merging it, so registry or networking
settings already in that file are lost unless they are included in the
variables. Set `docker_engine_manage_daemon_config: false` to leave the file
untouched. If the rendered file differs, Docker is restarted.

## Prerequisites

- Ansible with Python 3 on the control node.
- SSH access to the target nodes. Both plays run with `become: true`, so the SSH
  user needs root or sudo privileges; add `-K` to the playbook commands when
  sudo requires a password.
- AlmaLinux/RHEL/Rocky/CentOS 8, 9, or 10. Docker CE repository support must be
  verified before extending `docker_engine_supported_major_versions`.
- A private-network interface shared by the Swarm nodes. When `firewall_manage`
  is true, set `firewall_swarm_interface` to that interface on each node; if
  `firewalld` is active, the playbook assigns it to the `internal` zone and opens
  `2377/tcp`, `7946/tcp`, `7946/udp`, and `4789/udp` there. Portainer's
  `9443/tcp` is opened separately in the `public` zone. Add `80/tcp` and
  `443/tcp` to `firewall_extra_ports` when Traefik runs on the node.

Docker manages iptables rules for published container ports, but it does not
open `firewalld` for Swarm control and overlay traffic. If `firewalld` is active,
set `firewall_swarm_interface` to the private interface, or set
`firewall_manage: false` to manage rules outside Ansible; then restrict Swarm
ports to the private network and configure published-port rules externally. The
interface binding moves that
interface into firewalld's `internal` zone; published ports remain in `public`.
The `internal` zone trusts the network and applies all of that zone's default
and locally configured rules, not just the Swarm ports listed above. Use only
an interface connected to the trusted private network.

## Install the collection

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
```

## Configure the inventory

```bash
cp inventory.example.yml inventory.yml
cp group_vars/all.yml.example group_vars/all.yml
```

`inventory.yml` and `group_vars/all.yml` are ignored by git. Replace the example
addresses and labels.

- `new_nodes` contains the hosts provisioned by this run.
- `swarm_managers` is used to deploy Portainer once. For a worker-only run, keep
  an existing manager in this group and leave it out of `new_nodes`.
- `swarm_role` must be `init`, `join_manager`, or `join_worker`.
- `swarm_node_labels` is a map applied with `docker node update --label-add`.
  Use string values such as `"true"` to match the compose placement
  constraints.

## Create the join token file

Only `join_manager` and `join_worker` need a token.

```bash
cp secrets/swarm.yml.example secrets/swarm.yml
chmod 600 secrets/swarm.yml
```

Set `swarm_join_token` in `secrets/swarm.yml`. Get it from an existing manager
and use the token type that matches `swarm_role`:

```bash
docker swarm join-token worker -q
# or
docker swarm join-token manager -q
```

`swarm_manager_addr` belongs in `group_vars/all.yml`; use the manager's
private-network host without `:2377`. It must match the manager's
`--advertise-addr`: the playbook verifies membership against it, and a node
already in another Swarm fails with an error instead of joining silently.

## Bootstrap the first manager

```bash
cd ansible
ansible-playbook node-setup.yml --limit node1
```

This installs Docker, initializes Swarm, and applies the manager labels.
Portainer is deployed only when `portainer_deploy` is true. After it finishes,
put a worker token into `secrets/swarm.yml`.

## Add workers or managers

Run joins one role at a time, with a token that matches that role. A manager
token joins the target as a manager, so do not share one `swarm_join_token`
across a mixed set of managers and workers. Join workers with a worker token:

```bash
cd ansible
ansible-playbook node-setup.yml \
  -e @secrets/swarm.yml \
  --limit node2
```

Join additional managers with a manager token in `secrets/swarm.yml` and a run
limited to those hosts:

```bash
ansible-playbook node-setup.yml \
  -e @secrets/swarm.yml \
  --limit <new-manager>
```

For a worker node with labels, `swarm_label_delegate` must name an existing
manager that is present in the inventory. Label updates are delegated there
because `docker node update` only works on managers.

Portainer runs only when `portainer_deploy` is true and the first manager listed
in `swarm_managers` is in the play; otherwise Portainer is not touched.

Port `9443` is published through the Swarm routing mesh, so it is reachable on
every node, and the firewall role opens `9443/tcp` in the public zone whenever
`firewall_extra_ports` lists it. Restrict 9443/tcp on every Swarm node in a way
this playbook does not override (keep it out of `firewall_extra_ports` and
manage the rule outside `firewall_manage`), then set `portainer_deploy: true`
and run with the join inputs. A public-zone input rule alone may not block
Docker's Swarm ingress forwarding, so keep the upstream restriction active
during first deployment:

```bash
cd ansible
ansible-playbook node-setup.yml -e @secrets/swarm.yml
```

Running the playbook deploys Portainer; it does not verify ingress access. After
Portainer is running, test `9443/tcp` on every node from a source that should be
allowed and from one that should be denied.

Use a full-inventory run only when `new_nodes` contains a single join role; if it
mixes managers and workers, use the per-role `--limit` runs above so each role
gets its matching token.

Without `-e @secrets/swarm.yml` a full-inventory run fails the join validation
for the example worker and aborts the run (`any_errors_fatal`), so Portainer
never deploys. To deploy or update only Portainer, limit the run to that first
manager and select only the `portainer` tag so the provisioning roles do not
run:

```bash
ansible-playbook node-setup.yml --limit <first-manager> --tags portainer
```

## Configuration reference

| Variable                             | Default                 | Purpose                                                                                                   |
| ------------------------------------ | ----------------------- | --------------------------------------------------------------------------------------------------------- |
| `swarm_role`                         | none                    | `init`, `join_manager`, or `join_worker`                                                                  |
| `swarm_manager_addr`                 | none                    | Existing manager host for joins                                                                           |
| `swarm_join_token`                   | none                    | Manager or worker join token, supplied externally                                                         |
| `swarm_advertise_addr`               | primary IPv4            | Address advertised to the cluster; set per host to a private address when the primary IPv4 is not private |
| `swarm_node_labels`                  | `{}`                    | Labels applied to the node                                                                                |
| `swarm_label_delegate`               | none                    | Existing manager used to label workers                                                                    |
| `docker_engine_remove_conflicts`     | `false`                 | Remove Docker-conflicting packages (example file sets `true`)                                             |
| `docker_engine_conflict_packages`    | podman/buildah/runc set | Explicit package list for conflict removal                                                                |
| `docker_engine_manage_daemon_config` | `true`                  | Render and replace `/etc/docker/daemon.json`; set `false` to leave it untouched                           |
| `docker_engine_daemon_options`       | `log-driver: journald`  | Docker daemon JSON options                                                                                |
| `firewall_manage`                    | `true`                  | Configure active `firewalld`; set `false` to manage rules outside Ansible                                 |
| `firewall_swarm_interface`           | empty                   | Private interface assigned to the `internal` zone                                                         |
| `firewall_swarm_zone`                | `internal`              | Zone for Swarm-only ports                                                                                 |
| `firewall_public_zone`               | `public`                | Zone for published ports                                                                                  |
| `firewall_extra_ports`               | `9443/tcp`              | Additional published ports                                                                                |
| `portainer_deploy`                   | `false`                 | Deploy or update Portainer                                                                                |
| `portainer_stack_name`               | `portainer`             | Docker stack name                                                                                         |

`group_vars/all.yml.example` sets `docker_engine_remove_conflicts: true`, so
copying the example enables removal of the listed conflicting packages on hosts
without Docker CE.

The root `.gitignore` excludes `inventory.yml`, `group_vars/all.yml`, join
tokens, and retry files.

## Validation

On the control node:

```bash
cd ansible
ansible-playbook --syntax-check -i inventory.example.yml node-setup.yml
ansible-lint node-setup.yml roles/
```

On a provisioned node, run the Docker commands with `sudo` or as a user in the
`docker` group so the Docker socket is accessible:

```bash
sudo docker version
sudo docker info
firewall-cmd --get-active-zones
firewall-cmd --zone=internal --list-ports
firewall-cmd --zone=public --list-ports
```

Cluster commands must run on a manager; they fail on workers:

```bash
sudo docker node ls
sudo docker node inspect self
sudo docker stack services portainer
```

`portainer` is the default stack name; replace it with your configured
`portainer_stack_name`. When `portainer_deploy` is false the role skips a new
deployment; an existing stack is not removed and may still be running.

To inspect a worker, run `docker node inspect <worker-name-or-id>` on a manager.

Runtime verification requires a real target host. The repository checks only
syntax and lint unless a host is available.
