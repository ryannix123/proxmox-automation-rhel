# Proxmox VM Automation with Ansible

Automate the creation and configuration of virtual machines on Proxmox using Ansible and cloud-init. This solution enables rapid deployment of RHEL-based VMs from templates with automatic hostname assignment, network configuration, and initial system setup.

## Features

- 🚀 Automated VM cloning from templates
- 🔢 Automatic VMID assignment
- 🌐 Cloud-init integration for initial configuration
- 🔧 Customizable VM specifications (CPU, memory, storage)
- 📡 Automated network configuration (static or DHCP)
- 👥 User provisioning with SSH key deployment
- 📦 Bulk VM deployment with sequential naming

## Prerequisites

### Software Requirements

- **Proxmox VE**: Version 8.0 or higher (tested on Proxmox 9.x)
- **Ansible**: Version 2.9 or higher
- **Python**: Version 3.6 or higher

### Ansible Collections

```bash
ansible-galaxy collection install community.proxmox
```

### Proxmox Template Requirements

Your template VM must have the following packages installed:
- `cloud-init`
- `qemu-guest-agent`
- `cloud-utils-growpart` (optional, for automatic disk resizing)

## Setup Instructions

### 1. Create Proxmox API User

On your Proxmox host, create a dedicated API user for Ansible automation:

```bash
# Create API user
pveum user add ansible@pve

# Set password for the user
pveum passwd ansible@pve

# Grant full administrative permissions
pveum aclmod / -user ansible@pve -role Administrator

# Grant storage permissions
pveum aclmod /storage -user ansible@pve -role Administrator

# Grant SDN permissions (required for Proxmox 9.x)
pveum aclmod /sdn -user ansible@pve -role Administrator
```

#### Alternative: API Token Authentication (More Secure)

```bash
# Create API token
pveum user token add ansible@pve automation --privsep=0

# Note the token secret - you'll need this for Ansible
# Format: ansible@pve!automation=<secret-token-here>
```

#### Verify Permissions

```bash
# Check user permissions
pveum user permissions ansible@pve
```

You should see entries for:
- `/` (root path)
- `/storage`
- `/sdn`

### 2. Prepare RHEL Template with Cloud-Init

#### Install Required Packages

SSH into your RHEL VM and run:

```bash
# Install cloud-init and qemu-guest-agent
dnf install -y cloud-init qemu-guest-agent cloud-utils-growpart

# Enable qemu-guest-agent
systemctl enable qemu-guest-agent
systemctl start qemu-guest-agent
```

#### Configure Cloud-Init for Proxmox

```bash
# Configure cloud-init datasource
cat > /etc/cloud/cloud.cfg.d/99-proxmox.cfg << 'EOF'
datasource_list: [ NoCloud, ConfigDrive ]
datasource:
  NoCloud:
    fs_label: cidata
EOF

# Disable NetworkManager cloud-init interaction
cat > /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg << 'EOF'
network: {config: disabled}
EOF
```

#### Enable Cloud-Init Services

```bash
systemctl enable cloud-init.service
systemctl enable cloud-init-local.service
systemctl enable cloud-config.service
systemctl enable cloud-final.service
```

#### Clean the Template

Before converting to a template, clean up the VM:

```bash
# Clean cloud-init state
cloud-init clean --logs --seed

# Remove machine-id (will be regenerated on first boot)
truncate -s 0 /etc/machine-id
rm -f /var/lib/dbus/machine-id
ln -s /etc/machine-id /var/lib/dbus/machine-id

# Remove SSH host keys (will be regenerated)
rm -f /etc/ssh/ssh_host_*

# Clean logs and temporary files
find /var/log -type f -exec truncate -s 0 {} \;
rm -rf /tmp/*
rm -rf /var/tmp/*

# Clear bash history
history -c
cat /dev/null > ~/.bash_history

# Shutdown the VM
shutdown -h now
```

#### Convert VM to Template

On your Proxmox host:

```bash
# Convert VM to template (replace 104 with your VM ID)
qm template 104

# Verify qemu-guest-agent is responsive
qm agent 104 ping
```

### 3. Configure Ansible Playbook

#### Clone this Repository

```bash
git clone https://github.com/yourusername/proxmox-ansible-automation.git
cd proxmox-ansible-automation
```

#### Update Variables

Edit the playbook and update the following variables:

