# Application Data Management and Administration

## Introduction

Secure Shell (SSH) is a cryptographic network protocol for operating network services securely over the Internet. This access requires the full path to the file that contains the private key associated with the public key used when the DB system was launched.

Oracle Enterprise Manager Database Express, also referred to as EM Express, is a web-based tool for managing Oracle Database 19c. Built inside the database server, it offers support for basic administrative tasks.

The multitenant architecture enables an Oracle database to function as a multitenant container database (CDB). A CDB includes zero, one, or many customer-created pluggable databases (PDBs). A PDB is a portable collection of schemas, schema objects, and nonschema objects.

The sample database schemas provide a common platform for examples in each release of the Oracle Database. The sample schemas are a set of interlinked database schemas. The Oracle Database sample schemas are based on a fictitious sample company that sells goods through various channels. The company operates worldwide to fill orders for products. It has several divisions, each of which is represented by a sample database schema. Schema Human Resources (HR) represents Division Human Resources and tracks information about the company employees and facilities. Schema Sales History (SH) represents Division Sales and tracks business statistics to facilitate business decisions.

This lab explains how to connect to an active DB system with SSH, enable EM Express, create a new PDB in the existing CDB, and install HR sample schema.

## Step 1: Database Node SSH Connection

After provisioning the DB System, Database State will be Backup In Progress... for a few minutes. Click **Nodes** on the left menu, and copy Public IP Address in your notes.

Connect to the Database node using SSH.

````
ssh -C -i id_rsa opc@<DB Node Public IP Address>
````

All Oracle software components are installed with **oracle** OS user. Use the substitute user command to start a session as **oracle** user.

````
sudo su - oracle
````

You can verify the services provided by the Oracle Listener.

````
lsnrctl status
````

You can connect to the database instance specified by environment variables.

````
sqlplus / as sysdba
````

List all pluggable databases.

````
show pdbs
````

Type **exit** command tree times followed by Enter to close all sessions (SQL*Plus, oracle user, and SSH).

````
exit

exit

exit
````

## Step 2: Enterprise Manager Express 

Connect to the Database node using SSH, this time adding an SSH tunneling or SSH port forwarding option for port 5500.

````
ssh -C -i id_rsa -L 5500:localhost:5500 opc@<DB Node Public IP Address>
````

Use the substitute user command to start a session as **oracle** user.

````
sudo su - oracle
````

Connect to the database instance specified by environment variables.

````
sqlplus / as sysdba
````

Unlock **xdb** database user account.

````
alter user xdb account unlock;
````

Enable Oracle Enterprise Manager Database Express (EM Express) clients to use a single port (called a global port), for the session rather than using a port dedicated to the PDB.

````
exec dbms_xdb_config.SetGlobalPortEnabled(TRUE);
````

Open the web browser on your computer, and navigate to **https://localhost:5500/em**.

Use the following credentials:

- Username: system
- Password: DBlearnPTS#20_
- Container Name: CDB$ROOT for the Container Database, or PDB011 for the Pluggable Database. Try both.

Explore Enterprise Manager Express console, and see what this tool has to offer.

## Step 3: Create a Pluggable Database

Connect to the Compute node using SSH.

````
ssh -C -i id_rsa opc@<Compute Public IP Address>
````

Connect to your DB System database using SQL*Plus.

````
sqlplus sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/<Database Unique Name>.<Host Domain Name> as sysdba
````

List pluggable databases.

````
show pdbs
````

Create a new pluggable database called **PDB012**.

````
CREATE PLUGGABLE DATABASE pdb012 ADMIN USER Admin IDENTIFIED BY DBlearnPTS#20_;
````

List pluggable databases and confirm the new pluggable database is there.

````
show pdbs
````

Change the state of the new pluggable database PDB012 to **OPEN**.

````
ALTER PLUGGABLE DATABASE pdb012 OPEN;
````

List pluggable databases and confirm it is **OPEN**.

````
show pdbs
````

Connect to the new pluggable database.

````
conn sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name> as sysdba
````

Display the current container name.

````
show con_name
````

List all users in PDB012.

````
select username from all_users order by 1;
````

This pluggable database doesn't have Oracle Sample Schemas either. Exit SQL*Plus.

