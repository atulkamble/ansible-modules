# Ansible Modules: Basic Practice (Amazon Linux lab)

This repository documents and practices the Ansible setup used on the control node at `/etc/ansible`. The example inventory separates the two EC2 managed hosts into `web` and `db` groups.

## Lab layout

Use this structure on the control node:

```text
/etc/ansible/
├── ansible.cfg
├── aws/
│   └── ansible.pem          # local private key; never commit or share
├── inventory/
│   └── hosts
├── playbooks/
│   ├── docker.yml
│   ├── httpd.yml
│   ├── jenkins.yml
│   ├── mysql.yml
│   ├── ping.yml
│   └── pkg.yml
└── roles/
```

The private key is intentionally excluded from this repository. Put your own key at `/etc/ansible/aws/ansible.pem` on the control node and restrict its permissions:

```bash
sudo chmod 400 /etc/ansible/aws/ansible.pem
sudo chown ec2-user:ec2-user /etc/ansible/aws/ansible.pem
```

Never paste a private key into GitHub, chat, or a README.

## 1. Ansible configuration

Save this as `/etc/ansible/ansible.cfg`:

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

Because these paths are relative, run Ansible commands from `/etc/ansible`. To ensure Ansible loads this config:

```bash
cd /etc/ansible
ansible --version
```

The output should show `/etc/ansible/ansible.cfg` as the config file.

## 2. Inventory

Save as `/etc/ansible/inventory/hosts`:

```ini
[all]
23.22.227.133
32.202.123.57

[web]
23.22.227.133

[db]
32.202.123.57
```

These are the addresses provided for this lab. Update them if the EC2 public IPs change. Both instances must allow inbound SSH from the control node. The SSH login is `ec2-user`.

## 3. Verify configuration and connectivity

Run from `/etc/ansible`:

```bash
ansible-inventory --graph
ansible all --list-hosts
ansible all -m ansible.builtin.ping
ansible web -m ansible.builtin.ping
ansible db -m ansible.builtin.ping
```

The Ansible `ping` module checks SSH connectivity and Python execution. It is not an ICMP ping. A successful result looks like:

```text
23.22.227.133 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

If you see `UNREACHABLE`, check the instance IP, security group rules, SSH user, key path, key permissions, and network route.

## 4. Basic modules with ad-hoc commands

```bash
# Run a command on every host
ansible all -m ansible.builtin.command -a 'uptime'

# Read OS facts
ansible all -m ansible.builtin.setup -a 'filter=ansible_distribution*'

# Check a file
ansible all -m ansible.builtin.stat -a 'path=/etc/hosts'

# Create a directory (become is enabled in ansible.cfg)
ansible web -m ansible.builtin.file -a 'path=/tmp/ansible-practice state=directory mode=0755'

# Write a file
ansible web -m ansible.builtin.copy -a 'content="Created by Ansible
" dest=/tmp/ansible-practice/hello.txt mode=0644'

# Display the file
ansible web -m ansible.builtin.command -a 'cat /tmp/ansible-practice/hello.txt'
```

Prefer purpose-built modules such as `file`, `copy`, and `dnf` over `shell`. Use `command` for commands that do not require shell parsing; use `shell` only for shell features such as pipes and redirects.

## 5. Run the lab playbooks

From `/etc/ansible`, run the playbooks in your `playbooks/` directory:

```bash
ansible-playbook playbooks/ping.yml
ansible-playbook playbooks/pkg.yml
ansible-playbook playbooks/httpd.yml
ansible-playbook playbooks/docker.yml
ansible-playbook playbooks/jenkins.yml
ansible-playbook playbooks/mysql.yml
```

Use a targeted host group when the playbook is written for one group:

```bash
ansible-playbook -i inventory/hosts playbooks/httpd.yml --limit web
ansible-playbook -i inventory/hosts playbooks/mysql.yml --limit db
```

The playbook's `hosts:` value selects the group it targets. Review each playbook before running it, especially package installation and service tasks. Package names and repository setup can vary by Amazon Linux release; verify the operating system with `ansible all -m setup -a 'filter=ansible_distribution*'`.

If sudo asks for a password, add `-K`:

```bash
ansible-playbook -K playbooks/pkg.yml
```

## 6. Module quick reference

| Module | Example use |
|---|---|
| `ansible.builtin.ping` | Validate Ansible connectivity |
| `ansible.builtin.command` | Run a command without shell parsing |
| `ansible.builtin.setup` | Gather managed-host facts |
| `ansible.builtin.debug` | Display a value or task result |
| `ansible.builtin.stat` | Inspect file metadata |
| `ansible.builtin.file` | Create files/directories and set permissions |
| `ansible.builtin.copy` | Copy a file or inline content |
| `ansible.builtin.dnf` | Install or remove packages on supported RHEL-family systems |
| `ansible.builtin.service` | Start, stop, or enable services |
| `ansible.builtin.user` | Create or manage a local user |
| `ansible.builtin.group` | Create or manage a local group |

## 7. Practice and idempotency

Try these tasks in a disposable lab:

1. Run the ping playbook against `all`, then separately against `web` and `db`.
2. Use the `file` module to create `/tmp/lab` with mode `0750`.
3. Use `copy` to place a message in `/tmp/lab/notes.txt`.
4. Use `stat` and `debug` to report whether that file exists.
5. Install `curl` with the correct package module for the host OS.
6. Run the same playbook twice. Explain why the second run should report `ok` for tasks already in the desired state.

Preview a playbook before applying changes:

```bash
ansible-playbook --check --diff playbooks/pkg.yml
```

Check mode simulates changes, and not every module can predict every outcome.

## Troubleshooting

- **Ansible uses the wrong config:** run commands from `/etc/ansible`; check `ansible --version`.
- **Permission denied for key:** run `chmod 400 /etc/ansible/aws/ansible.pem`.
- **SSH timeout:** verify EC2 public IP, security group inbound SSH rules, subnet route, and control-node egress.
- **Permission denied (publickey):** verify `remote_user = ec2-user` and that the private key matches the instance key pair.
- **Python interpreter error:** install Python on the managed host or set `ansible_python_interpreter` in inventory to its actual path.
- **Package not found:** confirm the managed host distribution/version and the package's repository availability.

## Security note

The sample inventory contains public IP addresses supplied for this lab. Replace them if they are no longer current. Keep private keys out of version control. If a key was ever committed or exposed, revoke/replace it through AWS and update the control node.
