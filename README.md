# Network Device Activation Scripts Documentation

This directory contains two Expect scripts for activating silent network devices to trigger NAC (Network Access Control) role assignment on Aruba switches.

## Prerequisites

Both scripts require:
- `expect` command-line tool
- SSH access to network devices
- Environment variable `PWORD` set with SSH password

```bash
export PWORD='your_ssh_password'
```

## Scripts Overview

### cg_ping.exp

**Purpose**: Ping silent Cash Guard devices to activate network traffic, making them visible for NAC role assignment.

**Usage**:
```bash
./cg_ping.exp <client_ip>
```

**Process**:
1. Takes client IP as input (e.g., `10.181.195.62`)
2. Calculates switch IP: first two octets same, third octet +3, fourth octet = 252 (e.g., `10.181.198.252`)
3. SSH connects to the switch
4. Enables Aruba Central support mode
5. Configures VLAN 1 with IP: client's first 3 octets + `.10` (e.g., `10.181.195.10/24`)
6. Pings the client IP to generate traffic and activate the device
7. Cleans up: removes IP, disables support mode, exits

**Example**:
```bash
./cg_ping.exp 10.181.195.62
# Connects to: 10.181.198.252
# Configures: 10.181.195.10/24
# Pings: 10.181.195.62 (activates Cash Guard device for NAC)
```

### envo_ping.exp

**Purpose**: Two-step process to discover infrastructure and ping silent devices for NAC activation.

**Usage**:
```bash
./envo_ping.exp <client_ip>
```

**Process**:
1. **Step 1 - Router Discovery**:
   - Takes client IP as input (e.g., `10.181.195.64`)
   - Calculates router IP: client's first 3 octets + `.1` (e.g., `10.181.195.1`)
   - SSH connects to router
   - Executes `show ip interface brief | inc 3999` to find VLAN 3999 IP

2. **Step 2 - Device Activation**:
   - Calculates switch IP: VLAN 3999's first 3 octets + `.253`
   - SSH connects to the switch
   - Pings client device to generate traffic for NAC role assignment

**Example**:
```bash
./envo_ping.exp 10.181.195.64
# Router: 10.181.195.1 → finds VLAN 3999: 10.205.167.1
# Switch: 10.205.167.253 → configures 10.181.195.10/24 → pings 10.181.195.64 (activates for NAC)
```

## Key Differences

| Feature | cg_ping.exp | envo_ping.exp |
|---------|-------------|---------------|
| **Steps** | Single step | Two steps |
| **Switch Discovery** | Calculated directly | Via router VLAN 3999 lookup |
| **Router Access** | Not required | Required |
| **Use Case** | Cash Guard devices (known switch pattern) | Dynamic switch discovery |
| **Purpose** | Direct device activation | Infrastructure discovery + activation |

## Common Features

Both scripts:
- Use `PWORD` environment variable for authentication
- Handle SSH first-time connection prompts
- Enable/disable Aruba Central support mode
- Configure temporary IP on VLAN 1
- Ping silent devices to generate network traffic for NAC activation
- Clean up configuration before exit
- Provide detailed progress output
- Include error handling and timeouts

## Network Access Control (NAC) Integration

These scripts are designed to work with NAC systems by:
- **Generating network traffic** from silent devices that may not be actively communicating
- **Triggering device visibility** in the network infrastructure
- **Enabling proper role assignment** based on device characteristics and policies
- **Supporting automated network onboarding** for devices that need traffic activation
- **Specifically targeting Cash Guard devices** (cg_ping.exp) which typically use IP addresses ending in 60-66

## Error Handling

Scripts will exit with error messages if:
- No client IP argument provided
- `PWORD` environment variable not set
- Invalid IP address format
- Connection timeouts
- Command execution failures
- VLAN 3999 not found (envo_ping.exp only)

## Notes

- Scripts use SSH options to skip host key verification
- Timeout is set to 30 seconds for most operations
- All configuration changes are cleaned up before script exit
- Scripts are designed for Aruba switches with specific command syntax
