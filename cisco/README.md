# Cisco IOS Automation Examples

This directory contains Ansible playbooks for automating Cisco IOS network devices.

## Prerequisites

1. **Ansible Collections**: Install required collections
   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

2. **Network Access**: Ensure SSH connectivity to Cisco devices
   - Default port: 22
   - SSH must be enabled on the switch: `ip ssh version 2`

3. **Credentials**: Update `secrets.yml` with appropriate credentials
   - `cisco_password`: SSH user password
   - `cisco_enable_password`: Enable mode password (if required)

4. **Inventory**: Update `hosts` file with your switch IP addresses/hostnames

## Configuration Files

- `ansible.cfg` - Ansible configuration optimized for network devices
- `hosts` - Inventory file with Cisco switch definitions
- `requirements.yml` - Required Ansible collections
- `secrets.yml` - Credentials (gitignored)

## Playbooks

### show_power.yml
Runs the `show power` command on Cisco switches to display power supply and PoE information.

**Usage**:
```bash
ansible-playbook show_power.yml
```

**Output**: 
- Displays power information on screen
- Optionally saves output to timestamped file in current directory

**Run against specific host**:
```bash
ansible-playbook show_power.yml --limit cisco-switch-01
```

**Test connectivity first**:
```bash
ansible cisco_switches -m ping
```

## Authentication Options

### Password Authentication (Default)
Credentials stored in `secrets.yml`:
```yaml
cisco_password: YourPassword
```

### SSH Key Authentication
1. Copy SSH public key to switch
2. Update `hosts` file:
   ```ini
   ansible_ssh_private_key_file = ~/.ssh/cisco_key
   ```
3. Comment out `ansible_password` line

### Enable Mode
If privileged commands require enable mode:
1. Uncomment in `hosts`:
   ```ini
   ansible_become = yes
   ansible_become_method = enable
   ansible_become_password = "{{ cisco_enable_password }}"
   ```
2. Add to `secrets.yml`:
   ```yaml
   cisco_enable_password: YourEnablePassword
   ```

## Common Issues

**SSH Connection Timeout**:
- Verify network connectivity: `ping <switch-ip>`
- Ensure SSH is enabled on switch: `show ip ssh`
- Check firewall rules

**Authentication Failed**:
- Verify credentials in secrets.yml
- Check user privileges on switch
- Ensure AAA configuration allows password authentication

**Module Not Found**:
- Install collections: `ansible-galaxy collection install -r requirements.yml`
- Verify installation: `ansible-galaxy collection list | grep cisco`

## Cisco IOS Switch Configuration Requirements

Minimum switch configuration for SSH access:
```
hostname <switch-name>
ip domain-name example.com
crypto key generate rsa modulus 2048
ip ssh version 2
line vty 0 15
 transport input ssh
 login local
username admin privilege 15 secret YourPassword
```

## Getting Started

1. **Install collections**:
   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

2. **Update inventory**: Edit `hosts` file with your switch IP address

3. **Set credentials**: Edit `secrets.yml` with your password

4. **Test connectivity**:
   ```bash
   ansible cisco_switches -m ping
   ```

5. **Run playbook**:
   ```bash
   ansible-playbook show_power.yml
   ```

## Troubleshooting

**Verbose output** for debugging:
```bash
ansible-playbook show_power.yml -vvv
```

**Syntax check**:
```bash
ansible-playbook show_power.yml --syntax-check
```

**List inventory**:
```bash
ansible-inventory --list
```

## Adding More Playbooks

Follow the pattern from `show_power.yml`:
1. Use `gather_facts: false` for network devices
2. Use `cisco.ios.ios_command` for show commands
3. Use `cisco.ios.ios_config` for configuration changes
4. Register and display output for visibility
5. Include error handling with `failed_when` or `ignore_errors`
