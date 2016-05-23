
## Export Pluggable Database Keys, Import to Clone into 12c Cloud Container ##

Simple 101 Example on how to export and import Oracle TDE keys.

This example is based around the requirement of cloning a Pluggable Database that is TDE encrypted and opening it in the _same container database_ as the source pluggable database that it was cloned from.

If the pluggable database is moved to a different container database, the key in the Oracle Wallet will need to be copied from the source container and merged into the target container database.  This is not covered in this example. 

### Prerequisites ###
+ Database Clone Operation is assumed to have already been done.
 	
+ This example assumes the old database was PDB1 and the new clone is NEW_PDB
	
+ We need to know the password for the Key Store and the DBA Password
  - these are both "welcome1" in this example

+ NOTE: the keystore "secret" and the DBA password could be different - they are both "welcome1" in this example.

### Export The Key from Source Pluggable Database, PDB1 ###

View the status of the Key-Store:
``` {SQL}
-- Make sure we are in the Root Container
ALTER SESSION SET CONTAINER = CDB$ROOT;
-- View the Key-Store configuration and status
SELECT STATUS, WALLET_TYPE FROM v$encryption_wallet;
```

EG

```
STATUS WALLET_TYPE  
------ ------------ 
OPEN   AUTOLOGIN	
```

If using the **WALLET_TYPE = AUTOLOGIN** Key Store, close it and then re-open using syntax below:

``` {SQL}
-- Make sure we are in the Root Container
ALTER SESSION SET CONTAINER = CDB$ROOT;
-- Close the root key-store
ADMINISTER KEY MANAGEMENT SET KEYSTORE CLOSE;
```

(As the key-store is in Auto-Login state, don't supply a password to close it)

ELSE, If using **WALLET_TYPE=PASSWORD** key store, close it using syntax below:

``` {SQL}
-- Make sure we are in the Root Container
ALTER SESSION SET CONTAINER = CDB$ROOT;
-- Close the root key-store
ADMINISTER KEY MANAGEMENT SET KEYSTORE CLOSE IDENTIFIED BY "welcome1";
```

Replace the key-store password used in the example as appropriate - this was set up when the key-store was created during Cloud VM Provisioning.


Now, **Open** the **Container Root Keystore** as **WALLET_TYPE=PASSWORD**
``` {SQL}
ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN IDENTIFIED BY "welcome1";
-- View the Key-Store configuration and status
SELECT STATUS, WALLET_TYPE FROM v$encryption_wallet;
```

EG

```
STATUS                         WALLET_TYPE
------------------------------ --------------------
OPEN                           PASSWORD

```

Next, **Open** the **SOURCE Pluggable Database Keystore**:

``` {SQL}
--Switch to Source Container 
ALTER SESSION SET CONTAINER = PDB1;
-- Open the Key-Store
ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN IDENTIFIED BY "welcome1";
```

Replace the key-store password used in the example as appropriate - this was set up when the key-store was created during Cloud VM Provisioning.

Finally, **Export the Encryption Keys** from the **SOURCE Pluggable Database**

``` {SQL}
ADMINISTER KEY MANAGEMENT EXPORT ENCRYPTION KEYS WITH SECRET welcome1
TO '/home/oracle/export_keys.exp' IDENTIFIED BY welcome1;
```

### Import The Key to Target Pluggable Database, NEW_PDB ###

First, check the state of the Wallet Key-Store

``` {SQL}
-- Make sure we are in the Root Container
ALTER SESSION SET CONTAINER = CDB$ROOT;
-- View the Key-Store configuration and status
SELECT STATUS, WALLET_TYPE FROM v$encryption_wallet;
```

Correct output is 
```
STATUS                         WALLET_TYPE
------------------------------ --------------------
OPEN                           PASSWORD
```
If WALLET_TYPE=AUTO-LOGIN, close and then re-open as a PASSWORD authenticated Wallet (see earlier example in the Export section).


``` {SQL}
--Switch to Target Container 
ALTER SESSION SET CONTAINER = NEW_PDB;
-- View the Key-Store configuration and status
SELECT STATUS, WALLET_TYPE FROM v$encryption_wallet;
```

Example output:
```
STATUS                         WALLET_TYPE
------------------------------ --------------------
CLOSED                         UNKNOWN
```

Next, **Open the Key Store** so that both the Container and Target PDB Wallets are open as PASSWORD-authenticated key-stores:

``` {SQL}
ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN IDENTIFIED BY "welcome1";
```

Then **Import the Wallet Key** to the Target PDB:

``` {SQL}
ADMINISTER KEY MANAGEMENT IMPORT KEYS WITH SECRET welcome1
FROM '/home/oracle/export_keys.exp' IDENTIFIED BY welcome1 WITH BACKUP;
```

Finally, switch to the root container and **Open the Target PDB**:

``` {SQL}
-- Make sure we are in the Root Container
ALTER SESSION SET CONTAINER = CDB$ROOT;
-- Close PLuggable Database and then Open  Read-Write
ALTER PLUGGABLE DATABASE NEW_PDB CLOSE IMMEDIATE;
ALTER PLUGGABLE DATABASE NEW_PDB OPEN READ WRITE;
```

Check the status of all the Pluggable Databases in the Container:

``` {SQL}
-- Make sure we are in the Root Container
ALTER SESSION SET CONTAINER = CDB$ROOT;
SELECT d.con_id,v.Name, v.Open_Mode, Nvl(v.Restricted, 'n/a') "Restricted", d.Status
FROM v$pdbs v , dba_pdbs d
WHERE v.GUID = d.GUID
ORDER BY v.Create_SCN;
```

View any Pluggable Database Violations:

``` {SQL}
set linesize 200
SELECT time || ' ' || name || ' ' || message || ' ' || action
FROM pdb_plug_in_violations
ORDER BY time;
```