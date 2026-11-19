# Ansible Modules: Basic Practice

A beginner-friendly, hands-on lab for learning common Ansible modules. It includes ad-hoc commands, runnable playbooks, exercises, and expected outcomes.

## Learning objectives

By completing this practice, you will be able to:

- Explain what Ansible modules do.
- Test connectivity to managed hosts.
- Manage files, packages, services, users, and groups.
- Gather and display task results.
- Write idempotent playbooks and check what would change.

## Lab requirements

- Ansible installed on a Linux, macOS, or WSL control node.
- One Linux managed host reachable over SSH.
- A working SSH key and permission to use `sudo` or `become` for privileged tasks.
- Python available on the managed host (normally present on Linux hosts).

> Use a disposable lab VM. The exercises create a user, install Nginx and curl, and create files on the managed host.

## Project files

```text
.
├── inventory.ini
├── ansible.cfg
├── playbooks/
│   ├── 01-files.yml
│   ├── 02-packages-services.yml
│   ├── 03-users.yml
│   └── 04-facts-and-results.yml
└── README.md
```

## 1. Configure the inventory

Edit `inventory.ini`. Replace the sample address, login name, and private key path with values for your VM.

```ini
[web]
web1 ansible_host=192.0.2.10 ansible_user=ec2-user ansible_ssh_private_key_file=~/.ssh/lab-key.pem

[web:vars]
ansible_python_interpreter=auto_silent
```

For Ubuntu, the SSH user is often `ubuntu`; for Amazon Linux, it is often `ec2-user`. Ensure the key file is readable only by its owner:

```bash
chmod 400 ~/.ssh/lab-key.pem
```

This repository includes an `ansible.cfg` that points Ansible to this inventory.

## 2. Check connectivity

```bash
ansible web -m ansible.builtin.ping
ansible web -m ansible.builtin.setup -a 'filter=ansible_distribution*'
```

The Ansible `ping` module checks that Ansible can connect and execute Python on the managed host. It is not an ICMP network ping.

## 3. Try ad-hoc module commands

Run each from the repository root:

```bash
# Run a simple command
ansible web -m ansible.builtin.command -a 'uptime'

# Inspect a path
ansible web -m ansible.builtin.stat -a 'path=/etc/hosts'

# Create a directory
ansible web -b -m ansible.builtin.file -a 'path=/tmp/ansible-practice state=directory mode=0755'

# Write a file
ansible web -b -m ansible.builtin.copy -a 'content="Created by Ansible\n" dest=/tmp/ansible-practice/hello.txt mode=0644'

# Read the file
ansible web -m ansible.builtin.command -a 'cat /tmp/ansible-practice/hello.txt'
```

Use `-b` (become) when the operation needs elevated privileges. Avoid using the `shell` module unless you need shell features such as pipes or redirection; prefer a purpose-built module or `command`.

## 4. Run the playbooks

```bash
ansible-playbook playbooks/01-files.yml
ansible-playbook playbooks/02-packages-services.yml
ansible-playbook playbooks/03-users.yml
ansible-playbook playbooks/04-facts-and-results.yml
```

If your sudo setup requires a password, add `-K`:

```bash
ansible-playbook -K playbooks/02-packages-services.yml
```

### Playbook 1: files and permissions

Creates a directory, writes a file, checks its metadata, and displays the result.

### Playbook 2: packages and services

Installs Nginx and curl using the package manager for the managed host, then enables and starts Nginx. It supports Debian-family and RHEL-family Linux distributions.

### Playbook 3: users and groups

Creates a `student` group and a `student` account. This is a lab account; choose a different name if it already exists.

### Playbook 4: facts and results

Shows gathered operating system facts, registers command output, and prints it with `debug`.

## 5. Check idempotency

Run a playbook once, then run it again:

```bash
ansible-playbook playbooks/01-files.yml
ansible-playbook playbooks/01-files.yml
```

On the second run, tasks should generally report `ok` instead of `changed` when the desired state is already present. This behavior is called **idempotency**.

Preview changes without applying them:

```bash
ansible-playbook --check --diff playbooks/01-files.yml
```

Check mode is a simulation; not every module can fully predict changes.

## 6. Practice exercises

Try these without copying an existing task:

1. Create `/tmp/lab` with mode `0750`.
2. Copy a file named `notes.txt` into that directory.
3. Install `git`.
4. Create a user named `trainee` with a home directory.
5. Use `stat` to verify the file and display whether it exists.
6. Ensure the Nginx service is stopped, then change the playbook to start it again.
7. Run the same playbook twice and explain the second run's output.

## Modules covered

| Module | Use |
|---|---|
| `ansible.builtin.ping` | Test Ansible connectivity |
| `ansible.builtin.command` | Run a command without shell parsing |
| `ansible.builtin.debug` | Display values and task results |
| `ansible.builtin.stat` | Inspect file or directory metadata |
| `ansible.builtin.file` | Manage files, directories, and permissions |
| `ansible.builtin.copy` | Copy content or files to managed hosts |
| `ansible.builtin.apt` | Manage packages on Debian-family systems |
| `ansible.builtin.dnf` | Manage packages on RHEL-family systems |
| `ansible.builtin.service` | Manage services |
| `ansible.builtin.group` | Manage local groups |
| `ansible.builtin.user` | Manage local user accounts |
| `ansible.builtin.setup` | Gather system facts |

## Troubleshooting

- **UNREACHABLE / Permission denied:** check the host address, SSH username, security group or firewall, and private key.
- **Bad permissions on private key:** run `chmod 400 ~/.ssh/lab-key.pem`.
- **Missing sudo privileges:** verify the remote account can run sudo; use `-K` if sudo prompts for a password.
- **No package match:** confirm the managed host's distribution and use the matching package manager.
- **Python interpreter error:** install Python on the managed host or set `ansible_python_interpreter` to its actual path.

## Cleanup

Remove the practice files and lab account if you no longer need them:

```bash
ansible web -b -m ansible.builtin.file -a 'path=/opt/ansible-modules-practice state=absent'
ansible web -b -m ansible.builtin.user -a 'name=student state=absent remove=true'
ansible web -b -m ansible.builtin.group -a 'name=student state=absent'
```

The package and service playbook installs Nginx and curl and enables Nginx. Remove packages or disable the service separately if you want to revert those changes.
