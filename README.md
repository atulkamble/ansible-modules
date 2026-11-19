# Ansible Modules: Basic Practice on Amazon Linux 2023

A small practice repository for learning Ansible's built-in modules against Amazon Linux 2023 EC2 instances. Examples use ad-hoc commands and one compact playbook; no application deployment playbooks are included.

## Lab layout

Run Ansible from the control node directory `/etc/ansible`:

```text
/etc/ansible/
├── ansible.cfg
├── aws/
│   └── ansible.pem          # local key; keep out of Git
├── inventory/
│   └── hosts
└── ansible-modules/
    ├── README.md
    └── practice.yml
```

Keep the private key on the control node only. Do not commit it to this repository.

```bash
chmod 400 /etc/ansible/aws/ansible.pem
```

## Configuration

Use this `/etc/ansible/ansible.cfg`:

```ini
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
```

Ansible resolves these relative paths from the current working directory. Run commands from `/etc/ansible` and confirm the active config:

```bash
cd /etc/ansible
ansible --version
```

## Inventory

Example `/etc/ansible/inventory/hosts`:

```ini
[all]
23.22.227.133
32.202.123.57

[web]
23.22.227.133

[db]
32.202.123.57

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

Replace these example IP addresses with the current EC2 public IPs. Ensure SSH is permitted from the control node and the instances use the `ec2-user` account.

## Check the lab

```bash
ansible-inventory --graph
ansible all --list-hosts
ansible all -m ansible.builtin.ping
ansible all -m ansible.builtin.setup -a 'filter=ansible_distribution*'
```

The Ansible `ping` module validates SSH and Python execution; it is not an ICMP ping.

## Basic module practice

Run these commands from `/etc/ansible`:

```bash
# Run a command without shell parsing
ansible all -m ansible.builtin.command -a 'uptime'

# Read the operating system facts
ansible all -m ansible.builtin.setup -a 'filter=ansible_distribution*'

# Inspect a file
ansible all -m ansible.builtin.stat -a 'path=/etc/hosts'

# Create a directory on the web host
ansible web -m ansible.builtin.file -a 'path=/tmp/ansible-practice state=directory mode=0750'

# Create a file with inline content
ansible web -m ansible.builtin.copy -a 'content=Ansible module practice dest=/tmp/ansible-practice/notes.txt mode=0640'

# Check the file contents
ansible web -m ansible.builtin.command -a 'cat /tmp/ansible-practice/notes.txt'

# Install curl using Amazon Linux 2023's DNF package manager
ansible web -m ansible.builtin.dnf -a 'name=curl state=present'

# Check a service with the service_facts module
ansible web -m ansible.builtin.service_facts
```

The configuration enables privilege escalation, so package management tasks run with sudo. Avoid putting secrets in command arguments or playbook output.

## Run the practice playbook

The `practice.yml` file demonstrates `file`, `copy`, `stat`, `command`, `register`, and `debug` modules on the web group:

```bash
ansible-playbook ansible-modules/practice.yml
```

Run it a second time. Tasks that already match the desired state should report `ok`, not `changed`. This is Ansible's idempotent behavior.

Preview the changes first:

```bash
ansible-playbook --check --diff ansible-modules/practice.yml
```

Check mode simulates changes; some modules cannot predict all results.

## Modules in this practice

| Module | What it practices |
|---|---|
| `ansible.builtin.ping` | Test Ansible connectivity |
| `ansible.builtin.command` | Run a command without shell parsing |
| `ansible.builtin.setup` | Gather system facts |
| `ansible.builtin.file` | Create directories and set permissions |
| `ansible.builtin.copy` | Create a file from inline content |
| `ansible.builtin.stat` | Inspect file metadata and existence |
| `ansible.builtin.dnf` | Manage packages on Amazon Linux 2023 |
| `ansible.builtin.service_facts` | Gather installed service information |
| `ansible.builtin.debug` | Display values and registered results |

Use purpose-built modules such as `file`, `copy`, and `dnf` when available. Prefer `command` over `shell` unless shell parsing is needed.

## Exercises

1. Target only `db` with the ping and setup modules.
2. Create `/tmp/lab` with mode `0750` on both hosts.
3. Copy a file to `/tmp/lab/notes.txt`, then use `stat` to verify it exists.
4. Install `git` with `ansible.builtin.dnf`.
5. Register the result of `uptime` in a playbook and display its `stdout`.
6. Run the same playbook twice and identify which tasks changed.

## Troubleshooting

- **Wrong Ansible config:** run commands from `/etc/ansible`; check `ansible --version`.
- **SSH permission denied:** verify the current EC2 IP, `ec2-user`, key pair, and key permissions.
- **SSH timeout:** check inbound SSH rules and network routing.
- **Python interpreter error:** confirm Python 3 is installed and set `ansible_python_interpreter=/usr/bin/python3`.
- **Package unavailable:** verify the host is Amazon Linux 2023 and inspect enabled repositories with `dnf repolist`.

## Security

Never add `ansible.pem` to Git. Avoid using `host_key_checking = False` outside a temporary lab; production environments should verify SSH host keys. Restrict SSH access in the EC2 security group to trusted source addresses.
