# OPC DA Remote Connection Setup Guide
## Hyper-V VMs with Kepware Server and Matrikon Client

This guide provides complete step-by-step instructions for setting up remote OPC DA connectivity between two Hyper-V VMs using DCOM.

---

## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [1. Hyper-V Network Setup](#1-hyper-v-network-setup)
- [2. IP Address Validation](#2-ip-address-validation)
- [3. Connectivity Checks](#3-connectivity-checks)
- [4. Server VM DCOM / OPC Services Validation](#4-server-vm-dcom--opc-services-validation)
- [5. Windows Firewall + Sharing](#5-windows-firewall--sharing)
- [6. Fix Remote Registry + Admin Token Filtering](#6-fix-remote-registry--admin-token-filtering)
- [7. User Account Setup](#7-user-account-setup)
- [8. DCOM Settings](#8-dcom-settings)
- [9. OPCEnum ServerList Registry Fix](#9-opcenum-serverlist-registry-fix)
- [10. DCOM Hardening Compatibility](#10-dcom-hardening-compatibility)
- [11. Client VM Critical Setup](#11-client-vm-critical-setup)
- [12. Final Connection Test](#12-final-connection-test)
- [Troubleshooting](#troubleshooting)

---

## Architecture Overview

### Components

- **Server VM**: Kepware OPC DA Server
- **Client VM**: Matrikon OPC Explorer (OPC DA Client)

### Goal

Enable the client to browse Kepware remotely via DCOM.

---

## 1. Hyper-V Network Setup

### ⚠️ IMPORTANT: Do NOT Use Default Switch

The default switch causes unstable NAT/APIPA behavior and DCOM issues.

### 1.1 Create External Virtual Switch

1. Open **Hyper-V Manager**
2. Navigate to **Virtual Switch Manager**
3. Create **External Network**
4. Select your physical **Wi-Fi** or **Ethernet** adapter
5. ✅ Enable: **Allow management OS to share this adapter**
6. Assign both VMs to this switch

---

## 2. IP Address Validation

After VM restart, verify proper IP addresses:

### Server VM
- Must get proper LAN IP (example: `192.168.1.47`)

### Client VM
- Must get proper LAN IP (example: `192.168.1.46`)

### ⚠️ APIPA Warning

If you get `169.254.x.x` addresses:
- This is APIPA (Automatic Private IP Addressing)
- DHCP is not working
- Fix switch/adapter selection and restart VMs

---

## 3. Connectivity Checks

Run these tests from the **Client VM**:

### Test 1: Ping
```powershell
ping <ServerIP>
```

### Test 2: RPC Port (DCOM backbone)
```powershell
Test-NetConnection <ServerIP> -Port 135
```

### Test 3: RPC Dynamic Port
```powershell
Test-NetConnection <ServerIP> -Port 49664
```

**Expected Result**: `TcpTestSucceeded : True`

---

## 4. Server VM DCOM / OPC Services Validation

### 4.1 OPCEnum Service

Ensure the service is running:

```powershell
sc query opcenum
```

### 4.2 Kepware Runtime Service

```powershell
sc query kepserverexv6
```

Both services must be in the **RUNNING** state.

---

## 5. Windows Firewall + Sharing

**Apply these settings to BOTH VMs.**

### 5.1 Set Network Profile to Private

```powershell
Set-NetConnectionProfile -NetworkCategory Private
```

### 5.2 Enable Sharing + Discovery

1. Open **Control Panel**
2. Navigate to **Network and Sharing Center** → **Advanced Sharing Settings**
3. Enable:
   - ✅ **Network Discovery ON**
   - ✅ **File & Printer Sharing ON**

### 5.3 Allow Firewall Rules on Server VM

```powershell
netsh advfirewall firewall set rule group="File and Printer Sharing" new enable=Yes
```

---

## 6. Fix Remote Registry + Admin Token Filtering

**Apply these settings to the Server VM.**

### 6.1 Enable Remote Registry

```powershell
Set-Service RemoteRegistry -StartupType Automatic
Start-Service RemoteRegistry
```

### 6.2 Allow Remote Admin Access

```powershell
reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f
```

**Reboot the server after making these changes.**

---

## 7. User Account Setup

Create the same user on **BOTH VMs**:

### Account Details
- **Username**: `opcuser1`
- **Password**: (must be identical on both VMs)

### Group Membership
Add `opcuser1` to:
- **Administrators** group
- **opcgroup** (create if it doesn't exist)

---

## 8. DCOM Settings

**Apply these settings to the Server VM.**

### Launch DCOM Configuration

```powershell
dcomcnfg
```

### 8.1 My Computer → Default Properties

1. Expand **Component Services** → **Computers** → **My Computer**
2. Right-click **My Computer** → **Properties**
3. Set:
   - **Default Authentication Level**: `Connect`
   - **Default Impersonation Level**: `Identify`

### 8.2 COM Security (Edit Limits)

1. On **My Computer** → **Properties** → **COM Security** tab
2. Click **Edit Limits** for both:
   - Access Permissions
   - Launch and Activation Permissions

3. Add the following accounts:
   - `opcuser1`
   - `opcgroup`

4. Grant permissions:
   - ✅ Remote Access
   - ✅ Remote Launch
   - ✅ Remote Activation

### 8.3 Kepware DCOM Configuration

1. Navigate to **Component Services** → **Computers** → **My Computer** → **DCOM Config**
2. Find **KEPServerEX 6.18**
3. Right-click → **Properties**

#### General Tab
- **Authentication Level**: `Connect`

#### Security Tab
- **Launch and Activation Permissions**: `Customize`
  - Add `opcuser1` and `opcgroup`
  - Allow: Remote Launch, Remote Activation
- **Access Permissions**: `Customize`
  - Add `opcuser1` and `opcgroup`
  - Allow: Remote Access
- **Configuration Permissions**: `Customize`
  - Add `opcuser1` and `opcgroup`

#### Identity Tab
- Select: **This User**
- Enter: `opcuser1` credentials

### Restart Services

```powershell
net stop kepserverexv6
net start kepserverexv6
net stop opcenum
net start opcenum
```

---

## 9. OPCEnum ServerList Registry Fix

**Apply to Server VM.**

Kepware may be registered, but the OPCEnum list might be missing. Create it manually:

```powershell
reg add "HKLM\SOFTWARE\WOW6432Node\Classes\OPC.ServerList" /f
reg add "HKLM\SOFTWARE\WOW6432Node\Classes\OPC.ServerList\Kepware.KEPServerEX.V6" /f
reg add "HKLM\SOFTWARE\WOW6432Node\Classes\OPC.ServerList\Kepware.KEPServerEX.V6" /ve /t REG_SZ /d "KEPServerEX 6.18" /f
```

### Restart OPCEnum Service

```powershell
net stop opcenum
net start opcenum
```

### Verify Registry Entry

```powershell
reg query "HKLM\SOFTWARE\WOW6432Node\Classes\OPC.ServerList"
```

---

## 10. DCOM Hardening Compatibility

**Apply to Server VM.**

### 10.1 Disable Strict Integrity Requirements

```powershell
reg add "HKLM\SOFTWARE\Microsoft\Ole\AppCompat" /v "RequireIntegrityActivationAuthenticationLevel" /t REG_DWORD /d 0 /f
```

### 10.2 Add Kepware Exemption

```powershell
reg add "HKLM\SOFTWARE\Microsoft\Ole\AppCompat\ActivationSecurityCheckExemptionList" /v "{7BC0CC8E-482C-47CA-ABDC-0FE7F9C6E729}" /t REG_SZ /d "KEPServerEX" /f
```

**Reboot the server after making these changes.**

---

## 11. Client VM Critical Setup

### ✅ THE MOST IMPORTANT FIX

**Install OPC Core Components (x86) on Client VM**

Without this installation, Matrikon Explorer **CANNOT** connect remotely.

### Installation Steps

1. Download **OPC Core Components 3.00.108 (x86)** from OPC Foundation
2. Install on the Client VM
3. **Reboot the client VM**

This is the critical component that enables remote OPC DA browsing.

---

## 12. Final Connection Test

**Execute from Client VM.**

### Step 1: Authenticate IPC$

```powershell
net use \\<ServerIP>\IPC$ /user:SERVERA\opcuser1 *
```

Enter the password when prompted.

### Step 2: Open Matrikon Explorer

1. Launch **Matrikon OPC Explorer**
2. Browse for remote servers

### Expected Results ✅

- Kepware server is visible
- Tags are visible
- Values are readable

---

## 13. Connecting UA server of Fusion to LE on gateway/VM on same network 
Run this on where Fusion installed
### Inbound rule for 4840
```powershell
netsh advfirewall firewall add rule name="Litmus OPC Fusion UA 4840" dir=in action=allow protocol=TCP localport=4840
```
### outbound rule for 4840
```poweshell
netsh advfirewall firewall add rule name="Litmus OPC Fusion UA 4840 OUT" dir=out action=allow protocol=TCP localport=4840
```
---
## Troubleshooting

### Connection Fails

1. **Verify IP addresses** - No APIPA (`169.254.x.x`)
2. **Test connectivity** - All three connectivity checks must pass
3. **Check services** - Both `opcenum` and `kepserverexv6` must be running
4. **Verify DCOM settings** - Authentication level, security permissions
5. **Check firewall** - Network profile must be Private, sharing enabled
6. **Confirm OPC Core Components** - Must be installed on client

### DCOM Errors

- **Access Denied**: Check user permissions in DCOM Config
- **Server Unavailable**: Verify network connectivity and firewall rules
- **RPC Server Unavailable**: Check that port 135 is open

### Registry Issues

If OPCEnum doesn't list Kepware:
- Re-run Section 9 registry commands
- Restart OPCEnum service
- Verify with `reg query` command

### Still Not Working?

1. Reboot **both VMs**
2. Verify all steps in order
3. Check Windows Event Viewer for DCOM errors
4. Ensure identical user accounts exist on both VMs

---


# Patch README — Litmus OPC Fusion Remote Browse + Live Tag Update Fix

## Problem Statement
Litmus OPC Fusion was able to:
- Discover Kepware OPC DA Server remotely
- Connect to the server
- Manually map tags (ItemID)
- Read tag value (single/static)

But it was NOT able to:
- Browse tags remotely (error: "Failed to connect to server")
- Get continuous live updates (values not updating continuously)

Local setup worked fine.

---

## Root Cause
After applying RPC/DCOM related firewall rules and dynamic port range allowances,
the existing OPC DA/DCOM sessions were still using old cached connections.

Because of this:
- Browse interface calls failed
- Subscription callbacks (OPC DataChange) did not work
until the services were restarted.

---

## Fix Applied (Patch Steps)

### Step 1: Allow RPC Dynamic Ports (Both Server + Client)
Run as Admin on BOTH machines:

```cmd
netsh advfirewall firewall add rule name="DCOM RPC Dynamic Ports" dir=in action=allow protocol=TCP localport=49152-65535


## Additional Notes

- This configuration uses **OPC DA** (legacy), not OPC UA
- DCOM requires specific network and security configurations
- Always use External Virtual Switch, never Default Switch
- The OPC Core Components on the client is absolutely critical

---

## Version Information

- **Kepware**: KEPServerEX 6.18
- **Client Tool**: Matrikon OPC Explorer
- **OPC Standard**: OPC DA (Data Access)
- **Platform**: Windows on Hyper-V

---

## Security Considerations

⚠️ This configuration enables remote access for testing/development purposes. For production environments:

- Use stronger authentication methods
- Implement network isolation
- Apply principle of least privilege
- Consider migrating to OPC UA for better security

---

**Document Created**: 2026-02-17  
**Last Updated**: 2026-02-17
