# Troubleshooting: Error 0x80070035 — "The network path was not found"

## Symptom

When browsing **Network** in File Explorer and double-clicking a remote computer (e.g. `\\PC1`), Windows displays:

> **Network Error**
> Windows cannot access \\PC1
> Check the spelling of the name. Otherwise, there might be a problem with your network. To try to identify and resolve network problems, click Diagnose.
>
> Error code: 0x80070035
> The network path was not found.

The remote computer is visible in Network discovery, but its shares cannot be opened.

---

## Environment

| Item | Value |
|---|---|
| Client OS | Windows 10 / Windows 10 Pro |
| Server / host | Windows 10 PC or NAS/router share (e.g. `\\PC1`, `\\PC2`) |
| Protocol | SMB2 / SMB3 |
| Access type | Guest / no credentials, or workgroup (non-domain) |

---

## Likely Cause

Modern Windows builds harden SMB by default. Two settings commonly break access to workgroup PCs, NAS boxes, and router-attached USB shares:

1. **Insecure guest logons are blocked** — Windows refuses unauthenticated (guest) SMB2/3 sessions, which is what most NAS/router shares and simple workgroup shares use.
2. **SMB security signing is required** — the client demands signed packets; older or lightweight SMB servers do not support signing, so the session is rejected before the share list is returned.

The error surfaces as "network path was not found" even though the host is reachable, because the SMB session negotiation fails rather than the name resolution.

---

## Resolution

Run **PowerShell as Administrator** on the client PC and enter:

```powershell
Set-SmbClientConfiguration -EnableInsecureGuestLogons $true -Force
Set-SmbClientConfiguration -RequireSecuritySignature $false -Force
Set-SmbServerConfiguration -RequireSecuritySignature $false -Force
```

Then retry `\\PC1` in File Explorer or via **Win + R** → `\\PC1`.

No reboot is normally required. If access still fails, restart the Workstation service or reboot:

```powershell
Restart-Service LanmanWorkstation -Force
```

### Verify the settings applied

```powershell
Get-SmbClientConfiguration | Select-Object EnableInsecureGuestLogons, RequireSecuritySignature
Get-SmbServerConfiguration | Select-Object RequireSecuritySignature
```

Expected: `EnableInsecureGuestLogons = True`, both `RequireSecuritySignature = False`.

---

## ⚠️ Security Note

These commands **lower SMB security** on the client:

- Guest logons are unauthenticated and unencrypted — traffic and credentials-free sessions can be intercepted or spoofed on an untrusted network.
- Disabling required signing removes protection against man-in-the-middle / SMB relay attacks.

Apply only on a **trusted LAN** (office or home network you control). Do not apply on laptops that connect to public Wi-Fi, or on domain-joined machines without approval from IT/security.

### Rollback (restore defaults)

```powershell
Set-SmbClientConfiguration -EnableInsecureGuestLogons $false -Force
Set-SmbClientConfiguration -RequireSecuritySignature $true -Force
Set-SmbServerConfiguration -RequireSecuritySignature $true -Force
```

---

## If the Error Persists — Additional Checks

Work through these in order.

### 1. Confirm the host is reachable

```powershell
ping PC1
Test-NetConnection PC1 -Port 445
```

- Ping fails but IP works → name resolution issue; try `\\192.168.x.x` and check DNS/NetBIOS.
- Port 445 closed → firewall on the host, or File and Printer Sharing is off.

### 2. Enable File and Printer Sharing on the host

Control Panel → Network and Sharing Center → **Change advanced sharing settings**:

- Turn on **Network discovery**
- Turn on **File and printer sharing**
- Under **All Networks** → turn **off** password-protected sharing (for guest-style access)

Confirm the network profile is **Private**, not Public:

```powershell
Get-NetConnectionProfile
Set-NetConnectionProfile -InterfaceAlias "Ethernet" -NetworkCategory Private
```

### 3. Check required services are running (both machines)

```powershell
Get-Service LanmanWorkstation, LanmanServer, FDResPub, SSDPSRV, upnphost |
    Select-Object Name, Status, StartType
```

Start anything stopped:

```powershell
Start-Service LanmanWorkstation, LanmanServer, FDResPub
```

### 4. Allow SMB through Windows Firewall

```powershell
Enable-NetFirewallRule -DisplayGroup "File and Printer Sharing"
Enable-NetFirewallRule -DisplayGroup "Network Discovery"
```

Third-party antivirus/firewall suites often block SMB independently — test with them temporarily disabled.

### 5. Enable SMB 1.0 / CIFS client (legacy devices only)

Needed for very old NAS units, routers, and Windows XP-era hosts.

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol-Client
```

> SMB1 is deprecated and vulnerable (WannaCry-class exploits). Enable only if the device genuinely cannot do SMB2, and disable it again once the device is replaced:
> `Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol`

### 6. Clear stale cached credentials

An old saved credential for the host will silently block a new session.

```cmd
net use * /delete /y
cmdkey /list
cmdkey /delete:PC1
```

Then map explicitly with credentials:

```cmd
net use \\PC1\ShareName /user:user1\username *
```

### 7. Verify the share actually exists on the host

Run on **PC1**:

```powershell
Get-SmbShare
```

Then try connecting directly to a named share rather than browsing the root:

```
\\PC1\SharedFolder
```

### 8. Reset the network stack (last resort)

```cmd
netsh winsock reset
netsh int ip reset
ipconfig /flushdns
nbtstat -R
```

Reboot afterwards.

---

## Quick Reference Checklist

- [ ] Ran the three `Set-Smb*Configuration` commands as Administrator
- [ ] Verified settings with `Get-SmbClientConfiguration`
- [ ] Host responds to ping and TCP 445
- [ ] Network profile set to **Private** on both machines
- [ ] Network discovery + File and printer sharing enabled
- [ ] Password-protected sharing off (guest access scenarios)
- [ ] `LanmanWorkstation` / `LanmanServer` running
- [ ] Firewall rules for File and Printer Sharing enabled
- [ ] Stale credentials cleared with `net use * /delete`
- [ ] Share confirmed present via `Get-SmbShare` on the host
- [ ] Security settings reverted if the machine leaves the trusted LAN
