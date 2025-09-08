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
./cg_ping.exp <network>.255    # Broadcast mode
```

**Process**:
1. Takes client IP as input (e.g., `10.181.195.62`)
2. **Broadcast Detection**: If IP ends with `.255`, activates broadcast mode
3. Calculates switch IP: first two octets same, third octet +3, fourth octet = 253 (e.g., `10.181.198.253`)
4. SSH connects to the switch
5. Enables Aruba Central support mode
6. Configures VLAN 1 with IP: client's first 3 octets + `.10` (e.g., `10.181.195.10/24`)
7. **Single Mode**: Pings the specific client IP OR **Broadcast Mode**: Pings all IPs .2-.254
8. Cleans up: removes IP, disables support mode, exits

**Examples**:
```bash
# Single device activation
./cg_ping.exp 10.181.195.62
# Connects to: 10.181.198.253
# Configures: 10.181.195.10/24
# Pings: 10.181.195.62 (activates Cash Guard device for NAC)

# Broadcast activation (all devices in segment)
./cg_ping.exp 10.181.195.255
# Connects to: 10.181.198.253
# Configures: 10.181.195.10/24
# Pings: 10.181.195.2 through 10.181.195.254 (activates all devices)
```

### envo_ping.exp

**Purpose**: Two-step process to discover infrastructure and ping silent devices for NAC activation.

**Usage**:
```bash
./envo_ping.exp <client_ip>
./envo_ping.exp <network>.255    # Broadcast mode
```

**Process**:
1. **Step 1 - Router Discovery**:
   - Takes client IP as input (e.g., `10.181.195.64`)
   - **Broadcast Detection**: If IP ends with `.255`, activates broadcast mode
   - Calculates router IP: client's first 3 octets + `.1` (e.g., `10.181.195.1`)
   - SSH connects to router
   - Executes `show ip interface brief | inc 3999` to find VLAN 3999 IP

2. **Step 2 - Device Activation**:
   - Calculates switch IP: VLAN 3999's first 3 octets + `.253`
   - SSH connects to the switch
   - **Single Mode**: Pings specific client OR **Broadcast Mode**: Pings all IPs .2-.254

**Examples**:
```bash
# Single device activation
./envo_ping.exp 10.181.195.64
# Router: 10.181.195.1 → finds VLAN 3999: 10.205.167.1
# Switch: 10.205.167.253 → configures 10.181.195.10/24 → pings 10.181.195.64

# Broadcast activation (all devices in segment)
./envo_ping.exp 10.181.195.255
# Router: 10.181.195.1 → finds VLAN 3999: 10.205.167.1
# Switch: 10.205.167.253 → configures 10.181.195.10/24 → pings .2-.254
```

## Key Differences

| Feature | cg_ping.exp | envo_ping.exp |
|---------|-------------|---------------|
| **Steps** | Single step | Two steps |
| **Switch Discovery** | Calculated directly | Via router VLAN 3999 lookup |
| **Router Access** | Not required | Required |
| **Use Case** | Cash Guard devices (known switch pattern) | Dynamic switch discovery |
| **Purpose** | Direct device activation | Infrastructure discovery + activation |
| **Broadcast Support** | ✅ Single or all devices (.255) | ✅ Single or all devices (.255) |

## Broadcast Mode

Both scripts support broadcast mode for activating all devices in a network segment:

### Activation
Use `.255` as the last octet to enable broadcast mode:
```bash
./cg_ping.exp 10.181.195.255     # Activates all devices in 10.181.195.x
./envo_ping.exp 10.181.195.255   # Activates all devices in 10.181.195.x
```

### Behavior
- **Range**: Pings all IP addresses from `.2` to `.254` in the /24 network
- **Skipped IPs**: `.0` (network), `.1` (gateway), `.255` (broadcast)
- **Use Case**: Mass activation of Cash Guard devices in a store/location
- **Output**: Shows progress for each IP being pinged

### Performance
- **Time**: Approximately 4-5 minutes for full /24 range (253 IPs)
- **Timeout**: Individual ping timeouts don't stop the process
- **Logging**: Each IP ping attempt is logged for troubleshooting

## Common Features

Both scripts:
- Use `PWORD` environment variable for authentication
- Handle SSH first-time connection prompts
- Enable/disable Aruba Central support mode
- Configure temporary IP on VLAN 1
- Support both single device and broadcast ping modes
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