```yaml
vars:
  proxmox_host: "192.168.1.3"          # Your Proxmox host IP or hostname
  proxmox_user: "ansible@pve"          # API user created earlier
  proxmox_password: "your-password"    # API user password
  proxmox_node: "proxmox1"             # Your Proxmox node name
  template_name: "RHEL10"              # Name of your template
```

#### Secure Credentials with Ansible Vault (Recommended)

```bash
# Create encrypted variables file
ansible-vault create vars/proxmox_creds.yml
```

Add the following content:
```yaml
proxmox_password: "your-secure-password"
```

Update your playbook to use vault:
```yaml
vars_files:
  - vars/proxmox_creds.yml
```

Run playbook with vault:
```bash
ansible-playbook proxmox_template.yml --ask-vault-pass
```

## Usage

### Create a Single VM

```bash
ansible-playbook proxmox_template.yml -e "vm_name=myapp-server vm_memory=4096 vm_cores=2"
```

### Create Multiple VMs with Sequential Naming

```bash
ansible-playbook proxmox_template.yml -e "vm_base_name=trading-app vm_count=5"
```

This creates:
- trading-app-1
- trading-app-2
- trading-app-3
- trading-app-4
- trading-app-5

### Create VMs with Custom Network Configuration

```bash
ansible-playbook proxmox_template.yml -e "vm_base_name=web-server vm_count=3 starting_ip_octet=200"
```

This assigns IPs:
- 192.168.1.200
- 192.168.1.201
- 192.168.1.202

### Override Multiple Variables

```bash
ansible-playbook proxmox_template.yml \
  -e "vm_base_name=database" \
  -e "vm_count=2" \
  -e "vm_memory=16384" \
  -e "vm_cores=8" \
  -e "starting_ip_octet=50"
```

## Playbook Variables

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `proxmox_host` | Proxmox server IP/hostname | `192.168.1.3` |
| `proxmox_user` | API user | `ansible@pve` |
| `proxmox_password` | API password | `secret123` |
| `proxmox_node` | Proxmox node name | `proxmox1` |
| `template_name` | Template VM name | `RHEL10` |

### Optional Variables

| Variable | Description | Default | Example |
|----------|-------------|---------|---------|
| `vm_base_name` | Base name for VMs | `rhel10` | `trading-app` |
| `vm_count` | Number of VMs to create | `1` | `5` |
| `vm_memory` | Memory in MB | `4096` | `8192` |
| `vm_cores` | CPU cores | `2` | `4` |
| `network_subnet` | Network subnet | `192.168.1` | `10.0.0` |
| `network_gateway` | Default gateway | `192.168.1.1` | `10.0.0.1` |
| `network_netmask` | Network mask | `24` | `24` |
| `starting_ip_octet` | Starting IP last octet | `150` | `100` |
| `dns_servers` | DNS servers | `8.8.8.8 8.8.4.4` | `1.1.1.1` |
| `default_user` | Cloud-init user | `ansible` | `admin` |
| `auto_start` | Start VMs after creation | `true` | `false` |

## Cloud-Init Configuration

The playbook automatically configures VMs with:

- **Hostname**: Set to VM name
- **User Account**: Created with sudo access
- **SSH Keys**: Deployed from `~/.ssh/id_rsa.pub`
- **Network**: Static IP or DHCP
- **DNS**: Custom nameservers and search domains
- **Timezone**: Configurable (default: UTC)
- **Packages**: Install additional packages on first boot

### Example: Custom Cloud-Init Configuration

Modify the playbook to add custom configurations:

```yaml
# In the proxmox_kvm task, add:
ciuser: "ansible"
cipassword: "changeme123"
sshkeys: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"

# Static IP
ipconfig:
  ipconfig0: "ip=192.168.1.100/24,gw=192.168.1.1"

# Or DHCP
ipconfig:
  ipconfig0: "ip=dhcp"

# DNS
nameservers: "8.8.8.8 8.8.4.4"
searchdomain: "example.com"
```

## Troubleshooting

### Permission Denied Errors

**Error**: `403 Forbidden: Permission check failed`

**Solution**: Verify API user has correct permissions:
```bash
pveum user permissions ansible@pve
```

Ensure permissions exist for `/`, `/storage`, and `/sdn`.

### Clone Timeout Errors

**Error**: `Reached timeout while waiting for creating VM`

**Solution**: Increase the timeout value:
```yaml
timeout: 300  # Increase to 5 minutes (300 seconds)
```

