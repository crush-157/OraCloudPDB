# Quick Ref Guide for Common PDB Operations #

A great starting point and reference: [Tim Halls Introduction to Oracle Multitenant Database](https://oracle-base.com/articles/12c/multitenant-overview-container-database-cdb-12cr1).  Includes a summary of naming convention for data-dictionary views and overview of architecture, strategy. 

**Summary of View Hierarchy**

```
CDB_<viewname>                 (view across all PDBs)
 |
 -- DBA_<viewname>             (view across all objects in database)
     |
     -- ALL_<viewname>         (view of all objects the user can access)
         |
         -- USER_<viewname>    (view of all objects in the user schema)
```

### Container Database and PDBs Summary ###
#### What is the CDB name, what is in it, what state are the PDBs in ####
```
SQL> select name, cdb,con_id from v$database;

NAME      CDB     CON_ID
--------- --- ----------
MYDB      YES          0

```
This is the name of the database in which the pluggable databases reside.

Datafile default locations if using Oracle Managed Files:

```
show parameter db_create_file_dest
 db_create_file_dest                                string      +DATA

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_create_file_dest                  string      /u02/app/oracle/oradata
```


List the pluggable databases in the container:
```
SQL> COLUMN NAME FORMAT A8

SELECT NAME, CON_ID, DBID, CON_UID, GUID 
FROM V$CONTAINERS ORDER BY CON_ID;

NAME         CON_ID       DBID    CON_UID GUID
-------- ---------- ---------- ---------- --------------------------------
CDB$ROOT          1 4040439200          1 FD9AC20F64D344D7E043B6A9E80A2F2F
PDB$SEED          2 3487591558 3487591558 04CAC5E84017187BE053BE3A6A0A21B0
PDB1              3  722031645  722031645 2A3C900B91F21E33E0534E32C40A91D7
SWB1              4 1714094753 2982733209 29D84FC1094D1029E0533634C40AA650
```

Show status of  Pluggable Databases:

``` {SQL}
 SELECT d.con_id,v.Name, v.Open_Mode, Nvl(v.Restricted, 'n/a') "Restricted", d.Status
 FROM v$pdbs v , dba_pdbs d
 WHERE v.GUID = d.GUID
 ORDER BY v.Create_SCN;
```

### Switch Containers, Show Container Name ###

Switch context so session is in a specific PDB:

```
SQL> alter session set container = PDB1;

Session altered.
```

Show current session PDB:
```
SQL> show con_name;

CON_NAME
------------------------------
PDB1
SQL> show con_id

CON_ID
------------------------------
3
```

Switch back to root
```
SQL> alter session set container = cdb$ROOT;

Session altered.

SQL> show con_name;

CON_NAME
------------------------------
CDB$ROOT
```

### DataFiles and Users ###

List DataFiles for each Pluggable Database.

This shows breakdown of data-files for all pluggables, but excludes the Container Database.  Run this from the root container context.
```{SQL}
SELECT p.pdb_name, round(f.bytes/1024/1024) MBYTES, f.tablespace_name, f.file_name 
FROM cdb_data_files f, dba_pdbs p
WHERE f.con_id = p.pdb_id
ORDER by 1;
```

List Users by Pluggable Database.

This shows Common and Local users as well as usual ALL_USERS details.

```{SQL}
SELECT p.pdb_name, u.username, u.common, u.account_status, u.default_tablespace, u.temporary_tablespace
FROM cdb_users u, dba_pdbs p
WHERE u.con_id = p.pdb_id
ORDER by 1,2;
```

### Start / Stop PDB ###

Obviously the CDB needs to be running.  Connect as SYSDBA to CDB 
```
$> sqlplus "/ as sysdba"
```
in order to run the PDB start/stop/maintenance commands listed below.

**Start** Pluggable Database Examples:

```
alter pluggable database PDB1 open READ WRITE [RESTRICTED] [FORCE];
alter pluggable database PDB1 open READ ONLY [RESTRICTED] [FORCE];
```

**Stop** Pluggable Database

```
alter pluggable database PDB1 close immediate;
```


**Upgrade** Start Pluggable Database for Patch Maintenance

```
alter pluggable database PDB1 open UPGRADE [RESTRICTED];
```

### Basic Connectivity ###

When the PDB is opened it is automatically registered with the listener - EG:

```
$ lsnrctl service

LSNRCTL for Linux: Version 12.1.0.2.0 - Production on 24-MAR-2016 11:15:27

Copyright (c) 1991, 2014, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=myhostnamne.oraclecloud.internal)(PORT=1521)))
Services Summary...
Service "ORCL.myclouddomain.oraclecloud.internal" has 1 instance(s).
  Instance "ORCL", status READY, has 1 handler(s) for this service...
    Handler(s):
      "DEDICATED" established:1 refused:0 state:ready
         LOCAL SERVER
Service "pdb1.myclouddomain.oraclecloud.internal" has 1 instance(s).
  Instance "ORCL", status READY, has 1 handler(s) for this service...
    Handler(s):
      "DEDICATED" established:1 refused:0 state:ready
         LOCAL SERVER

```

The example above shows the format used in the Oracle Cloud.

To connect, create an appropriate TNS entry 

```
$> cat $ORACLE_HOME/network/admin/tnsnames.ora

ORCL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = myhostnamne.oraclecloud.internal)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ORCL.myclouddomain.oraclecloud.internal)
    )
  )

PDB1 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = myhostnamne.oraclecloud.internal)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = PDB1.myclouddomain.oraclecloud.internal)
    )
  )

```
and connect using SQL*Net:

```
$ sqlplus scott/tiger@PDB1
```


### Patching a Pluggable Database ###

**Patch**
+ Apply patches to the CDB using standard ```opatch```
+ Shutdown PDBs and startup in UPGRADE mode
+ run ```datapatch -verbose``` to bring the PDBs up to CDB patch level.

**Backout**
+ Shutdown PDB and startup in UPGRADE mode
+ run ```datapatch -rollback <patchid> -pdbs <list of PDBs>``` EG:

``` $ datapatch -rollback 21962590 -pdbs SWB```

*NOTE*: The appropriate
```
$ORACLE_HOME/sqlpatch/<patchid>
```

directory needs to be in place locally to perform the rollback.  Copy this directory across from the server where the PDB was patched, if necessary

Ref and further detail:
https://blogs.oracle.com/UPGRADE/entry/applying_a_psu_or_bp

### Troublshooting and Errors ###

Trace, Alerts: The alert log for all PDBs in the common CDB alert file, EG:

```
/u01/app/oracle/diag/rdbms/orcl/ORCL/trace/alert_ORCL.log
```

To view Plug-In Errors / Violations, use the ```PDB_PLUG_IN_VIOLATIONS``` view,  EG:

```{SQL}
set linesize 200
SELECT time || ' ' || name || ' ' || message || ' ' || action
FROM pdb_plug_in_violations
ORDER BY time;

```

### Create a PDB - Clone from Seed Database ###

In this example, create a new PDB called `SWB1`:

```{SQL}
CREATE PLUGGABLE DATABASE swb1 admin user swb1admin identified by welcome1
roles = (DBA)
FILE_NAME_CONVERT = (
    '/u02/app/oracle/oradata/ORCL/pdbseed',
    '/u02/app/oracle/oradata/ORCL/SWB1',
    '/u04/app/oracle/oradata/temp/pdbseed_temp012016-09-12_07-35-13-AM.dbf',
    '/u02//app/oracle/oradata/ORCL/SWB1/temp01.dbf')
```

Then open the PDB database:

```
alter pluggable database SWB1 open READ WRITE
```

For TDE Encrypted environments, you will get the following error when you try to create a new tablespace:

```The following statement failed : create tablespace ... ORA-28374: typed master key not found in wallet```

** Extra Step Required for TDE Encrypted Databases **

```
alter session set container=cdb$root;

administer key management set keystore close;

administer key management set keystore open identified by welcome1 container=all;

alter session set container=SWB1;

ADMINISTER KEY MANAGEMENT SET KEY USING TAG  "tde_dbaas" identified by welcome1   
WITH BACKUP USING "tde_dbaas_bkup";

set linesize 120
column WRL_PARAMETER format a40
SELECT WRL_PARAMETER,STATUS,WALLET_TYPE FROM V$ENCRYPTION_WALLET;


WRL_PARAMETER                            STATUS                         WALLET_TYPE
---------------------------------------- ------------------------------ --------------------
/u01/app/oracle/admin/CSDEMO/tde_wallet/ OPEN                           PASSWORD

```

Re-Try the Create Tablespace command - it should now work.
