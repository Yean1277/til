## Resolution:
1. Run Powershell as *administrator* and Enter
```
Set-SmbClientConfiguration -EnableInsecureGuestLogons $true -Force 
Set-SmbClientConfiguration -RequireSecuritySignature $false -Force 
Set-SmbServerConfiguration -RequireSecuritySignature $false -Force
```

## Environment
Windows version: Windows 10/Windows 10 Pro
