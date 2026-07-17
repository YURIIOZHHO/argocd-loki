## Instructions for Using the Logrotate Ansible Role.

**The Logrotate Ansible role has the following structure:**
```
└── logrotate
    ├── inventory.ini
    ├── playbook.yaml
    ├── README.md
    └── roles
        └── logrotate-config
            ├── tasks
            │   └── main.yaml
            └── templates
                └── custom-logrotate.j2
```

**1. The `inventory.ini` file contains the information required for Ansible to connect to the target servers, including the host IP address, SSH user, and the path to the private SSH key.**
```
[servers]
#private ip(for ex: 1.1.1.1) ansible_user=for example: ec2-user ansible_ssh_private_key_file=path/to/key
1.1.1.1 ansible_user=ec2-user ansible_ssh_private_key_file=/home/ec2-user/key.pem
```

**2. The `playbook.yaml` file defines which hosts the role will be executed on and applies the `logrotate-config` role to those hosts.**
```
---
- name: Configure logrotate
  hosts: servers # Host group defined in inventory.ini
  become: true

  roles:
    - logrotate-config # Role's name
```
**In this case, the playbook runs the `logrotate-config` role on all hosts that belong to the servers group defined in the inventory file**

**3. The `roles` directory contains the `logrotation-config` role. This role consists of two main directories:**
**`tasks/` – contains the tasks responsible for installing and configuring logrotate.**
```
---
- name: Update packages
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600
  become: true
  when: ansible_os_family == "Debian" # Update the package cache on Debian-based systems

- name: Install logrotate
  ansible.builtin.apt:
    name: logrotate
    state: present
  become: true
  when: ansible_os_family == "Debian" # Install logrotate on Debian-based systems

- name: Update package cache
  ansible.builtin.dnf:
    update_cache: true
  become: true
  when: ansible_os_family == "RedHat" # Update the package cache on Red Hat-based systems

- name: Install logrotate
  ansible.builtin.dnf:
    name: logrotate
    state: present
  become: true
  when: ansible_os_family == "RedHat" # Update the package cache on Red Hat-based systems

- name: Find default logrotate configs
  ansible.builtin.find:
    paths: /etc/logrotate.d
    file_type: file
  register: logrotate_configs # Find all existing logrotate configuration files

- name: Remove default logrotate configs
  ansible.builtin.file:
    path: "{{ item.path }}"
    state: absent
  loop: "{{ logrotate_configs.files }}"
  become: true # Remove the default configuration files

- name: Install logrotate configuration
  ansible.builtin.template:
    src: custom-logrotate.j2
    dest: /etc/logrotate.d/custom
    owner: root
    group: root
    mode: "0644"
  become: true # Deploy the custom logrotate configuration

- name: Print done
  debug:
    msg: "Finished"
```

**The `templates` directory contains the `custom-logrotate.j2` template, which defines the log rotation policy.**

```jinja2
/var/log/*.log
/var/log/*/*.log
/var/log/*/*/*.log   # Includes log files in subdirectories (tested up)

{
    size 500M         # Rotate a log file when it exceeds 500 MB
    rotate 7          # Keep the last 7 rotated log files
    compress          # Compress rotated logs using gzip
    missingok         # Ignore missing log files without reporting an error
    notifempty        # Do not rotate empty log files
    copytruncate      # Copy the log file and truncate the original without restarting the service
{% if ansible_os_family == "Debian" %}
    su root syslog    # Required on Debian/Ubuntu because /var/log is owned by root:syslog
{% endif %}
}
```

**From the ```logrotate``` directory, run the following command to execute the Ansible playbook: ```ansible-playbook -i inventory.ini playbook.yaml```**
