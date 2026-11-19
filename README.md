# Ansible Modules: Basic Practice on Amazon Linux 2023

A focused practice repository for Ansible built-in modules on Amazon Linux 2023 EC2 hosts. It contains an inventory, Ansible configuration, and one short practice playbook. It does not include application setup or deployment playbooks.

## Repository layout

Use this repository as the contents of /etc/ansible on the control node:

~~~text
/etc/ansible/
├── ansible.cfg
├── aws/
│   └── ansible.pem          # local key; never commit
├── inventory/
│   └── hosts
├── practice.yml
└── README.md
~~~

The private key is not included. Place your own key at /etc/ansible/aws/ansible.pem on the control node and restrict its permissions:

~~~bash
chmod 400 /etc/ansible/aws/ansible.pem
~~~

Never commit or share the private key.

## Configuration

The included ansible.cfg expects paths relative to the current working directory:

~~~ini
[defaults]
inventory = ./inventory/hosts
remote_user = ec2-user
private_key_file = ./aws/ansible.pem
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
~~~

Run commands from /etc/ansible so Ansible loads this configuration:

~~~bash
cd /etc/ansible
ansible --version
~~~

Confirm the output shows /etc/ansible/ansible.cfg as the active configuration file.

## Inventory

The included inventory/hosts defines two groups:

~~~ini
[all]
23.22.227.133
32.202.123.57

[web]
23.22.227.133

[db]
32.202.123.57

[all:vars]
ansible_python_interpreter=/usr/bin/python3
~~~

Replace the example public IP addresses with the current EC2 addresses. Allow inbound SSH from the control node and use the ec2-user account. Amazon Linux 2023 uses DNF for package management.

## Check connectivity and gather facts

Run from /etc/ansible:

~~~bash
ansible-inventory --graph
ansible all --list-hosts
ansible all -m ansible.builtin.ping
ansible all -m ansible.builtin.setup -a 'filter=ansible_distribution*'
~~~

The Ansible ping module checks SSH connectivity and Python execution. It is not an ICMP network ping.

## Practice modules with ad-hoc commands

~~~bash
# Run a command without shell parsing
ansible all -m ansible.builtin.command -a 'uptime'

# Inspect a file
ansible all -m ansible.builtin.stat -a 'path=/etc/hosts'

# Create a directory on the web host
ansible web -m ansible.builtin.file -a 'path=/tmp/ansible-practice state=directory mode=0750'

# Create a file with inline content
ansible web -m ansible.builtin.copy -a 'content="Ansible module practice" dest=/tmp/ansible-practice/notes.txt mode=0640'

# Read the file
ansible web -m ansible.builtin.command -a 'cat /tmp/ansible-practice/notes.txt'

# Install curl with the Amazon Linux 2023 package manager
ansible web -m ansible.builtin.dnf -a 'name=curl state=present'

# Gather service facts
ansible web -m ansible.builtin.service_facts

# Display a value
ansible web -m ansible.builtin.debug -a 'msg="Basic module practice complete"'
~~~

Privilege escalation is enabled in the config for package installation and other privileged operations. Prefer purpose-built modules such as file, copy, and dnf; use command for simple commands. Use shell only when shell features such as pipes or redirects are required.

## Practice playbook

The root-level practice.yml demonstrates file, copy, stat, command, register, and debug on the web group:

~~~bash
ansible-playbook practice.yml
~~~

Run it twice. Tasks that already match the requested state should report ok on the second run. This is called idempotency.

Preview possible changes:

~~~bash
ansible-playbook --check --diff practice.yml
~~~

Check mode is a simulation and some modules cannot predict every result.

## Modules covered

| Module | Practice |
|---|---|
| ansible.builtin.ping | Test Ansible connectivity |
| ansible.builtin.command | Run a command without shell parsing |
| ansible.builtin.setup | Gather system facts |
| ansible.builtin.file | Create directories and set permissions |
| ansible.builtin.copy | Create a file from inline content |
| ansible.builtin.stat | Inspect file metadata and existence |
| ansible.builtin.dnf | Manage packages on Amazon Linux 2023 |
| ansible.builtin.service_facts | Gather installed service information |
| ansible.builtin.debug | Display values and registered results |

## Exercises

1. Run ping and setup against only the db group.
2. Create /tmp/lab with mode 0750 on both hosts.
3. Copy a file to /tmp/lab/notes.txt and use stat to check that it exists.
4. Install git with ansible.builtin.dnf.
5. Add a task that registers uptime output and displays its stdout.
6. Run your playbook twice and identify which tasks changed.

## Troubleshooting

- Wrong Ansible config: run commands from /etc/ansible and check ansible --version.
- SSH permission denied: verify the EC2 IP, ec2-user, key pair, and key permissions.
- SSH timeout: check inbound SSH rules and network routing.
- Python interpreter error: verify Python 3 is installed and inventory points to /usr/bin/python3.
- Package unavailable: confirm the host is Amazon Linux 2023 and inspect enabled repositories with dnf repolist.

## Security

Never commit ansible.pem. The lab config disables SSH host-key checking for convenience; enable host-key verification in production. Restrict SSH access in EC2 security groups to trusted source addresses.
