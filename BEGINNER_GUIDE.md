# OpenStack Caracal — Ansible Deployment Guide for Beginners
### Complete Step-by-Step Guide: From Zero Ansible Knowledge to a Running Cloud

---

## Before You Start — What is This?

This guide helps you deploy a full multi-node OpenStack cloud automatically using **Ansible**. Instead of typing hundreds of commands manually on five different servers, you run **one command** from your laptop and Ansible does everything for you.

**You need:**
- 5 Ubuntu 22.04 servers (controller, network, compute1, compute2, storage)
- A laptop/desktop to run Ansible from (Mac, Linux, or Windows WSL)
- SSH access to all 5 servers
- About 30 minutes of setup time

---

## Part 1 — Install Ansible on Your Control Machine

Your control machine is your laptop or desktop — the one you type commands on. Ansible is only installed here, NOT on the servers.

### On Ubuntu / Debian / WSL:
```bash
sudo apt update
sudo apt install ansible python3-pip -y
pip3 install openstacksdk
```

### On macOS:
```bash
brew install ansible
pip3 install openstacksdk
```

### Verify Ansible is installed:
```bash
ansible --version
# You should see: ansible [core 2.x.x] or higher
```

---

## Part 2 — Download the Playbooks

Copy the entire `openstack-ansible/` folder to your control machine. The folder structure looks like this:

```
openstack-ansible/
├── site.yml                  ← Main playbook (run this!)
├── validate.yml              ← Check everything is working
├── inventories/
│   └── hosts.ini             ← YOUR SERVER IPs GO HERE
├── group_vars/
│   └── all.yml               ← YOUR PASSWORDS GO HERE
├── host_vars/                ← Per-server settings
└── roles/                    ← The automation logic (don't edit)
```

---

## Part 3 — Set Up SSH Access

Ansible connects to your servers using SSH. You need passwordless SSH access (using a key, not typing a password every time).

### Step 1: Generate an SSH key (skip if you already have one)
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/openstack_key
# Press Enter twice to skip passphrase
```

### Step 2: Copy your key to each server
Run this command **once for each of your 5 servers**, replacing the IP each time:
```bash
ssh-copy-id -i ~/.ssh/openstack_key.pub ubuntu@10.0.0.100   # controller
ssh-copy-id -i ~/.ssh/openstack_key.pub ubuntu@10.0.1.100   # network
ssh-copy-id -i ~/.ssh/openstack_key.pub ubuntu@10.0.2.100   # compute1
ssh-copy-id -i ~/.ssh/openstack_key.pub ubuntu@10.0.3.100   # compute2
ssh-copy-id -i ~/.ssh/openstack_key.pub ubuntu@10.0.4.100   # storage
```

### Step 3: Verify SSH works without a password
```bash
ssh -i ~/.ssh/openstack_key ubuntu@10.0.0.100 "echo SSH works!"
# Should print: SSH works!
```

### Step 4: Configure SSH for Ansible (optional but convenient)
```bash
# Add to ~/.ssh/config:
cat >> ~/.ssh/config << 'EOF'
Host 10.0.*.*
  User ubuntu
  IdentityFile ~/.ssh/openstack_key
  StrictHostKeyChecking no
EOF
```

---

## Part 4 — Configure Your Environment

This is the most important step. You tell Ansible about YOUR network and passwords.

### Step 1: Edit the inventory file
Open `inventories/hosts.ini` in any text editor:

```ini
[controller]
controller ansible_host=10.0.0.100 ansible_user=ubuntu

[network]
network ansible_host=10.0.1.100 ansible_user=ubuntu

[compute]
compute1 ansible_host=10.0.2.100 ansible_user=ubuntu
compute2 ansible_host=10.0.3.100 ansible_user=ubuntu

[storage]
storage ansible_host=10.0.4.100 ansible_user=ubuntu
```

**Change the IP addresses** (`ansible_host=...`) to match your actual servers.  
**Change `ansible_user`** if your server username is not `ubuntu` (e.g., use `root` if you set up root SSH).

### Step 2: Edit the variables file
Open `group_vars/all.yml` and update:

```yaml
# ---- YOUR NETWORK IPs ----
controller_mgmt_ip: "10.0.0.100"   # ← Change these
network_mgmt_ip:    "10.0.1.100"   # ← to your actual
compute1_mgmt_ip:   "10.0.2.100"   # ← server IPs
compute2_mgmt_ip:   "10.0.3.100"
storage_mgmt_ip:    "10.0.4.100"

# ---- YOUR NETWORK INTERFACES ----
provider_nic:   "ens4"    # ← Run "ip link show" on your servers
management_nic: "ens3"    # ← to find your actual NIC names

# ---- PASSWORDS ----
# Change ALL of these in production!
admin_password:    "AdminPass123"
service_password:  "service_pass"
rabbitmq_password: "RabbitPass123"
db_root_password:  "rootDBPass"

# ---- STORAGE DISK ----
# Run "lsblk" on your storage node to find the empty disk
cinder_storage_disk: "/dev/sdb"

# ---- VIRTUALIZATION ----
# "kvm" if your servers support hardware virtualization
# "qemu" if running inside VMs without nested virt
nova_virt_type: "kvm"
```

> ⚠️ **Find your network interface names** by running this on any server:
> ```bash
> ssh ubuntu@10.0.0.100 "ip link show"
> # Look for names like: ens3, ens4, enp0s3, eth0, eth1
> ```

### Step 3: Edit per-host IPs (if your hosts have different variable names)
The `host_vars/` files map each hostname to its management IP. If you renamed hosts in `hosts.ini`, update these files to match.

---

## Part 5 — Test Connectivity

Before running the full deployment, verify Ansible can reach all your servers:

```bash
cd openstack-ansible/

