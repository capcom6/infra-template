# Ansible node setup

Provisions a new Linux node for the Docker Swarm infrastructure in this
repository:

- installs Docker CE on a supported RHEL-family distribution
- initializes a new Swarm or joins an existing one
- applies Swarm node labels
- restricts Swarm ports to a configured private interface and opens published
  ports separately in `firewalld` when it is active
- deploys or updates the existing [compose.portainer.yml](../compose.portainer.yml)

The playbook does not deploy the infra or monitoring stacks, and it does not
render `.env` or create Docker secrets. It is intended for new nodes: if you
re-run it on an existing node and the rendered `daemon.json` differs, Docker is
restarted.

## Prerequisites

- Ansible with Python 3 on the control node.
- SSH access to the target nodes.
- AlmaLinux/RHEL/Rocky/CentOS 8, 9, or 10. Docker CE repository support must be
  verified before extending `docker_engine_supported_major_versions`.
- A private-network interface shared by the Swarm nodes. Set
  `firewall_swarm_interface` to that interface on each node. If `firewalld` is
  active, the playbook assigns it to the `internal` zone and opens
  `2377/tcp`, `7946/tcp`, `7946/udp`, and `4789/udp` there. Portainer's
  `9443/tcp` is opened separately in the `public` zone. Add `80/tcp` and
  `443/tcp` to `firewall_extra_ports` when Traefik runs on the node.

Docker manages iptables rules for published container ports, but it does not
open `firewalld` for Swarm control and overlay traffic. If `firewalld` is active,
set `firewall_swarm_interface` to the private interface or open Swarm ports only
on the private network outside Ansible. The interface binding moves that
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

`swarm_manager_addr` belongs in `group_vars/all.yml`; use the manager host
without `:2377`.

## Bootstrap the first manager

```bash
cd ansible
ansible-playbook node-setup.yml --limit node1
```

This installs Docker, initializes Swarm, applies the manager labels, and
deploys Portainer. After it finishes, put a worker token into
`secrets/swarm.yml`.

## Add workers or managers

```bash
cd ansible
ansible-playbook node-setup.yml \
  -e @secrets/swarm.yml \
  --limit node2
```

For a worker node with labels, `swarm_label_delegate` must name an existing
manager that is present in the inventory. Label updates are delegated there
because `docker node update` only works on managers.

If the Portainer play is limited out, Portainer is not touched. Run the playbook
without `--limit`, or include the manager, to deploy or update Portainer.

## Configuration reference

| Variable                          | Default                 | Purpose                                           |
| --------------------------------- | ----------------------- | ------------------------------------------------- |
| `swarm_role`                      | none                    | `init`, `join_manager`, or `join_worker`          |
| `swarm_manager_addr`              | none                    | Existing manager host for joins                   |
| `swarm_join_token`                | none                    | Manager or worker join token, supplied externally |
| `swarm_advertise_addr`            | primary IPv4            | Address advertised to the cluster                 |
| `swarm_node_labels`               | `{}`                    | Labels applied to the node                        |
| `swarm_label_delegate`            | none                    | Existing manager used to label workers            |
| `docker_engine_remove_conflicts`  | `false`                 | Remove Docker-conflicting packages on fresh hosts |
| `docker_engine_conflict_packages` | podman/buildah/runc set | Explicit package list for conflict removal        |
| `docker_engine_daemon_options`    | log rotation only       | Docker daemon JSON options                        |
| `firewall_manage`                 | `true`                  | Configure active `firewalld`                      |
| `firewall_swarm_interface`        | empty                   | Private interface assigned to the `internal` zone |
| `firewall_swarm_zone`             | `internal`               | Zone for Swarm-only ports                         |
| `firewall_public_zone`            | `public`                 | Zone for published ports                          |
| `firewall_extra_ports`            | `9443/tcp`              | Additional published ports                        |
| `portainer_deploy`                | `true`                  | Deploy or update Portainer                        |
| `portainer_stack_name`            | `portainer`             | Docker stack name                                 |

The root `.gitignore` excludes `inventory.yml`, `group_vars/all.yml`, join
tokens, and retry files.

## Validation

On the control node:

```bash
cd ansible
ansible-playbook --syntax-check -i inventory.example.yml node-setup.yml
ansible-lint node-setup.yml roles/
```

On a provisioned node:

```bash
docker version
docker info
docker node ls
docker node inspect self
firewall-cmd --get-active-zones
firewall-cmd --zone=internal --list-ports
firewall-cmd --zone=public --list-ports
docker stack services portainer
```

Runtime verification requires a real target host. The repository checks only
syntax and lint unless a host is available.
