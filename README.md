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

Result:
```
Performing soft recovery...
Database recovery is successful.
```

### Step 2: Verify Recovery - Integrity Check Again

```cmd
esentutl /G C:\Windows\NTDS\ntds.dit
```

Result:
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

### Key Takeaway
> Don't do `esentutl /P` (hard repair) unless you have no other DC. Soft recovery + replication from healthy DC is the safest way.

Author: Riyad | Junior SysAdmin | Baku