````
exit
````

## Step 4: Install HR Sample Schema

Use the same Compute node SSH connection. Connect to the new pluggable database PDB012.

````
sqlplus sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name> as sysdba
````

List all tablespaces.

````
select name from v$tablespace;
````

List all datafiles.

````
select name from v$datafile;
````

For the Oracle Sample Schemas, we need a tablespace called **USERS**. Try to create it.

````
CREATE TABLESPACE users;
ORA-28361: master key not yet set
````

We get an error about a Master Key. To use Oracle Transparent Data Encryption (TDE) in a pluggable database (PDB), you must create and activate a master encryption key for the PDB.

In a multitenant environment, each PDB has its own master encryption key which is stored in a single keystore used by all containers.

Create and activate a master encryption key in the PDB by executing the following command:

````
ADMINISTER KEY MANAGEMENT SET KEY FORCE KEYSTORE IDENTIFIED BY DBlearnPTS#20_ WITH BACKUP;
````

Now we can create tablespace **USERS**.

````
CREATE TABLESPACE users;
````

List all tablespaces to confirm the new tablespace was created.

````
select name from v$tablespace;
````

List all datafiles and see the corresponding files.

````
select name from v$datafile;
````

Exit SQL*Plus.

````
exit
````

Download Oracle Sample Schemas installation package from GitHub.

````
wget https://github.com/oracle/db-sample-schemas/archive/v19c.zip
````

Unzip the archive.

````
unzip v19c.zip
````

Open the unzipped folder.

````
cd db-sample-schemas-19c
````

Run this Perl command to replace **__SUB__CWD__** tag in all scripts with your current working directory, so all embedded paths to match your working directory path.

````
perl -p -i.bak -e 's#__SUB__CWD__#'$(pwd)'#g' *.sql */*.sql */*.dat
````

Go back to the parent folder (this should be /home/opc).

````
cd ..
````

Create a new folder for logs.

````
mkdir logs
````

Connect to the **PDB012** pluggable database.

````
sqlplus sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name> as sysdba
````

Run the HR schema installation script. For more information about [Oracle Database Sample Schemas](https://github.com/oracle/db-sample-schemas) installation process, please follow the link. Make sure to replace **DB Node Private IP Address** and **Host Domain Name** with the actual values.

````
@db-sample-schemas-19c/human_resources/hr_main.sql DBlearnPTS#20_ USERS TEMP DBlearnPTS#20_ /home/opc/logs/ <DB Node Private IP Address>:1521/pdb012.<Host Domain Name>
````

## Step 5: Verify HR Sample Schema

Display current user. If all steps were followed, the current user should be **HR**.

````
show user
````

Select tables from **HR** schema and their row numbers.

````
select TABLE_NAME, NUM_ROWS from USER_TABLES;
````

Some SQL*Plus formatting.

````
set linesize 130
````

Select all rows from the **HR.EMPLOYEES** table.

````
select * from EMPLOYEES;
````

If everything is fine, exit SQL*Plus.

````
exit
````

## Step 6: Install SH Sample Schema

Connect to the **PDB012** pluggable database.

````
sqlplus sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name> as sysdba
````

Run the SH schema installation script. Make sure to replace **DB Node Private IP Address** and **Host Domain Name** with the actual values.

````
@db-sample-schemas-19c/sales_history/sh_main.sql DBlearnPTS#20_ USERS TEMP DBlearnPTS#20_ /home/opc/db-sample-schemas-19c/sales_history/ /home/opc/logs/ v3 <DB Node Private IP Address>:1521/pdb012.<Host Domain Name>
````

Display current user. If all steps were followed, the current user should be **SH**.

````
show user
````

Select tables from **SH** schema and their row numbers.

````
select TABLE_NAME, NUM_ROWS from USER_TABLES;
````

If everything is fine, exit SQL*Plus.

````
exit
````

## Acknowledgements

- **Author** - Valentin Leonard Tabacaru
- **Last Updated By/Date** - Valentin Leonard Tabacaru, Principal Product Manager, DB Product Management, Aug 2020

See an issue? Please open up a request [here](https://github.com/oracle/learning-library/issues). Please include the workshop name and lab in your request.

