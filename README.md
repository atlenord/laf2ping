# Network Device Activation Scripts Documentation

This directory contains three Expect scripts for activating silent network devices to trigger NAC (N| Feature | cg_ping.exp | envo_ping.exp | user_ping.exp |
|---------|-------------|---------------|---------------|
| **Steps** | Single step | Two steps | Single step |
| **Switch Discovery** | Calculated directly | Via router VLAN 3999 lookup | Not required |
| **Router Access** | Not required | Required | Required (gateway) |
| **Use Case** | Cash Guard devices (known switch pattern) | Dynamic switch discovery | Direct gateway ping |
| **Purpose** | Direct device activation | Infrastructure discovery + activation | Gateway-based ping activation |
| **Ping Modes** | Single, Cash Guard range (.6x), Broadcast (.255) | Single, 7x range (.7x), Broadcast (.255) | Single, Range (x-y), Broadcast (.255) |ccess Control) role assignment on Aruba switches.
Tested on linuxmaster

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
   - Takes client IP as input (e.g., `10.54.140.73`)
   - **Mode Detection**: 
     - `.255` = Broadcast mode (pings all .2-.254)
     - `.7x` = 7x range mode (pings all .73-.77)
     - Other = Single device mode
   - Calculates router IP: client's first 3 octets + `.1` (e.g., `10.54.140.1`)
   - SSH connects to router
   - Executes `show ip interface brief | inc 3999` to find VLAN 3999 IP

2. **Step 2 - Device Activation**:
   - Calculates switch IP: VLAN 3999's first 3 octets + `.253`
   - SSH connects to the switch
   - Executes ping based on detected mode

**Examples**:
```bash
# Single device activation
./envo_ping.exp 10.54.140.73
# Router: 10.54.140.1 → finds VLAN 3999: 10.205.167.1
# Switch: 10.205.167.253 → configures 10.54.140.10/24 → pings 10.54.140.73

# 7x range activation (devices in 73-77 range)
./envo_ping.exp 10.54.140.7x
# Router: 10.54.140.1 → finds VLAN 3999: 10.205.167.1
# Switch: 10.205.167.253 → configures 10.54.140.10/24 → pings .73-.77

# Broadcast activation (all devices in segment)
./envo_ping.exp 10.54.140.255
# Router: 10.54.140.1 → finds VLAN 3999: 10.205.167.1
# Switch: 10.205.167.253 → configures 10.54.140.10/24 → pings .2-.254
```

### user_ping.exp

**Purpose**: Simple ping from gateway device to activate network traffic directly from the network gateway.

**Usage**:
```bash
./user_ping.exp <client_ip>
./user_ping.exp <network>.255      # Broadcast mode
./user_ping.exp <network>.60-69    # Range mode
```

**Process**:
1. Takes client IP as input (e.g., `10.54.140.73`)
2. **Mode Detection**: 
   - `.255` = Broadcast mode (pings all .2-.254)
   - Range format (e.g., `.60-69`) = Range mode (pings specified range)
   - Other = Single device mode
3. Calculates gateway IP: client's first 3 octets + `.1` (e.g., `10.54.140.1`)
4. SSH connects to the gateway
5. Executes ping commands based on detected mode
6. Exits gateway connection

**Examples**:
```bash
# Single device activation
./user_ping.exp 10.54.140.73
# Gateway: 10.54.140.1 → pings 10.54.140.73 (4 ping packets)

# Range activation (devices in specified range)
./user_ping.exp 10.54.140.73-77
# Gateway: 10.54.140.1 → pings 10.54.140.73 through 10.54.140.77 (2 pings each)

# Range activation (Cash Guard devices)
./user_ping.exp 10.54.140.60-69
# Gateway: 10.54.140.1 → pings 10.54.140.60 through 10.54.140.69 (2 pings each)

# Broadcast activation (all devices in segment)
./user_ping.exp 10.54.140.255
# Gateway: 10.54.140.1 → pings 10.54.140.2 through 10.54.140.254 (2 pings each)
```

## Key Differences

