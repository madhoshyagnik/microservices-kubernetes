### This ansible role is in case this role is ever pushed to ansible-galaxy

K3s Ansible Role
================

This role automates the installation and configuration of a K3s (Lightweight Kubernetes) cluster. It is structured to handle bootstrap, control plane (server) setup, worker node (agent) setup, and cluster-wide configuration (such as taints, kubeconfig retrieval, etc.).

Requirements
------------

This role requires a Debian-based target system (e.g., Debian, Ubuntu) and passwordless SSH configuration between the Ansible controller and the nodes.

Role Variables
--------------

The following variables can be configured:

| Variable | Default Value | Description |
|----------|---------------|-------------|
| `packages` | `[curl, wget, git, vim, htop, jq]` | The list of basic utility packages to install during bootstrapping. |
| `k3s_action` | `all` | Specific task file to run (`bootstrap`, `server`, `agent`, `configure`). If not set, it defaults to executing tasks based on host groups. |
| `k3s_server_ip` | *(Required)* | The IP address of the K3s control plane (server) node. |
| `k3s_token` | *(Required)* | The shared secret token used to join nodes to the cluster. |

Dependencies
------------

None.

Example Playbook
----------------

```yaml
- name: Bootstrap K3s Cluster
  hosts: k3s
  become: true
  gather_facts: true
  roles:
    - role: k3s
      vars:
        k3s_action: bootstrap

- name: Install K3s Server
  hosts: control
  become: true
  gather_facts: true
  roles:
    - role: k3s
      vars:
        k3s_action: server

- name: Install K3s Workers
  hosts: workers
  become: true
  gather_facts: true
  roles:
    - role: k3s
      vars:
        k3s_action: agent

- name: Configure Cluster
  hosts: control
  become: true
  gather_facts: true
  roles:
    - role: k3s
      vars:
        k3s_action: configure
```

License
-------

MIT

Author Information
------------------

Created for provisioning local multi-node development K3s clusters.

