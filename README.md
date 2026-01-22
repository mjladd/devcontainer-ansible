# README

## TODO

- [ ] setup k8s

## ToC

- [Setup Ansible User](#setup-ansible-user)
- [Roles vs Playbooks](#roles-vs-playbooks)
- [Updates and Patching](#updates-and-patching)
- [Proxmox Cleanup](#proxmox-cleanup)
- [Generate Proxmox Tokens](#generate-proxmox-tokens)
- [Setup Datadog on Proxmox](#setup-datadog-proxmox)

## Setup Ansible User

```shell
# SSH into each node and run:
sudo adduser ansible
# You'll be prompted for a password - set one temporarily
<random_password_here>

# Add ansible user to sudo group
sudo usermod -aG sudo ansible

# add dedicated sudoer file
echo "ansible ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ansible
sudo chmod 0440 /etc/sudoers.d/ansible

# on the control machine
> ssh-keygen -t rsa -b 4096 -C "ansible@control"
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/<username>/.ssh/id_rsa): /Users/mladd/.ssh/ansible-user
Enter passphrase for "/Users/mladd/.ssh/ansible-user" (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /Users/mladd/.ssh/ansible-user

# Copy the public key to each managed node
ssh-copy-id ansible@node1
ssh-copy-id ansible@node2
ssh-copy-id ansible@node3

# lock password for ansible user on all nodes
sudo passwd -l ansible

```

## Roles vs Playbooks

- Playbooks - Orchestration, combining multiple roles, one-off tasks
- Roles - Reusable components (install nginx, configure firewall, etc.)

## Updates and Patching

```shell
# Syntax check
ansible-playbook playbooks/update-packages.yml --syntax-check

# Dry run (check mode)
ansible-playbook playbooks/update-packages.yml --check

# Run on specific hosts
ansible-playbook playbooks/update-packages.yml --limit kn01.consul,kn02.consul

# Run with verbose output
ansible-playbook playbooks/update-packages.yml -v

# Run and auto-reboot if needed
ansible-playbook playbooks/update-and-reboot.yml -e "auto_reboot=true"
```


## Proxmox Cleanup

### Proxmox Repo

Remove the Proxmox Enterprise Repo and logon nag box.

- In the proxmox web gui > navigate to the machine > click on updates > repositories.
- select `enterprise.proxmox.com/debian/pve` click disable
- select `Add` button, click `ok` that you don't have a subscription
- click on the drop-down menu for `Enterprise` and select `No-Subscription`

### Proxmox Nag Box

```shell
 cd /usr/share/javascript/proxmox-widget-toolkit
 vi proxmoxlib.js
 # replace Ext.Mag.show with void
Ext.Msg.show({
 title: gettext('No valid subscription'),

# with
void({
 title: gettext('No valid subscription'),
```

## Generate PROXMOX tokens

Via Web UI:
- Go to Datacenter → Permissions → API Tokens
- Click "Add"
- Select user (or create one like monitoring@pve)
- Enter token ID (e.g., datadog)
- Uncheck "Privilege Separation" if you want token to inherit user permissions
- Copy the secret shown (only displayed once)

2. Via CLI on Proxmox host:
# Create user (if needed)
pveum user add monitoring@pve --comment "Datadog monitoring"

# Give read-only access
pveum acl modify / --user monitoring@pve --role PVEAuditor

# Create API token
pveum user token add monitoring@pve datadog

## Setup Datadog on Proxmox

- add ansible user as above (requires adding `sudo` package to proxmox host)

```shell
# Basic Proxmox monitoring configuration
ansible-playbook playbooks/configure-datadog-proxmox-monitoring.yml

# Advanced with API integration
export PROXMOX_TOKEN_ID="monitoring@pve!datadog"
export PROXMOX_TOKEN_SECRET="your-token-uuid"
ansible-playbook playbooks/configure-datadog-proxmox-advanced.yml

# Verify checks are running
ansible <hostname> -m command -a "datadog-agent status" --become

# Check specific integration
ansible <hostname> -m command -a "datadog-agent check disk" --become
ansible <hostname> -m command -a "datadog-agent check process" --become


# Check all running checks
ansible <hostname> -m shell -a "datadog-agent status | grep -A 50 'Checks'" --become

# View custom metrics script output
ansible <hostname> -m command -a "/usr/local/bin/datadog-proxmox-metrics.sh" --become

# Check cron jobs
ansible <hostname> -m command -a "crontab -l" --become

# Tail Datadog logs
ansible <hostname> -m shell -a "tail -n 50 /var/log/datadog/agent.log" --become

# Install DD client on VMs
export DD_API_KEY="XXXXXXXXXXXXX-key-here"
ansible-playbook playbooks/install-datadog-agent.yml
```
