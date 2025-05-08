# ansible-playbooks

This project contains a collection of [Ansible playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html) I use or have used in my homelab.

It's always a work in progress, so review the playbooks before applying!

## Playbooks

### homelab

Provisioning and management of homelab resources

Pre-requisites:

- SSH keypair for your local client to server interactions
- `ansible-galaxy collection install ansible.posix`
- `export USER_PASSWORD='your_secure_password'` for user creation
- `pip install passlib` to allow password hashing in users role

```sh
ssh-keygen -t ecdsa -b 521 -f ~/.ssh/id_ecdsa -C "your_email@example.com"
```

- SSH keypair for your GitHub interactions

```sh
ssh-keygen -t ecdsa -b 521 -f ~/.ssh/id_github -C "your_email@example.com"
```

Preparation:

- Populate hosts.yml file with hosts by group
- Populate hosts.yml file with desired variable values

Dry run:

```sh
ansible-playbook -i hosts.yml main.yml --check
```

Provision one host:

```sh
ansible-playbook -i hosts.yml main.yml --limit photos
```

Provision all hosts with:

```sh
ansible-playbook -i hosts.yml main.yml
```

Update all hosts:

```sh
ansible-playbook -i hosts.yml main.yml --tags update
```

#### Operation

- Update system
- Install packages
- Setup users (create + keys)
- Setup shell (spaceship + chezmoi)
- Install docker (when)
- Setup github_clone (when)

### k8s-cluster

Creates a Kubernetes cluster using Rocky Linux 8.

### k8s-proxmox

Creates a kubernetes cluster where one or more nodes are LXC containers in Proxmox

### samba-server

!! Not working yet !! Installs and configures samba server on destination host

## Troubleshooting

Test a configuration of a single host:

```sh
ansible -i hosts.yml frigate -m ping
```

Show what changes on one or more hosts:

```sh
ansible-playbook -i hosts.yml main.yml --check --diff --limit paperless,lancache,micropop
```

## License

Distributed under the MIT License. See LICENSE for more information.
