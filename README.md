# ansible-playbooks

This project contains a collection of [Ansible playbooks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html) I use or have used in my homelab.

## Playbooks

### homelab

Provisioning and management of homelab resources

Pre-requisites:

- SSH keypair for your local client to server interactions

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

### k8s-cluster

Creating a Kubernetes cluster using Rocky Linux 8.

### k8s-proxmox

Creating a kubernetes cluster where one or more nodes are LXC containers in Proxmox

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