```
sudo ansible all -m ansible.builtin.ping
sudo ansible all -m ansible.builtin.setup -a 'filter=ansible_distribution*'

sudo nano  inventory/hosts 
[web]
23.22.227.133

[db]
32.202.123.57

sudo ansible web -m ansible.builtin.setup -a 'filter=ansible_distribution*'
sudo ansible db -m ansible.builtin.setup -a 'filter=ansible_distribution*'

sudo ansible all -m ansible.builtin.command -a 'uptime'
sudo ansible all -m ansible.builtin.stat -a 'path=/etc/hosts'

sudo ansible web -m ansible.builtin.file -a 'path=/tmp/ansible-practice state=directory mode=0750'

sudo ansible web -m ansible.builtin.copy -a 'content="Ansible module practice" dest=/tmp/ansible-practice/notes.txt mode=0640'

sudo ansible web -m ansible.builtin.dnf -a 'name=tree state=present'

sudo ansible web -m ansible.builtin.service_facts

sudo ansible web -m ansible.builtin.service_facts

// module as ansible playbook 

sudo touch /etc/ansible/playbooks/practice.yml

sudo nano /etc/ansible/playbooks/practice.yml



---
- name: Practice basic Ansible modules on Amazon Linux 2023
  hosts: web
  become: true
  tasks:
    - name: Create a practice directory
      ansible.builtin.file:
        path: /tmp/ansible-practice
        state: directory
        mode: "0750"

    - name: Write a practice notes file
      ansible.builtin.copy:
        content: |
          Ansible module practice on Amazon Linux 2023.
        dest: /tmp/ansible-practice/notes.txt
        mode: "0640"

    - name: Inspect the notes file
      ansible.builtin.stat:
        path: /tmp/ansible-practice/notes.txt
      register: notes_file

    - name: Show whether the notes file exists
      ansible.builtin.debug:
        msg: "File exists: {{ notes_file.stat.exists }}"

    - name: Read the notes file
      ansible.builtin.command: cat /tmp/ansible-practice/notes.txt
      register: notes_content
      changed_when: false

    - name: Display the notes content
      ansible.builtin.debug:
        var: notes_content.stdout


cd /etc/ansible 

ansible-playbook -i inventiry/hosts /playbooks/practice.yml


```
