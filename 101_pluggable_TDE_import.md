
## Plug Swingbench Pluggable Database into 12c Cloud Container ##

Simple 101 Example based around "SWB1" pluggable database.  This is a TDE encrypted database (TDE encryption is enabled by default for Oracle Cloud databases)

### Prerequisites ###
+ Copy Datafiles into place
 	- Sample Swingbench Database "SWB1" used in this guide
+ Export Keys from Source Database - need the Pluggable Database Keys not just CDB keys
	- Copy the wallet file from the source if plugging into another container
+ Need a prepared XML pluggable database manifest
	- this is pre-prepared for the example in this guide

### Copy Files in Place ###

scp etc...

Example - changing the data-file path in the XML manifest:
```
cat SWB1.XML | sed 's_/u02/app/oracle/oradata/SWB1_/u01/app/oracle/oradata/MYDB1_' > SWB1b.XML
```
(change the path as appropriate for different directory-path destinations)


### Plug Database In ###
``` {SQL}
CREATE PLUGGABLE DATABASE "SWB1" USING '/home/oracle/SWB1.XML'
SOURCE_FILE_NAME_CONVERT = NONE
NOCOPY
STORAGE UNLIMITED TEMPFILE REUSE;
```


### Close then Open the Key-Store ###

View the status of the Key-Store:
``` {SQL}
-- Make sure we are in the Root Container
ALTER SESSION SET CONTAINER = CDB$ROOT;
-- View the Key-Store configuration and status
SELECT * FROM v$encryption_wallet;
```

EG

```
WRL_TYPE WRL_PARAMETER                          STATUS WALLET_TYPE  WALLET_OR FULLY_BAC CON_ID
-------- -------------------------------------- ------ ------------ --------- --------- ------
FILE	/u01/app/oracle/admin/EDBC/tde_wallet/	OPEN   AUTOLOGIN	SINGLE	  NO	    0
```

If using the **WALLET_TYPE = AUTOLOGIN** Key Store, close it and then re-open using syntax below:

``` {SQL}
-- Make sure we are in the Root Container
ALTER SESSION SET CONTAINER = CDB$ROOT;
-- Close the root key-store
ADMINISTER KEY MANAGEMENT SET KEYSTORE CLOSE;
```

ELSE, If using **WALLET_TYPE=PASSWORD** key store, close it using syntax below:

``` {SQL}
-- Make sure we are in the Root Container
ALTER SESSION SET CONTAINER = CDB$ROOT;
-- Close the root key-store
ADMINISTER KEY MANAGEMENT SET KEYSTORE CLOSE IDENTIFIED BY "Welc#me1";
```
Replace the key-store password used in the example as appropriate - this was set up when the key-store was created during Cloud VM Provisioning.


Now, **Open** the **Container Root Keystore** as **WALLET_TYPE=PASSWORD**
``` {SQL}
ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN IDENTIFIED BY "Welc#me1";
SELECT * FROM v$encryption_wallet;
```

EG

```
WRL_TYPE WRL_PARAMETER                          STATUS WALLET_TYPE  WALLET_OR FULLY_BAC CON_ID
-------- -------------------------------------- ------ ------------ --------- --------- ------
FILE	/u01/app/oracle/admin/EDBC/tde_wallet/	OPEN   PASSWORD	SINGLE	  NO	    0
```

Next, **Open** the **Pluggable Database Keystore**:

```
--Switch to Container 
ALTER SESSION SET CONTAINER = SWB1;
-- Open the Key-Store
ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN IDENTIFIED BY "Welc#me1";
```

Replace the key-store password used in the example as appropriate - this was set up when the key-store was created during Cloud VM Provisioning.


### Open Pluggable Database ###

``` {SQL}
ALTER SESSION SET CONTAINER = cdb$ROOT;
ALTER PLUGGABLE DATABASE SWB1 OPEN;
```
**NOTE** Pluggable database opens with errors (as keys are missing) - this is expected.

### Import Keys to Pluggable Database ###

``` {SQL}
--Switch to Container 
ALTER SESSION SET CONTAINER = SWB1;

ADMINISTER KEY MANAGEMENT IMPORT KEYS WITH SECRET "Welc#me1"
FROM '/home/oracle/export_keys_swb1.exp' IDENTIFIED BY Welc#me1 WITH BACKUP;
```
The "*secret*" password associated with the key-dump was set when the "export_keys_swb1.exp" key export was created.


### Merge the Master Keys into the Target Container Database ###

The Master Keys are stored outside the database in the wallet.  In this example they have been copied into 

```
/home/oracle/wallet_merge
```
by unpacking the tar file `master_wallet.tar` in `/home/oracle`

Check the status of the Key-store and note the Location:
``` {SQL}
 SELECT * FROM v$encryption_wallet;
```
If the key-store is wallet-type **AUTOLOGIN** then use the following syntax to merge the source Master-Keys into this key store:

``` {SQL}
ADMINISTER KEY MANAGEMENT MERGE KEYSTORE '/home/oracle/wallet_merge'
INTO EXISTING KEYSTORE  '/u01/app/oracle/admin/TARGETDB/tde_wallet/'
IDENTIFIED BY Welc#me3 WITH BACKUP;
```

The path to the keystore location is taken from the output of `v$encryption_wallet`

*Otherwise*, If the key-store is wallet-type **PASSWORD** then use the following syntax to merge the source Master-Keys into this key store:

``` {SQL}
ADMINISTER KEY MANAGEMENT MERGE KEYSTORE '/home/oracle/wallet_merge'
IDENTIFIED BY Welc#me1
INTO EXISTING KEYSTORE  '/u01/app/oracle/admin/TARGETDB/tde_wallet/'
IDENTIFIED BY Welc#me3 WITH BACKUP;
```
The path to the keystore location is taken from the output of `v$encryption_wallet`



### Open the Pluggable Database ###

Shutdown and Re-Open the the Pluggable Database:
``` {SQL}
ALTER SESSION SET CONTAINER = CDB$ROOT;
ALTER PLUGGABLE DATABASE SWB1 CLOSE;
ALTER PLUGGABLE DATABASE SWB1 OPEN READ WRITE;
```

Check for errors:
``` {SQL}
SELECT * FROM pdb_plug_in_violations ORDER BY 1;
```

Close the Container Key-Store and Re-Open:
``` {SQL}
ADMINISTER KEY MANAGEMENT SET KEYSTORE CLOSE;
ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN IDENTIFIED BY "Welc#me3";
```

Open the Pluggable Database Keystore:

```
--Switch to Container 
ALTER SESSION SET CONTAINER = SWB1;
-- Open the Key-Store
ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN IDENTIFIED BY "Welc#me3";
```

### Connect to Swingbench and Test ###

+ Identify Service Name
In cloud SSH terminal session:
```
$ lsnrctl services | grep swb1
Service "swb1.metcsgse00453.oraclecloud.internal" has 1 instance(s).
```

+ Start an SSH Tunnel on port 1521


+ Start Swingbench in Windows Client
```
.\swingbench\winbin\swingbench.bat 
```

+ Set Swingbench Connect Details and Run

![](http://i.imgur.com/ga1cuV1.png)
 



 