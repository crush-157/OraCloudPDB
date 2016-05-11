# Quick Ref Guide for Common PDB Clone Operations #

## Local Clone of the Seed PDB to Create a Shell PDB ##

To view the Seed Database Data-Files:

```{SQL}
ALTER SESSION set exclude_seed_cdb_view = false;

SELECT file_name FROM cdb_data_files
WHERE con_id = 2
UNION
SELECT file_name 
FROM cdb_temp_files
WHERE con_id = 2;
```

Specify the File Name Convert option to determine the location for the new PDB when creating (File Name Convert is only necessary if not using Oracle Managed Files):

```{SQL}
CREATE PLUGGABLE DATABASE pdb2 
ADMIN USER pdb_adm IDENTIFIED BY welcome1
FILE_NAME_CONVERT=(  
'/u02/app/oracle/oradata/ORCL/pdbseed/system01.dbf',  
'/u02/app/oracle/oradata/ORCL/pdb2/system01.dbf',  
'/u02/app/oracle/oradata/ORCL/pdbseed/sysaux01.dbf',  
'/u02/app/oracle/oradata/ORCL/pdb2/sysaux01.dbf',  
'/u02/app/oracle/oradata/ORCL/29D627273C13166CE053D64EC40AC9E5/datafile/o1_mf_users_cl48t4c4_.dbf',  
'/u02/app/oracle/oradata/ORCL/pdb2/users01.dbf',  
'/u02/app/oracle/oradata/ORCL/pdbseed/pdbseed_temp012016-01-21_09-59-05-AM.dbf',  
'/u02/app/oracle/oradata/ORCL/pdb2/temp01.dbf');  


ALTER PLUGGABLE DATABASE pdb2 open READ WRITE;

```

This copies the files into place.  The database should register with the listener automatically and be available as a TNS Service to connect to.

To check the service name registered for this database, issue the following query:  

```
SELECT name 
FROM v$services
WHERE pdb = 'PDB2';

NAME
----------------------------------------------------------------
pdb2.gse00000379.oraclecloud.internal


```
Note the PDB name identifier has to be supplied in upper-case.

A TNS client entry can then be created to connect to this container database - an example is shown below:

```
ORCL.PDB2 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.101.1)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = pdb2.gse00000379.oraclecloud.internal)
    )
  )

```

Common Users from the CDB will be available in this database.

If **Apex (Oracle Application Express) is installed** in the cdb$ROOT then this will get copied to the PDB.  **See the following blog entry**:
https://blogs.oracle.com/UPGRADE/entry/apex_in_pdb_does_not


## Drop a Pluggable Database ##

```{SQL}
ALTER PLUGGABLE DATABASE pdb2 close;
DROP PLUGGABLE DATABASE pdb2 including datafiles;
```


## UnPlug a PDB, Copy and Clone to Remote CDB ##

In 12.1.0.2 this is an off-line operation.

Either close the source PDB first and do an offline copy (first example) or set Read-Only and copy across a database link (second example):
**Source Database**
```{SQL}
ALTER PLUGGABLE DATABASE pdb1 close immediate;
```
Un-Plug the database and create an XML manifest file:
**Source Database**
```{SQL}
ALTER PLUGGABLE DATABASE pdb1 unplug into 'PDB1.XML'
```
Then, package up the data-files and the XML manifest:
**Source Database**
```
cd /oradata/ORCL/pdb1
cp $ORACLE_HOME/dbs/PDB1.XML /oradata/ORCL/pdb1
# tar up data-files and XML manifest
tar -cvf ./pdb.tar ./*
gzip pdb.tar

```

Plug the database back in - *you have to drop the pluggable database before you can plug it back in*:
**Source Database**
```{SQL}
DROP PLUGGABLE DATABASE pdb1 keep datafiles;

CREATE PLUGGABLE DATABASE pdb1 using 'PDB1.XML'
source_file_name_convert = NONE
nocopy
storage unlimited tempfile reuse;

```
Open the Source Pluggable Database for general use:
**Source Database**
```{SQL}
ALTER PLUGGABLE DATABASE pdb1 open read write;

```

Copy the data-files and XML manifest to the remote CDB (this example is a Oracle Cloud instance):

**Source Database Server**
```
scp  pdb.tar.gz oracle@140.86.99.99:/u02/app/oracle/oradata/ORCL/PDB1
``` 

############################################################
####  Special Oracle Cloud Apex Requirements ####
Based on this blog:
https://blogs.oracle.com/UPGRADE/entry/clean_up_apex_journey_to
- remove all Apex installations from the cloud database.  Apex can be re-installed on a PDB by PDB basis.
- This will break the Oracle CLoud "DBaaS Monitor" which is driven off Apex.

**On the Cloud Remote Destination Database**
+ Download Apex APEX 4.2.0.6 and PDBSS 2.0 (Multitenant Self Service Provisioning Application)
+ Remove Apex from the Cloud CDB
+ Drop DBaaS Monitor common users

