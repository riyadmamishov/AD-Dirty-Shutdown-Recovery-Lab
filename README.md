# AD Disaster Recovery Lab - From Dirty Shutdown to Clean Replication

**Scenario:** PDC (WINSERV22) crashed hard. ntds.dit went into Dirty Shutdown. Domain still alive on ADC-SRV.

**Lab Environment:**
- PDC: WINSERV22 - Windows Server 2022 (10.0.20348.587)
- ADC: ADC-SRV
- Error: JET Database Dirty Shutdown

---
### Step 0: Diagnosis - Is the DB really corrupt?
Booted into DSRM and checked integrity:

```cmd
esentutl /G C:\Windows\NTDS\ntds.dit
```

 <img width="772" height="492" alt="ntds 1" src="https://github.com/user-attachments/assets/a224aff9-3397-4d90-833e-cb9122467a68" />
```

> The database is not up-to-date. This operation may find that this database is corrupt because data from the log files has yet to be placed in the database.
> To ensure the database is up-to-date please use the 'Recovery' operation.

**Meaning:** DB is not physically corrupt, log files not flushed. It's Dirty Shutdown.

### Step 1: Soft Recovery with ntdsutil

```cmd
ntdsutil
activate instance ntds
files
recover
quit
quit
```

 <img width="407" height="231" alt="rec 2" src="https://github.com/user-attachments/assets/c6b1721c-5faf-4791-9ebb-f8f2d75b17f4" />

```
Performing soft recovery...
Database recovery is successful.
```

### Step 2: Verify Recovery - Integrity Check Again

```cmd
esentutl /G C:\Windows\NTDS\ntds.dit
```

 <img width="787" height="452" alt="dsr 3" src="https://github.com/user-attachments/assets/6ef68642-fed8-4d6d-af45-d3d1a35f2b42" />

```
Integrity check successful.
Operation completed successfully in 0.953 seconds.
```

Then boot back to normal mode:
```cmd
bcdedit /deletevalue safeboot
```

### Step 3: Post-Recovery - Critical Replication (Avoid USN Rollback)

PDC USN is old. ADC has newer data. Correct way: Replicate ADC -> PDC

On ADC: Active Directory Sites and Services -> WINSERV22 -> NTDS Settings -> Replicate Now

Verification:
```powershell
repadmin /showrepl
repadmin /syncall /AdeP
```
<img width="942" height="476" alt="rep 4" src="https://github.com/user-attachments/assets/b8190e2c-a98a-4304-82e2-5af6f5c7f6f0" />

### Key Takeaway
> Don't do `esentutl /P` (hard repair) unless you have no other DC. Soft recovery + replication from healthy DC is the safest way.

Author: Riyad | Junior SysAdmin | Baku
