
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
