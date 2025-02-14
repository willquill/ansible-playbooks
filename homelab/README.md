# Homelab Ansible Playbooks

This repository contains Ansible playbooks for setting up and managing a homelab environment. The playbooks automate the installation and configuration of various services and applications commonly used in a homelab setup.

## Prerequisites

- Ansible installed on the control machine
- SSH access to the target machines
- Properly configured inventory file

## Usage

1. Clone the repository:

```sh
git clone https://github.com/yourusername/homelab.git
cd homelab
```

1. Update the inventory file with your target machines:

```ini
[homelab]
server1 ansible_host=192.168.1.10
server2 ansible_host=192.168.1.11
```

1. Run the playbooks:

```sh
ansible-playbook -i inventory site.yml
```

## Playbooks

- `site.yml`: Main playbook that includes all other playbooks.
- `setup.yml`: Playbook for initial setup and configuration.
- `services.yml`: Playbook for installing and configuring services.

## Contributing

Feel free to submit issues and pull requests. Contributions are welcome!

## License

This project is licensed under the MIT License.