.1. Download APEX 4.2.6 and PDBSS 2.0
http://www.oracle.com/technetwork/developer-tools/apex/application-express/apex-archive-42-1885734.html

http://www.oracle.com/technetwork/database/multitenant/downloads/multitenant-pdbss-2016324.html

.2. Copy the Downloaded zip files to the Cloud environment

.3. Connect to your Cloud environment and unzip both files

.4. Remove PDBSS by using the unzipped archive that was copied across
```
sqlplus / as sysdba
@pdbss_remove.sql
```

.5. Remove APEX 5 by using the existing default installation
```
cd $ORACLE_HOME/apex
sqlplus / as sysdba
SQL> @apxremov.sql
``` 

.6. Remove any APEX 4.2 parts by using the unzipped archive that was copied across
```
cd /home/oracle/apex
sqlplus / as sysdba
SQL> @apxremov_con.sql
```

.7. Drop the DBaaS Monitor common users
```
sqlplus / as sysdba
SQL> drop user C##DBAAS_BACKUP cascade;
SQL> drop user C##DBAAS_MONITOR cascade;
```

.8. Recompile and Check for Invalid Objects
```
sqlplus / as sysdba
SQL> @?/rdbms/admin/utlrp.sql
SQL> select object_name, owner from dba_objects where status='INVALID';
```

############################################################

#### Drop Apex User Fails - Workaround ####

If the following error is encountered when running the Apex Removal scripts:
**ORA-28014: cannot drop administrative users**
then use this procedure:
```
SQL> alter session set "_oracle_script"=true;
Session altered.

SQL> drop user APEX_040200 cascade;
User dropped.

SQL> alter session set container = PDB1;
Session altered.

SQL> alter session set "_oracle_script"=true;
Session altered.

SQL> drop user APEX_040200 cascade;
User dropped.
```



############################################################

#### Switch to working on the remote Destination Server ####

Extract the datafiles and XML manifest:
**Remote Destination Database**
```
cd /u02/app/oracle/oradata/ORCL/PDB1
gunzip pdb.tar.gz
tar -xvf pdb.tar
```

View the data-file paths in the XML manifest:
**Remote Destination Database**
```
 grep '<path>' PDB1.XML

      <path>/oradata/ORCL/pdb/system01.dbf</path>
      <path>/oradata/ORCL/pdb/sysaux01.dbf</path>
      <path>/oradata/ORCL/pdb/pdbseed_temp012016-03-21_07-39-37-PM.dbf</path>

```

Fix the data-file paths using sed (use # separator for sed, so that directory / chars can be included in search-and-replace):
**Remote Destination Database**
```
cat PDB1.XML | sed 's#/oradata/ORCL/pdb#/u02/app/oracle/oradata/ORCL/PDB#' > $ORACLE_HOME/dbs/PDB1_DEST.XML
```
Use grep on new XML manifest "PDB1_DEST.XML" to check data-paths.


**Remote Destination Database**
Create pluggable database from the data-files and manifest:

```{SQL}
CREATE PLUGGABLE DATABASE pdb1 using 'PDB1_DEST.XML'
source_file_name_convert = NONE
nocopy
storage unlimited tempfile reuse;

```

Open the PDB for use:
**Remote Destination Database**
```{SQL}
ALTER PLUGGABLE DATABASE pdb1 open read write;

```

### Dealing With Import Errors ###

Usually the first attempt to open the PDB will result in:
```
Warning: PDB altered with errors.
```
and it is necessary to work through the errors listed in PDB_PLUG_IN_VIOLATIONS view.


```{SQL}
set linesize 200
SELECT time || ' ' || name || ' ' || message || ' ' || action
FROM pdb_plug_in_violations
ORDER BY time;
```

Typically issues fall into these categories:

+ Missing Tablespaces for CDB common users - create the tablespace in the Source database before copying OR alter the Destination database common users to have a different default tablespace
+ Apex violations - drop Apex from the CDB in the Destination database and make sure it is dropped from the Source PDB to be plugged into the CDB.  Remove from the Source PDB by getting rid of Apex in the Source CDB *before* creating the PDB
+ Patch violations - use datapatch to resolve.  Patching issues fall into two categories: patches present in CDB but not PDB and patches in PDB that are not in CDB.
+ Character Set Mismatch: EG "PDB character set WE8MSWIN1252. CDB character set AL32UTF8.  Convert the character set of the PDB to match the CDB..."
https://docs.oracle.com/database/121/NLSPG/ch2charset.htm#NLSPG1035


In general patch violations where the patch is missing from the PDB being plugged in can be resolved as follows:
```
SQL> alter pluggable database pdb1 close immediate;
SQL> alter pluggable database pdb1 open upgrade;

$> datapatch -verbose

```
Patches in the PDB that are missing from CDB can be more complex to resolve, depending on the dependancies; attempt to patch the CDB up to match the PDB or use datapatch to rollback the PDB.

## Clone a PDB Across a Database Link ##

