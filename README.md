# Hello-Ansiblea
<h1 align="center">Ansible - Practical Automation Guide</h1>

<p align="center">Essential commands, inventory files, YAML playbooks, server configuration, automation, and troubleshooting.</p>

---

A practical reference for controlling Linux servers with **Ansible**. Every command uses a real syntax example. The playbooks are complete; use the provided inventory and replace the example host addresses before connecting.

> **Lab assumptions:** Ansible runs on an Ubuntu/Linux control node (or WSL). Managed Linux hosts have SSH access, Python, and an authorized account. The addresses `192.0.2.11` to `192.0.2.14` are **documentation-only examples** and are not reachable servers. Replace them with actual lab addresses. Commands using `become: true` or `-b` require sudo rights.

## Table of Contents

1. [Ansible Basics](#1-ansible-basics)
2. [Installation and SSH Setup](#2-installation-and-ssh-setup)
3. [Inventory Management](#3-inventory-management)
4. [Essential Ansible Commands](#4-essential-ansible-commands)
5. [YAML and Playbook Structure](#5-yaml-and-playbook-structure)
6. [Essential Ansible Modules](#6-essential-ansible-modules)
7. [Running and Checking Playbooks](#7-running-and-checking-playbooks)
8. [Variables, Facts, Loops and Handlers](#8-variables-facts-loops-and-handlers)
9. [Secrets and Security](#9-secrets-and-security)
10. [Practical Automation Projects](#10-practical-automation-projects)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Ansible Basics

Ansible automates server tasks using SSH and YAML playbooks. On normal Linux hosts, you **do not install an Ansible agent**; the control node connects remotely.

| Term | Meaning | Practical example |
|---|---|---|
| Control node | Computer running Ansible | Your Ubuntu lab machine |
| Managed node | Server controlled by Ansible | `node01` and `node02` |
| Inventory | File listing hosts and groups | `inventory.ini` |
| Host alias | Friendly name pointing to a target | `db1 ansible_host=192.0.2.13` |
| Group | Set of targets | `[web_nodes]` |
| Module | Tool that performs a task | `ansible.builtin.apt` installs packages |
| Ad hoc command | Single Ansible command | Check uptime on all web nodes |
| Playbook | YAML file defining repeatable tasks | `nginx.yml` |
| Play | Selects hosts and lists tasks | `hosts: web_nodes` |
| Task | One operation in a play | Ensure Nginx is installed |
| Handler | Task that runs when notified of a change | Reload Nginx after a config update |
| Idempotence | Re-running the same desired configuration should avoid unnecessary changes | `state: present` when Nginx is already installed |

**Typical flow:** inventory -> test connectivity -> write playbook -> syntax check -> run -> verify results.

## 2. Installation and SSH Setup

Install Ansible on a **Linux control node**. On Windows, use a Linux environment such as **WSL**; Windows is not a native Ansible control node. Managed Linux servers do not require Ansible itself, but normally need Python and working SSH access.

| Task | Complete command | Explanation |
|---|---|---|
| Install on Ubuntu | `sudo apt update && sudo apt install -y ansible` | Install Ansible from Ubuntu's package repository. |
| Alternative pipx installation | `pipx install --include-deps ansible` | Install the full Ansible package if `pipx` is already installed. Choose one installation approach. |
| Check version | `ansible --version` | Show installed version and configuration paths. |
| Check playbook CLI | `ansible-playbook --version` | Confirm playbook runner is available. |
| Check SSH | `ssh -V` | Display OpenSSH client version. |
| Generate SSH key | `ssh-keygen -t ed25519 -C 'ansible-lab'` | Create a key pair if you do not already have one; never overwrite an existing key accidentally. |
| Copy public key to first host | `ssh-copy-id student@192.0.2.11` | Add your public key to the authorized account on `node01`. Requires initial authorized login. |
| Test SSH access | `ssh student@192.0.2.11` | Verify authentication before trying Ansible. Type `exit` to leave. |
| Test local Ansible | `ansible localhost -i 'localhost,' -c local -m ansible.builtin.ping` | Runs the connection test locally; no remote server required. |

The `ansible.builtin.ping` module checks Ansible connectivity and Python; it is **not** an ICMP network ping. For remote SSH connections, ensure the actual host addresses and SSH users are correct before continuing.

## 3. Inventory Management

An inventory specifies which servers Ansible should manage. This example reflects our practice with **web nodes, database aliases, a Boston group, and a group of groups**.

**File: `inventory.ini`**

```ini
[web_nodes]
node01 ansible_host=192.0.2.11
node02 ansible_host=192.0.2.12

[db_nodes]
db1 ansible_host=192.0.2.13

[boston_nodes]
server4 ansible_host=192.0.2.14

[all_servers:children]
web_nodes
db_nodes
boston_nodes

[all_servers:vars]
ansible_user=student
```

**Replace the addresses and `student` user with the real ones in your lab.** Use a resolvable hostname or IP after `ansible_host=`. Do not put SSH passwords in this file.

| Inventory entry | Exact meaning |
|---|---|
| `[web_nodes]` | Defines a group containing `node01` and `node02`. |
| `node01 ansible_host=192.0.2.11` | Uses `node01` as the inventory name and the provided IP for connections. |
| `db1 ansible_host=192.0.2.13` | Aliases an actual database host as `db1`. |
| `[boston_nodes]` | Holds the `server4` practice host. |
| `[all_servers:children]` | Makes `web_nodes`, `db_nodes`, and `boston_nodes` child groups of `all_servers`. |
| `[all_servers:vars]` | Shares `ansible_user=student` with hosts in those child groups. |

### Inventory commands

| Task | Complete command | Explanation |
|---|---|---|
| Show inventory tree | `ansible-inventory -i inventory.ini --graph` | Displays group membership and nesting. |
| Show parsed inventory | `ansible-inventory -i inventory.ini --list` | Displays inventory in structured JSON. |
| Show host variables | `ansible-inventory -i inventory.ini --host node01` | Prints variables resolved for `node01`. |
| List all matching hosts | `ansible all_servers -i inventory.ini --list-hosts` | Displays hosts in the parent group. |
| List web nodes | `ansible web_nodes -i inventory.ini --list-hosts` | Displays the two web inventory hosts. |
| Test one host | `ansible node01 -i inventory.ini -m ansible.builtin.ping` | Connects only to `node01`. |
| Test every Linux host | `ansible all_servers -i inventory.ini -m ansible.builtin.ping` | Checks all listed Linux targets. |

### Optional: Windows inventory with WinRM

Windows is managed differently from Linux. For Windows practice, create a **separate** inventory, not a Windows entry in the Linux `all_servers` group.

**File: `windows.ini`**

```ini
[windows]
win01 ansible_host=win01.example.internal

[windows:vars]
ansible_user=Administrator
ansible_connection=winrm
ansible_port=5986
ansible_winrm_scheme=https
ansible_winrm_transport=ntlm
ansible_winrm_server_cert_validation=validate
```

Requirements: working WinRM HTTPS listener on Windows, a certificate trusted by the control node and matching the hostname, a reachable host, and authorized credentials. On a control node with Ansible installed through `pipx`, the following installs the WinRM Python dependency:

```bash
pipx inject ansible 'pywinrm>=0.4.0'
ansible-galaxy collection install ansible.windows
ansible windows -i windows.ini -m ansible.windows.win_ping --ask-pass
```

If Ansible was installed with system packages, install `pywinrm` into the Python environment that actually runs Ansible instead of using `pipx inject`. **Do not** disable certificate checking or store cleartext Windows passwords in inventory. `win_ping` is a Windows management check, not ICMP ping.

## 4. Essential Ansible Commands

Commands below assume the Linux `inventory.ini` from Section 3 and hosts reachable over SSH.

| Task | Complete command | Explanation |
|---|---|---|
| Verify all web nodes | `ansible web_nodes -i inventory.ini -m ansible.builtin.ping` | `-i` selects inventory; `-m` selects the module. |
| Check uptime | `ansible web_nodes -i inventory.ini -m ansible.builtin.command -a 'uptime'` | `-a` passes arguments to the `command` module. |
| Check hostname | `ansible node01 -i inventory.ini -m ansible.builtin.command -a 'hostname'` | Execute one command on `node01`. |
| Run a shell pipeline | `ansible web_nodes -i inventory.ini -m ansible.builtin.shell -a 'df -h / \| tail -n 1'` | `shell` interprets pipes and redirection; `command` does not. |
| Gather facts | `ansible node01 -i inventory.ini -m ansible.builtin.setup` | Collect OS, network and other machine facts. |
| Filter facts | `ansible node01 -i inventory.ini -m ansible.builtin.setup -a 'filter=ansible_distribution*'` | Restricts fact output to matching names. |
| Execute with sudo | `ansible web_nodes -i inventory.ini -b -m ansible.builtin.command -a 'whoami'` | `-b` enables privilege escalation; typically prints `root`. |
| Prompt for sudo password | `ansible web_nodes -i inventory.ini -b -K -m ansible.builtin.command -a 'whoami'` | `-K` prompts for a become password if needed. |
| Use another SSH user | `ansible node01 -i inventory.ini -u admin -m ansible.builtin.ping` | `-u admin` overrides the inventory's SSH username. |
| Specify SSH key | `ansible node01 -i inventory.ini --private-key ~/.ssh/id_ed25519 -m ansible.builtin.ping` | Uses the specified private key file. |
| Run at most two hosts at once | `ansible web_nodes -i inventory.ini -f 2 -m ansible.builtin.command -a 'uptime'` | `-f 2` sets two parallel forks. |
| Inspect a module's help | `ansible-doc ansible.builtin.copy` | Shows parameters and examples for `copy`. |

**Use `ansible.builtin.command` by default** for normal commands. Use `ansible.builtin.shell` only for shell-specific features such as `|`, `>`, or variable expansion. Some modules report `changed` simply because they executed; do not confuse that with a real configuration change.

## 5. YAML and Playbook Structure

YAML uses **spaces, not tabs**. A hyphen (`-`) starts a list item; indentation shows which values belong together.

### YAML list and dictionary example

```yaml
employee:
  name: Alex
  skills:
    - Linux
    - Ansible
  address:
    city: Boston
    postal_code: "02108"
```

`employee` is a dictionary, `skills` is a list, and `postal_code` is quoted to keep the leading zero.

### Complete read-only playbook

**File: `check.yml`**

```yaml
---
- name: Check Linux web servers
  hosts: web_nodes
  gather_facts: false

  tasks:
    - name: Read system uptime
      ansible.builtin.command: uptime
      register: uptime_result
      changed_when: false

    - name: Show uptime
      ansible.builtin.debug:
        var: uptime_result.stdout

    - name: Show hostname
      ansible.builtin.command: hostname
      register: hostname_result
      changed_when: false

    - name: Print hostname
      ansible.builtin.debug:
        var: hostname_result.stdout
```

| YAML field | Meaning in this playbook |
|---|---|
| `---` | Optional YAML document-start marker. |
| `- name:` under the top level | Begins a play; gives it a readable name. |
| `hosts: web_nodes` | Targets hosts in the `web_nodes` inventory group. |
| `gather_facts: false` | Skips automatic fact collection for this simple check. |
| `tasks:` | Begins the ordered list of tasks. |
| `ansible.builtin.command` | Runs a program without invoking a shell. |
| `register: uptime_result` | Saves that task's result in a variable. |
| `changed_when: false` | Marks a read-only check as unchanged. |
| `ansible.builtin.debug` | Prints the selected result value. |

Run it with `ansible-playbook -i inventory.ini check.yml`. A successful run reports `unreachable=0` and `failed=0` for each contacted host.

## 6. Essential Ansible Modules

Modules express the **desired result**, which is generally safer and more repeatable than executing raw shell commands.

| Task | Complete command | Explanation |
|---|---|---|
| Install Nginx on Ubuntu nodes | `ansible web_nodes -i inventory.ini -b -m ansible.builtin.apt -a 'name=nginx state=present update_cache=yes'` | Ensure `nginx` is installed; refresh apt metadata. |
| Start and enable Nginx | `ansible web_nodes -i inventory.ini -b -m ansible.builtin.service -a 'name=nginx state=started enabled=yes'` | Start service now and enable it at boot. |
| Create a remote directory | `ansible web_nodes -i inventory.ini -b -m ansible.builtin.file -a 'path=/opt/ansible-demo state=directory mode=0755'` | Ensure the directory exists. |
| Create a remote file | `ansible web_nodes -i inventory.ini -b -m ansible.builtin.file -a 'path=/tmp/ansible-note.txt state=touch mode=0644'` | Create/update a file; `touch` updates timestamps on repeats. |
| Copy local website to nodes | `ansible web_nodes -i inventory.ini -b -m ansible.builtin.copy -a 'src=index.html dest=/var/www/html/index.html mode=0644'` | Copies `index.html` from control node; create it in Project 4 first. |
| Create a lab user | `ansible web_nodes -i inventory.ini -b -m ansible.builtin.user -a 'name=labuser state=present create_home=yes'` | Ensure a local user exists on each target. |
| Remove the lab user | `ansible web_nodes -i inventory.ini -b -m ansible.builtin.user -a 'name=labuser state=absent remove=yes'` | Remove that test user and its home; use only for accounts you created for practice. |
| Install with generic package module | `ansible web_nodes -i inventory.ini -b -m ansible.builtin.package -a 'name=curl state=present'` | Uses each managed host's supported package manager. |

**Distribution note:** `ansible.builtin.apt` is for Debian/Ubuntu; `ansible.builtin.dnf` is for Fedora/RHEL-family systems. For a DNF-based host, add an *optional* group to `inventory.ini` (replace the sample address):

```ini
[rhel_nodes]
rhel01 ansible_host=192.0.2.15 ansible_user=student
```

Then run the complete DNF command on **only that group**:

```bash
ansible rhel_nodes -i inventory.ini -b -m ansible.builtin.dnf -a 'name=nginx state=present'
```

Do not run DNF against Ubuntu web nodes. For cross-distribution tasks, prefer `ansible.builtin.package` when package names match.

## 7. Running and Checking Playbooks

Use `check.yml`, `nginx.yml` and `deploy.yml` defined in this guide. The latter two change remote hosts, so confirm your target group before applying.

| Task | Complete command | Explanation |
|---|---|---|
| Check playbook syntax | `ansible-playbook -i inventory.ini check.yml --syntax-check` | Parse and validate playbook structure without running tasks. |
| List target hosts | `ansible-playbook -i inventory.ini nginx.yml --list-hosts` | Show which servers this playbook would target. |
| List tasks | `ansible-playbook -i inventory.ini nginx.yml --list-tasks` | Display the named tasks before execution. |
| Run a playbook | `ansible-playbook -i inventory.ini check.yml` | Run all tasks in the read-only example. |
| Preview supported changes | `ansible-playbook -i inventory.ini nginx.yml --check` | Check mode predicts supported module changes; not a guaranteed simulation. |
| Preview file differences | `ansible-playbook -i inventory.ini deploy.yml --check --diff` | Show expected file changes when supported; the output may contain sensitive content. |
| Run only one target | `ansible-playbook -i inventory.ini nginx.yml --limit node01` | Restrict execution to `node01`. |
| Run only the web group | `ansible-playbook -i inventory.ini nginx.yml --limit web_nodes` | Restrict to matching web hosts. |
| Show extra troubleshooting details | `ansible-playbook -i inventory.ini check.yml -vvv` | `-vvv` gives detailed connection and task diagnostics. |
| Ask for sudo password | `ansible-playbook -i inventory.ini nginx.yml --ask-become-pass` | Prompt for the remote privilege-escalation password. |
| Start at the installation task | `ansible-playbook -i inventory.ini nginx.yml --start-at-task 'Install Nginx'` | Begin at the matching task name, skipping earlier tasks. |

**Check-mode limitation:** Tasks using `command` or `shell` often cannot predict changes, and some tasks need other resources created first. A successful `--check` is **not** proof that a real run will succeed. Always review targets and run carefully.

### Read the play recap

| Result | Meaning |
|---|---|
| `ok` | Task finished successfully, including already-correct configurations. |
| `changed` | Ansible reports that a task modified the remote state. |
| `unreachable` | Ansible could not contact the target using the configured connection. |
| `failed` | Ansible connected, but a task did not succeed. |
| `skipped` | A condition, check mode or other rule skipped the task. |

## 8. Variables, Facts, Loops and Handlers

Use these features when a playbook needs to adapt to different hosts or avoid repeating the same tasks.

| Feature | Complete syntax example | Explanation |
|---|---|---|
| Variable | `vars: { site_label: "training" }` | Defines a reusable value for a play. |
| Variable usage | `msg: "Environment: {{ site_label }}"` | Inserts that value in a template or task parameter. |
| Registered result | `register: uptime_result` | Saves the result of a task under that name. |
| Gathered fact | `ansible_facts['os_family']` | OS family collected when facts are gathered. |
| Conditional task | `when: ansible_facts['os_family'] == 'Debian'` | Runs a task only on Debian-family hosts. |
| Loop | `loop: [logs, backups]` | Repeats a task with `item` set to each entry. |
| Template | `ansible.builtin.template` | Copies a Jinja2 template with variables rendered. |
| Notification | `notify: Reload Nginx` | Schedules a named handler only if the task reports a change. |
| Handler | `handlers:` | Defines special tasks run when notified. |

### Complete variables, facts, and loops example

**File: `facts-and-loops.yml`**

```yaml
---
- name: Practice variables and loops
  hosts: web_nodes
  become: true
  vars:
    demo_root: /tmp/ansible-demo

  tasks:
    - name: Show Linux distribution
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} runs {{ ansible_facts['distribution'] }}"

    - name: Create two directories
      ansible.builtin.file:
        path: "{{ demo_root }}/{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - logs
        - backups
      when: ansible_facts['os_family'] == 'Debian'
```

Run: `ansible-playbook -i inventory.ini facts-and-loops.yml`. On Debian-family web hosts, the two directories are created. On other Linux families, the directory task is skipped. The handler demonstration is in **Project 5**.

## 9. Secrets and Security

Ansible Vault encrypts selected files or variables. It protects secret values **at rest**, but those values still need careful handling when decrypted during execution.

| Task | Complete command | Explanation |
|---|---|---|
| Create encrypted file | `ansible-vault create secrets.yml` | Prompts for a Vault password, then opens an editor for new encrypted content. |
| View encrypted file | `ansible-vault view secrets.yml` | Prompts for the Vault password and displays decrypted content. |
| Edit securely | `ansible-vault edit secrets.yml` | Edits and re-encrypts the existing Vault file. |
| Encrypt an existing file | `ansible-vault encrypt secrets.yml` | Encrypts an existing **unencrypted** file; do not run against one already encrypted. |
| Run playbook with Vault | `ansible-playbook vault-check.yml --ask-vault-pass` | Prompts for the password used by the encrypted file. |

**Small Vault example:** Run `ansible-vault create secrets.yml`, then enter the following content into the editor and save it:

```yaml
demo_token: "replace-with-a-private-test-value"
```

**File: `vault-check.yml`** (checks the value but never prints it)

```yaml
---
- name: Check encrypted variable locally
  hosts: localhost
  connection: local
  gather_facts: false
  vars_files:
    - secrets.yml

  tasks:
    - name: Confirm a token is configured
      ansible.builtin.assert:
        that:
          - demo_token is defined
          - demo_token | length > 0
      no_log: true
```

Run: `ansible-playbook vault-check.yml --ask-vault-pass`.

**Security rules:** Never commit private SSH keys or plaintext credentials. Do not expose passwords through `debug`, command arguments, or logs. Use `no_log: true` on sensitive tasks. Keep Vault passwords somewhere secure and separate from encrypted files. Limit `become: true` to tasks that need elevated privileges.

## 10. Practical Automation Projects

These projects build on one another. **Projects 3-5 assume Ubuntu/Debian web nodes** with sudo access and package repositories reachable. The inventory uses placeholder IPs until you replace them.

### Project 1 - Connect to multiple Linux servers

1. Create `inventory.ini` from Section 3 and replace example IPs/user.
2. Confirm SSH works to `node01` and `node02`.
3. Run the following:

```bash
ansible-inventory -i inventory.ini --graph
ansible web_nodes -i inventory.ini -m ansible.builtin.ping
ansible web_nodes -i inventory.ini -m ansible.builtin.command -a 'uptime'
```

**Expected:** Two web hosts in the inventory graph, `pong` from reachable hosts, and uptime output. If a host is unreachable, check SSH before writing new playbooks.

### Project 2 - Run a multi-task playbook

Create `check.yml` exactly as in Section 5, then run:

```bash
ansible-playbook -i inventory.ini check.yml --syntax-check
ansible-playbook -i inventory.ini check.yml
```

**Expected:** Each contacted web host prints its uptime and hostname. The read-only commands use `changed_when: false`.

### Project 3 - Install and start Nginx

**File: `nginx.yml`**

```yaml
---
- name: Install and manage Nginx on Ubuntu hosts
  hosts: web_nodes
  become: true

  tasks:
    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure Nginx is running and enabled
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

Run and verify:

```bash
ansible-playbook -i inventory.ini nginx.yml --syntax-check
ansible-playbook -i inventory.ini nginx.yml
ansible web_nodes -i inventory.ini -m ansible.builtin.command -a 'systemctl is-active nginx'
```

**Expected:** Nginx becomes installed, active, and enabled. Re-running the same playbook should normally report no package/service changes. If sudo prompts are required, add `--ask-become-pass` to the playbook command.

### Project 4 - Deploy a website file

**File on the control node: `index.html`**

```html
<!doctype html>
<html lang="en">
  <head><meta charset="utf-8"><title>Ansible Lab</title></head>
  <body><h1>Website deployed by Ansible</h1></body>
</html>
```

**File: `deploy.yml`**

```yaml
---
- name: Deploy a static page to Ubuntu web nodes
  hosts: web_nodes
  become: true

  tasks:
    - name: Copy the website
      ansible.builtin.copy:
        src: index.html
        dest: /var/www/html/index.html
        mode: '0644'

    - name: Verify local HTTP response on each server
      ansible.builtin.uri:
        url: http://127.0.0.1/
        status_code: 200
```

Run:

```bash
ansible-playbook -i inventory.ini deploy.yml --syntax-check
ansible-playbook -i inventory.ini deploy.yml
```

**Expected:** Nginx serves the updated page from `/var/www/html/index.html`, and each remote HTTP check returns status 200. The verification does not require inbound web access from your laptop.

### Project 5 - Change a config using a variable and handler

A **handler** should run only when something actually changes. Here, changing a small Nginx header configuration reloads the service without rebuilding or reinstalling it.

**File: `demo-header.conf.j2`** (Jinja2 template on the control node)

```nginx
# Managed by Ansible
add_header X-Lab-Environment "{{ environment_label }}" always;
```

**File: `vars-handlers.yml`**

```yaml
---
- name: Manage a small Nginx configuration
  hosts: web_nodes
  become: true
  vars:
    environment_label: training

  tasks:
    - name: Ensure Nginx is installed
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure Nginx is running
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    - name: Deploy header configuration
      ansible.builtin.template:
        src: demo-header.conf.j2
        dest: /etc/nginx/conf.d/ansible-demo.conf
        mode: '0644'
        backup: true
      notify: Reload Nginx

    - name: Show managed hostname
      ansible.builtin.debug:
        msg: "Configured {{ inventory_hostname }}"

  handlers:
    - name: Reload Nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

Run:

```bash
ansible-playbook -i inventory.ini vars-handlers.yml --syntax-check
ansible-playbook -i inventory.ini vars-handlers.yml
ansible-playbook -i inventory.ini vars-handlers.yml
ansible web_nodes -i inventory.ini -b -m ansible.builtin.command -a 'nginx -t'
```

**Expected:** The first run copies the template and notifies `Reload Nginx`; the second run normally finds the same configuration and does not reload it. Editing `environment_label` and re-running triggers a new deployment and reload. The template assumes Ubuntu's normal `/etc/nginx/conf.d/*.conf` include inside the Nginx `http` configuration.

### Practice checklist

- [ ] Create and inspect inventory groups, aliases, and nested groups.
- [ ] Confirm SSH and `ansible.builtin.ping` work on two managed hosts.
- [ ] Execute `command` and `shell` correctly and explain the difference.
- [ ] Write, syntax-check, and run a multi-task YAML playbook.
- [ ] Install and start Nginx on Ubuntu web nodes.
- [ ] Deploy a file and verify its HTTP response.
- [ ] Use variables, conditions, loops, a template, and a handler.
- [ ] Create an encrypted Vault file without exposing credentials.

## 11. Troubleshooting

| Problem | Likely cause | Specific check or fix |
|---|---|---|
| `ansible: command not found` | Ansible not installed or not on PATH | Run `ansible --version`; check installation and terminal PATH. |
| Inventory does not show hosts | Wrong filename, INI header, or inventory syntax | Run `ansible-inventory -i inventory.ini --graph` and inspect the groups. |
| `UNREACHABLE!` | Wrong address, SSH user, credentials, network or port | Try `ssh student@REAL_HOST_IP` first; verify host address and firewall. |
| `Permission denied (publickey)` | SSH authentication failed | Check private-key permissions, correct SSH user, and remote `authorized_keys`. |
| `Missing sudo password` | Task requires privilege escalation | Run the playbook with `--ask-become-pass` if that user is allowed to use sudo. |
| `sudo: a password is required` | Remote sudo needs a password | Use `-K` for ad hoc commands or `--ask-become-pass` for playbooks. |
| `python not found` | Managed host lacks compatible Python | Install a supported Python on the managed Linux host before using normal Ansible modules. |
| `Syntax Error while loading YAML` | Tabs, indentation, or invalid YAML | Use spaces; check `ansible-playbook -i inventory.ini check.yml --syntax-check`. |
| `couldn't resolve module/action` | Misspelled module name or missing collection | Use full module name and check `ansible-doc MODULE`; install needed collections. |
| `No package matching nginx` | Wrong package manager, repository, or distro | Confirm OS facts; use `apt` on Debian/Ubuntu and `dnf` on Fedora/RHEL. |
| Task says `changed` on every run | Module behavior or non-idempotent task | Prefer `file`, `copy`, `apt`, `service`; use `changed_when: false` for read-only `command` tasks. |
| Copy task cannot find file | `src` file not on control node at expected path | Save `index.html` next to `deploy.yml` or provide its correct path. |
| `--check` skipped a command | Module cannot simulate arbitrary execution | Treat check mode as a partial preview, not a substitute for testing. |
| Handler never runs | Task did not change or `notify` name mismatches | Confirm the template really changed and `notify: Reload Nginx` matches the handler name. |
| Windows `win_ping` fails | WinRM not configured, certificate invalid, or dependency absent | Check listener on HTTPS port 5986, certificate trust, `pywinrm`, and the Windows collection. |
| Vault password error | Incorrect Vault password or wrong encrypted file | Confirm the right file and provide its matching Vault password; do not publish it. |

### Quick debugging sequence

1. `ansible-inventory -i inventory.ini --graph` - confirm the targets are correct.
2. `ssh student@192.0.2.11` - replace the sample address and test SSH.
3. `ansible node01 -i inventory.ini -m ansible.builtin.ping` - confirm Ansible connectivity.
4. `ansible-playbook -i inventory.ini check.yml --syntax-check` - confirm YAML validity.
5. `ansible-playbook -i inventory.ini check.yml -vvv` - inspect detailed errors if needed.

---

## Official References

- [Ansible documentation](https://docs.ansible.com/projects/ansible/latest/)
- [Installation guide](https://docs.ansible.com/projects/ansible/latest/installation_guide/intro_installation.html)
- [Inventory guide](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html)
- [Ad hoc commands](https://docs.ansible.com/projects/ansible/latest/command_guide/intro_adhoc.html)
- [Playbooks and handlers](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_handlers.html)
- [Ansible Vault](https://docs.ansible.com/projects/ansible/latest/vault_guide/index.html)
- [Managing Windows with WinRM](https://docs.ansible.com/projects/ansible/latest/os_guide/windows_winrm.html)

*This guide is for authorized lab and server administration. Use real host addresses, verify the target inventory, and review privileged operations before executing them.*