# Test connectivity to all hosts
ansible -i inventories/hosts.ini all_nodes -m ping

# Expected output for each host:
# controller | SUCCESS => {"ping": "pong"}
# network    | SUCCESS => {"ping": "pong"}
# compute1   | SUCCESS => {"ping": "pong"}
# ...
```

**If you see UNREACHABLE errors:**
- Check the IP addresses in `hosts.ini`
- Verify SSH works: `ssh ubuntu@<IP>`
- Check your firewall allows SSH (port 22)

**If you see "Missing sudo password":**
```bash
# Test with sudo:
ansible -i inventories/hosts.ini all_nodes -m ping --become --ask-become-pass
# Or add your user to sudoers without password on each server:
echo "ubuntu ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ubuntu
```

---

## Part 6 — Run the Deployment

Now the fun part. One command deploys everything:

```bash
cd openstack-ansible/

ansible-playbook -i inventories/hosts.ini site.yml
```

> ⏱️ **This takes 20–40 minutes.** Go make a coffee. Watch the output scroll by — green means success, red means something needs attention.

### What happens automatically:
1. All nodes get OS updates, NTP sync, /etc/hosts configured, and the OpenStack repo added
2. Network and compute nodes get OVS bridges configured
3. Controller gets MariaDB, RabbitMQ, Memcached, Keystone, Glance, Nova, Neutron, Horizon, and Cinder installed and configured
4. The network node gets L3, DHCP, and metadata agents
5. Compute nodes get nova-compute and neutron-openvswitch-agent
6. Storage node gets LVM set up and cinder-volume running
7. A test CirrOS image is uploaded to Glance

### To run only part of the deployment:
```bash
# Only run on the controller:
ansible-playbook -i inventories/hosts.ini site.yml --limit controller

# Only run specific tagged steps:
ansible-playbook -i inventories/hosts.ini site.yml --tags keystone

# Run in check mode (dry run — see what WOULD change):
ansible-playbook -i inventories/hosts.ini site.yml --check
```

---

## Part 7 — Validate the Deployment

After the playbook finishes, run the validation playbook:

```bash
ansible-playbook -i inventories/hosts.ini validate.yml
```

This checks that all services are running. You should see:
```
All nova services are UP
All neutron agents are ALIVE
All cinder services are UP
Hypervisors registered and UP
```

At the end you'll see a summary with your **Horizon URL and credentials**.

---

## Part 8 — Access Your Cloud

### Horizon Web Dashboard
Open a browser and go to:
```
http://10.0.0.100/horizon
```
*(replace with your controller IP)*

- **Domain:** Default
- **Username:** admin
- **Password:** whatever you set for `admin_password` in `all.yml`

### OpenStack CLI
SSH to the controller and run:
```bash
ssh ubuntu@10.0.0.100
source /root/admin-creds.sh

# List available images:
openstack image list

# List hypervisors:
openstack hypervisor list

# Launch a test VM:
openstack server create \
  --flavor m1.tiny \
  --image cirros \
  --network demo-net \
  test-vm
```

---

## Troubleshooting Common Issues

### "UNREACHABLE" error
- The server IP is wrong, or SSH isn't working
- Fix: `ssh ubuntu@<IP>` — can you connect manually?

### "Permission denied" error  
- Ansible can't use sudo
- Fix: Add `--ask-become-pass` to the command, or set up passwordless sudo

### Playbook fails on a specific task
- Read the error message carefully — Ansible tells you exactly what failed
- Run with `-v` (verbose) for more detail: `ansible-playbook ... site.yml -v`
- Run with `-vvv` for maximum detail

### Service fails to start
- SSH to the affected node and check logs:
```bash
systemctl status nova-compute     # or whatever service failed
journalctl -u nova-compute -n 50  # last 50 log lines
```

### Re-running after a failure
Ansible is **idempotent** — you can safely re-run the playbook. It only makes changes that are needed. If step 7 fails, fix the issue and run again; steps 1-6 will be skipped automatically.

```bash
# Re-run from where it failed:
ansible-playbook -i inventories/hosts.ini site.yml
```

---

## Quick Reference Card

| Task | Command |
|------|---------|
| Test connectivity | `ansible -i inventories/hosts.ini all_nodes -m ping` |
| Full deployment | `ansible-playbook -i inventories/hosts.ini site.yml` |
| Validate setup | `ansible-playbook -i inventories/hosts.ini validate.yml` |
| Deploy to one host | `ansible-playbook ... site.yml --limit controller` |
| Dry run | `ansible-playbook ... site.yml --check` |
| Verbose output | `ansible-playbook ... site.yml -v` |
| View inventory | `ansible-inventory -i inventories/hosts.ini --list` |

---

## Understanding the File Structure

```
openstack-ansible/
│
├── site.yml           The master playbook. Run this.
├── validate.yml       Checks all services are healthy after deployment.
│
├── inventories/
│   └── hosts.ini      YOUR server IPs and usernames. Edit this first.
│
├── group_vars/
│   └── all.yml        All passwords and settings. Edit this second.
│
├── host_vars/
│   ├── controller.yml  Per-host variables (management IP)
│   ├── network.yml
│   ├── compute1.yml
│   ├── compute2.yml
│   └── storage.yml
│
└── roles/
    ├── common/        Runs on ALL nodes (OS prep, NTP, repo)
    ├── controller/    Controller-only services (Keystone, Nova API, etc.)
    ├── network_node/  OVS kernel params + Neutron agents
    ├── compute/       Nova-compute + OVS agent on compute nodes
    └── storage/       LVM + Cinder volume on storage node
```

**Golden rule:** You only ever need to edit `hosts.ini` and `all.yml`. Everything else runs automatically.

---

*Built for OpenStack 2024.2 (Caracal) on Ubuntu 22.04 LTS*