| Feature | cg_ping.exp | envo_ping.exp | user_ping.exp |
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
./envo_ping.exp 10.54.140.7x     # Activates devices (73-77) in 10.54.140.x
```

**Behavior**:
- **Range**: Pings IP addresses from `.73` to `.77` (5 devices)
- **Use Case**: Targeted activation of specific device range
- **Performance**: ~15-30 seconds (very fast, focused range)

### Range Mode (user_ping.exp only)

Flexible range mode that allows pinging any specified range of IP addresses:

**Activation**: Use range format as the last octet:
```bash
./user_ping.exp 10.54.140.60-69     # Activates devices (60-69) in 10.54.140.x
./user_ping.exp 10.54.140.73-77     # Activates devices (73-77) in 10.54.140.x
./user_ping.exp 10.54.140.100-110   # Activates devices (100-110) in 10.54.140.x
```

**Behavior**:
- **Range**: Pings IP addresses from start to end range (inclusive)
- **Use Case**: Flexible targeted activation of any device range
- **Performance**: Depends on range size (~2-5 seconds per IP)

### Broadcast Mode

Both scripts support broadcast mode for activating all devices in a network segment:

**Activation**: Use `.255` as the last octet:
```bash
./cg_ping.exp 10.181.195.255     # Activates all devices in 10.181.195.x
./envo_ping.exp 10.54.140.255    # Activates all devices in 10.54.140.x
./user_ping.exp 10.54.140.255    # Activates all devices in 10.54.140.x
```

**Behavior**:
- **Range**: Pings all IP addresses from `.2` to `.254` in the /24 network
- **Skipped IPs**: `.0` (network), `.1` (gateway), `.255` (broadcast)
- **Use Case**: Mass activation of all devices in a store/location
- **Performance**: Approximately 4-5 minutes for full /24 range (253 IPs)

### Mode Comparison

| Mode | Trigger | IP Range | Device Count | Time | Use Case | Available In |
|------|---------|----------|--------------|------|----------|--------------|
| **Single** | Specific IP | One IP | 1 | ~5 seconds | Individual device | All scripts |
| **Cash Guard** | `.6x` | .60-.69 | 10 | ~1 minute | Cash Guard devices only | cg_ping.exp |
| **7x Range** | `.7x` | .73-.77 | 5 | ~30 seconds | Specific device range | envo_ping.exp |
| **Range** | `.x-y` | .x-.y | Variable | ~2-5s per IP | Flexible device range | user_ping.exp |
| **Broadcast** | `.255` | .2-.254 | 253 | ~5 minutes | All devices | All scripts |

## Common Features

All scripts:
- Use `PWORD` environment variable for authentication
- Handle SSH first-time connection prompts
- Support single device and broadcast ping modes
- Ping silent devices to generate network traffic for NAC activation
- Clean up configuration before exit (where applicable)
- Provide detailed progress output
- Include error handling and timeouts

**Script-specific features**:
- **cg_ping.exp**: Enables/disables Aruba Central support mode, configures temporary IP on VLAN 1, supports Cash Guard range mode (.6x)
- **envo_ping.exp**: Two-step router discovery process, enables/disables Aruba Central support mode, configures temporary IP on VLAN 1, supports 7x range mode (.7x)
- **user_ping.exp**: Simple gateway-based ping, supports flexible range mode (.x-y format)

## Network Access Control (NAC) Integration

These scripts are designed to work with NAC systems by:
- **Generating network traffic** from silent devices that may not be actively communicating
- **Triggering device visibility** in the network infrastructure
- **Enabling proper role assignment** based on device characteristics and policies
- **Supporting automated network onboarding** for devices that need traffic activation
- **Specifically targeting Cash Guard devices** (cg_ping.exp) which typically use IP addresses ending in 60-66
- **Providing flexible gateway-based activation** (user_ping.exp) for simple network troubleshooting and device activation

## Error Handling

Scripts will exit with error messages if:
- No client IP argument provided
- `PWORD` environment variable not set
- Invalid IP address format
- Connection timeouts
- Command execution failures
- VLAN 3999 not found (envo_ping.exp only)
- Gateway unreachable (user_ping.exp)

## Notes

- Scripts use SSH options to skip host key verification
- Timeout is set to 30 seconds for most operations
- All configuration changes are cleaned up before script exit (cg_ping.exp and envo_ping.exp)
- **cg_ping.exp** and **envo_ping.exp** are designed for Aruba switches with specific command syntax
- **user_ping.exp** works with any SSH-accessible gateway/router that supports ping commands
