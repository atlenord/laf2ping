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
./cg_ping.exp <network>.6x     # Cash Guard range mode
```

**Process**:
1. Takes client IP as input (e.g., `10.181.195.62`)
2. **Mode Detection**: 
   - `.255` = Broadcast mode (pings all .2-.254)
   - `.6x` = Cash Guard range mode (pings all .60-.69)
   - Other = Single device mode
3. Calculates switch IP: first two octets same, third octet +3, fourth octet = 253 (e.g., `10.181.198.253`)
4. SSH connects to the switch
5. Enables Aruba Central support mode
6. Configures VLAN 1 with IP: client's first 3 octets + `.10` (e.g., `10.181.195.10/24`)
7. Executes ping based on detected mode
8. Cleans up: removes IP, disables support mode, exits

**Examples**:
```bash
# Single device activation
./cg_ping.exp 10.181.195.62
# Connects to: 10.181.198.253
# Configures: 10.181.195.10/24
# Pings: 10.181.195.62 (activates specific Cash Guard device)

# Cash Guard range activation (all Cash Guards in typical range)
./cg_ping.exp 10.181.195.6x
# Connects to: 10.181.198.253
# Configures: 10.181.195.10/24
# Pings: 10.181.195.60 through 10.181.195.69 (activates all Cash Guards)

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
./envo_ping.exp <network>.7x     # 7x range mode
```

**Process**:
1. **Step 1 - Router Discovery**:
   - Takes client IP as input (e.g., `10.181.195.64`)
   - **Mode Detection**: 
     - `.255` = Broadcast mode (pings all .2-.254)
     - `.7x` = 7x range mode (pings all .73-.77)
     - Other = Single device mode
   - Calculates router IP: client's first 3 octets + `.1` (e.g., `10.181.195.1`)
   - SSH connects to router
   - Executes `show ip interface brief | inc 3999` to find VLAN 3999 IP

2. **Step 2 - Device Activation**:
   - Calculates switch IP: VLAN 3999's first 3 octets + `.253`
   - SSH connects to the switch
   - Executes ping based on detected mode

**Examples**:
```bash
# Single device activation
./envo_ping.exp 10.181.195.64
# Router: 10.181.195.1 → finds VLAN 3999: 10.205.167.1
# Switch: 10.205.167.253 → configures 10.181.195.10/24 → pings 10.181.195.64

# 7x range activation (devices in 73-77 range)
./envo_ping.exp 10.181.195.7x
# Router: 10.181.195.1 → finds VLAN 3999: 10.205.167.1
# Switch: 10.205.167.253 → configures 10.181.195.10/24 → pings .73-.77

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
| **Ping Modes** | Single, Cash Guard range (.6x), Broadcast (.255) | Single, 7x range (.7x), Broadcast (.255) |

## Ping Modes

### Cash Guard Range Mode (cg_ping.exp only)

Specifically designed for Cash Guard devices that typically use IP addresses ending in 60-69:

**Activation**: Use `.6x` as the last octet:
```bash
./cg_ping.exp 10.181.195.6x     # Activates all Cash Guards (60-69) in 10.181.195.x
```

**Behavior**:
- **Range**: Pings IP addresses from `.60` to `.69` (10 devices)
- **Use Case**: Targeted activation of Cash Guard devices only
- **Performance**: ~30-60 seconds (much faster than broadcast)

### 7x Range Mode (envo_ping.exp only)

Specifically designed for devices that use IP addresses ending in 73-77:

**Activation**: Use `.7x` as the last octet:
```bash
./envo_ping.exp 10.181.195.7x     # Activates devices (73-77) in 10.181.195.x
```

**Behavior**:
- **Range**: Pings IP addresses from `.73` to `.77` (5 devices)
- **Use Case**: Targeted activation of specific device range
- **Performance**: ~15-30 seconds (very fast, focused range)

### Broadcast Mode

Both scripts support broadcast mode for activating all devices in a network segment:

**Activation**: Use `.255` as the last octet:
```bash
./cg_ping.exp 10.181.195.255     # Activates all devices in 10.181.195.x
./envo_ping.exp 10.181.195.255   # Activates all devices in 10.181.195.x
```

**Behavior**:
- **Range**: Pings all IP addresses from `.2` to `.254` in the /24 network
- **Skipped IPs**: `.0` (network), `.1` (gateway), `.255` (broadcast)
- **Use Case**: Mass activation of all devices in a store/location
- **Performance**: Approximately 4-5 minutes for full /24 range (253 IPs)

### Mode Comparison

| Mode | Trigger | IP Range | Device Count | Time | Use Case | Available In |
|------|---------|----------|--------------|------|----------|--------------|
| **Single** | Specific IP | One IP | 1 | ~5 seconds | Individual device | Both scripts |
| **Cash Guard** | `.6x` | .60-.69 | 10 | ~1 minute | Cash Guard devices only | cg_ping.exp |
| **7x Range** | `.7x` | .73-.77 | 5 | ~30 seconds | Specific device range | envo_ping.exp |
| **Broadcast** | `.255` | .2-.254 | 253 | ~5 minutes | All devices | Both scripts |

## Common Features

Both scripts:
- Use `PWORD` environment variable for authentication
- Handle SSH first-time connection prompts
- Enable/disable Aruba Central support mode
- Configure temporary IP on VLAN 1
- Support single device and broadcast ping modes
- **cg_ping.exp** additionally supports Cash Guard range mode (.6x)
- **envo_ping.exp** additionally supports 7x range mode (.7x)
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
