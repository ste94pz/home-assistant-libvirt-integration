# Home Assistant Libvirt Integration

This is a custom integration for Home Assistant to monitor and control virtual machines using `virsh` over SSH or locally.

## Features

- View domain info and state
- Control power (start, shutdown, suspend, resume)
- Take screenshots from VMs
- Snapshot support
- Secure communication via SSH (remote hosts) or local direct connection
- **NEW**: Local libvirt connection support (no SSH required)

## Installation (via HACS)

1. Add this repository to HACS as a custom integration:
   - URL: `https://github.com/Bram-diederik/home-assistant-libvirt-integration`
2. Reboot Home Assistant.
3. Add the integration via YAML.

## Configuration

### Option 1: Local Connection (No SSH Required)

If Home Assistant is running on the same machine as libvirt, you can use a local connection by omitting the `ssh_host` parameter:

```yaml
sensor:
  - platform: libvirt
    uri: qemu:///system
    include_interfaces: true
    vms:
    - debian
    - kali

switch:
  - platform: libvirt
    uri: qemu:///system
```

**Requirements for local connection:**
- Home Assistant must have access to the libvirt socket (typically `/var/run/libvirt/libvirt-sock`)
- The Home Assistant process must have appropriate permissions (usually `libvirt` group membership or run as root)
- The `virsh` command must be available in the Home Assistant environment

### Option 2: SSH Connection (Remote Host)

To manage libvirt on a remote machine via SSH:

```yaml
sensor:
  - platform: libvirt
    uri: qemu://system
    ssh_host: "user@linux_host"
    include_interfaces: true
    vms:
    - debian
    - kali

switch:
  - platform: libvirt
    uri: "qemu://system"
    ssh_host: "user@linux_host"
```

#### SSH Key Setup

To allow Home Assistant to connect to your libvirt hosts via SSH:

1. Create an SSH key with no password:
   ```bash
   ssh-keygen -t ed25519 -f /share/libvirt/ssh_key -N ""
   ```

2. Copy it to the target host:
   ```bash
   ssh-copy-id -i /share/libvirt/ssh_key.pub user@linux_host
   ```

3. (Optional) Protect the authorized key with command restrictions using the script in `opt/`:
   ```bash
   command="/opt/ha_virt_protect.sh",no-agent-forwarding,no-user-rc,no-X11-forwarding,no-port-forwarding ssh-rsa AAAA... your-key-comment
   ```

**Note:** You may need to increase the SSH daemon's `MaxStartups` setting if using many SSH connections from Home Assistant.

## Dashboard Example

Create a text helper and a select helper for the snapshots:

```yaml
type: entities
entities:
  - entity: sensor.libvirt_kali
  - entity: switch.libvirt_kali
  - type: attribute
    entity: sensor.libvirt_kali
    attribute: ip
    name: IP Address
  - entity: input_select.libvirt_kali
  - type: custom:button-card
    name: Revert Snapshot
    icon: mdi:backup-restore
    tap_action:
      action: call-service
      service: libvirt.revert_snapshot
      service_data:
        name: kali
        snapshot: "[[[ return states[\"input_select.libvirt_kali\"].state ]]]"
    styles:
      card:
        - padding: 4px 8px
        - font-size: 12px
        - height: 30px
        - width: 220px
  - entity: input_text.libvirt_kali
  - type: custom:button-card
    name: Make Snapshot
    icon: mdi:camera
    tap_action:
      action: call-service
      service: libvirt.create_snapshot
      service_data:
        name: kali
        snapshot: "[[[ return states[\"input_text.libvirt_kali\"].state ]]]"
    styles:
      card:
        - padding: 4px 8px
        - font-size: 12px
        - height: 30px
        - width: 220px
```

## Housekeeping Automation

Example automation to take screenshots and manage snapshots:

```yaml
alias: libvirt screenshots and snapshots
description: make libvirt screenshots and manage the snapshots.
triggers:
  - minutes: /3
    trigger: time_pattern
conditions: []
actions:
  - data:
      name: kali
    action: libvirt.take_screenshot
  - action: input_select.set_options
    metadata: {}
    data:
      options: |-
        {{
          (state_attr('sensor.libvirt_kali', 'snapshots') 
          | default([]) 
          | map(attribute='name') 
          | list)
          or ['None'] }}
    target:
      entity_id: input_select.libvirt_kali
```

## Troubleshooting

### Local Connection Issues

- **Permission denied**: Ensure the Home Assistant user/container has access to the libvirt socket. In many cases, add the user to the `libvirt` group or run Home Assistant with appropriate permissions.
- **virsh command not found**: Make sure `virsh` is installed in your Home Assistant environment.
- **Connection refused**: Verify that the libvirt daemon is running and accessible.

### SSH Connection Issues

- **SSH key permissions**: Ensure SSH key file has `600` permissions: `chmod 600 /share/libvirt/ssh_key`
- **Host key verification failed**: The integration uses `StrictHostKeyChecking=accept-new` which accepts new host keys automatically.
- **Connection timeout**: Check your network and SSH server configuration.