For storage with slow I/O, increase further:
```yaml
timeout: 600  # 10 minutes
```

### Cloud-Init Not Working

**Symptoms**: VM boots but hostname, network, or users are not configured

**Solution**: 
1. Verify cloud-init is installed in template:
   ```bash
   ssh root@template-vm
   rpm -q cloud-init qemu-guest-agent
   ```

2. Check cloud-init status on cloned VM:
   ```bash
   cloud-init status
   cat /var/log/cloud-init.log
   ```

3. Ensure template was properly cleaned:
   ```bash
   cloud-init clean --logs --seed
   ```

### VM Not Visible in Proxmox UI

**Solution**: Enable and start qemu-guest-agent in the template:
```bash
systemctl enable qemu-guest-agent
systemctl start qemu-guest-agent
```

### SSH Connection Refused

**Solution**: 
1. Check firewall allows SSH:
   ```bash
   firewall-cmd --list-services
   firewall-cmd --permanent --add-service=ssh
   firewall-cmd --reload
   ```

2. Verify SSH service is running:
   ```bash
   systemctl status sshd
   ```

### Network Configuration Not Applied

**Solution**: Disable NetworkManager's cloud-init plugin:
```bash
cat > /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg << 'EOF'
network: {config: disabled}
EOF
```

## Advanced Usage

### Create VMs with Different Specifications

```yaml
# Define custom VM configurations
vms_to_create:
  - name: "web-01"
    memory: 4096
    cores: 2
    ip: "192.168.1.101"
  - name: "db-01"
    memory: 16384
    cores: 8
    ip: "192.168.1.102"
  - name: "app-01"
    memory: 8192
    cores: 4
    ip: "192.168.1.103"
```

### Integrate with Red Hat Satellite

Add to cloud-init configuration:
```yaml
runcmd:
  - curl -k https://satellite.example.com/pub/bootstrap.py -o /tmp/bootstrap.py
  - python3 /tmp/bootstrap.py --login=admin --password=changeme --activationkey=rhel-key
```

### Post-Deployment Configuration

Add additional tasks after VM creation:
```yaml
- name: Wait for VMs to be accessible
  wait_for:
    host: "{{ network_subnet }}.{{ starting_ip_octet + item }}"
    port: 22
    delay: 10
    timeout: 300
  loop: "{{ range(0, vm_count) | list }}"

- name: Add VMs to Ansible inventory
  add_host:
    name: "{{ vm_base_name }}-{{ item + 1 }}"
    groups: new_vms
    ansible_host: "{{ network_subnet }}.{{ starting_ip_octet + item }}"
  loop: "{{ range(0, vm_count) | list }}"

- name: Configure new VMs
  hosts: new_vms
  tasks:
    - name: Install additional packages
      dnf:
        name:
          - vim
          - git
          - htop
        state: present
```

## File Structure

```
proxmox-ansible-automation/
├── README.md
├── proxmox_template.yml          # Main playbook for VM creation
├── vars/
│   └── proxmox_creds.yml        # Ansible Vault encrypted credentials
├── templates/
│   └── cloud-init-userdata.j2   # Cloud-init configuration template
└── inventory/
    └── proxmox                   # Proxmox inventory file
```

## Security Best Practices

1. **Use Ansible Vault** for sensitive data:
   ```bash
   ansible-vault encrypt vars/proxmox_creds.yml
   ```

2. **Use API Tokens** instead of passwords when possible

3. **Limit API user permissions** to only what's needed:
   ```bash
   # Instead of Administrator, use specific roles
   pveum aclmod / -user ansible@pve -role PVEVMAdmin
   ```

4. **Use SSH keys** instead of passwords for VM access

5. **Enable firewall** on Proxmox and VMs

6. **Keep templates updated** with latest security patches

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Acknowledgments

- Proxmox VE team for excellent virtualization platform
- Ansible community for the `community.proxmox` collection
- Cloud-init project for VM initialization framework

## Support

For issues and questions:
- Open an issue on GitHub
- Check the [Proxmox documentation](https://pve.proxmox.com/pve-docs/)
- Review [Ansible Proxmox module docs](https://docs.ansible.com/ansible/latest/collections/community/proxmox/)

## Author

Created by Ryan Nix - Senior Solutions Architect at Red Hat

---

**Note**: This project is not officially associated with or endorsed by Proxmox, Red Hat, or Ansible.