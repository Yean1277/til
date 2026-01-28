To troubleshoot the WMI (Windows Management Instrumentation) during MSSQL installation

Notes: Ensure the WMI services is open and run automatically

1. Open CMD as admin
type `winmgmt /verifyrepository`

Result meaning:
- WMI repository is consistent → Step 3
- Repository is inconsistent →  Step 2

2. Repair WMI type `winmgmt /salvagerepository`
3. Re-register WMI components
```
  cd %windir%\system32\wbem
  for %i in (*.dll) do regsvr32 /s %i
  for %i in (*.exe) do %i /RegServer
```

4. Restart the PC

