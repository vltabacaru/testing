# Database Cloud Service Backup & Recovery

## Introduction

Backing up your database is a key aspect of any Oracle database environment. There are multiple options available for storing and recovering your backups. You can use the backup and restore feature either within the Oracle Cloud Infrastructure Console, CLI or REST APIs, or manually set up and manage backups using dbcli or RMAN. You can read about all the options in the [Oracle Cloud Infrastructure technical documentation](https://docs.us-phoenix-1.oraclecloud.com/Content/Database/Tasks/backingup.htm).

>**Note** : Your automatic incremental backups, on-demand full backups and on-premises to cloud backups are stored in Oracle Cloud Infrastructure Object Storage and you will be charged standard object storage cost.

When you use the Console, you can create full backups or set up automatic incremental backups with a few clicks. Similarly, you can view your backups and restore your database using the last known good state, a point-in-time, or SCN (System Change Number). You can also create a new database from your backup in an existing or a new DB system.

## Step 1: View Contents of Current Database Service

Connect to the Database node using SSH.

````
ssh -C -i id_rsa opc@<DB Node Public IP Address>
````

Use the substitute user command to start a session as **oracle** user, owner of Oracle software components.

````
sudo su - oracle
````

Connect to the database instance specified by environment variables.

````
sqlplus / as sysdba
````

List all pluggable databases.

````
show pdbs

    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			  READ ONLY  NO
	 3 PDB011			  READ WRITE NO
	 4 PDB012			  READ WRITE NO
````

There are three pluggable databases now, one seed PDB (system-supplied template that the CDB can use to create new PDBs) and two user-created PDBs (application data). Exit SQL*Plus.

````
exit
````

## Step 2: Create a Full Database Backup

On Oracle cloud console, click on hamburger menu ≡, then **Bare Metal, VM, and Exadata** under Databases. Click **[Your Initials]-DB** DB System.

Click the database name link **[Your Initials]DB** in the bottom table called Databases.

Review the backup called **Automatic Backup** in the bottom table called Backups. Click **Create Backup** button. Call it Manual-Backup, and click **Create Backup** to confirm. The new backup is added to the Backups table, having the State: Creating...

Access Work Requests table, and click **Create Database Backup**. Review all Resources: Log Messages (2), Error Messages (0), Associated Resources (2). Wait until this work request is 100% Complete. Under Associated Resources, click **DBS001** database name link.

At this point you can see the Manual-Backup on Backups table is now Active.

## Step 3: Restore Database Service from Backup

On Oracle cloud console, click on hamburger menu ≡, then **Bare Metal, VM, and Exadata** under Databases. Click **[Your Initials]-DB** DB System.

Click the database name link **[Your Initials]DB** in the bottom table called Databases.

Write down the Started and Ended times for backup called **Automatic Backup** in the bottom table called Backups - e.g. Started: 09:38:13 UTC, Ended: 09:56:36 UTC.

Up on Database Details page, click **Restore** button. Set field **Restore to the timestamp** to the next possible value after your Automatic Backup Ended field - e.g. 10:00 UTC. Click **Restore Database** to confirm.

Access Work Requests table, and click **Restore Database** having Status: In Progress... Review all Resources: Log Messages (2), Error Messages (0), Associated Resources (1). Wait until this work request is 100% Complete. Under Associated Resources, click **DBS001** database name link.

Connect again to the database instance specified by environment variables.

````
sqlplus / as sysdba
````

List one more time all pluggable databases.

````
show pdbs

    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			  READ ONLY  NO
	 3 PDB011			  READ WRITE NO
````

There are only two pluggable databases now, one seed PDB (system-supplied template that the CDB can use to create new PDBs) and one user-created PDBs (application data). Where did pluggable database PDB012 go? Pluggable database PDB012 was not created at the moment when the Automatic Backup you used to restore this database was taken.

Type **exit** command tree times followed by Enter to close all sessions (SQL*Plus, oracle user, and SSH).

````
exit

exit

exit
````

## Step 4: Configure Automatic Backups

On Oracle cloud console, click on hamburger menu ≡, then **Bare Metal, VM, and Exadata** under Databases. Click **[Your Initials]-DB** DB System.

Click the database name link **[Your Initials]DB** in the bottom table called Databases. Under Database Information details, review Backup Retention Period: 30 Days, and Backup Schedule: Anytime.

Click **Configure Automatic Backups** button. Set the following values:

- Backup retention period: 45 days
- Backup scheduling (UTC): 12:00AM - 2:00AM

Click **Save Changes**. Database lifecycle state changes to Backup In Progress... Wait for Database to become Available. Under Database Information details, review againBackup Retention Period and Backup Schedule fields. New configuration is displayed:

- Backup Retention Period: 45 Days
- Backup Schedule: 12:00AM - 2:00AM UTC

## Step 5: Create New Database Service from Backup

On Oracle cloud console, click on hamburger menu ≡, then **Bare Metal, VM, and Exadata** under Databases. Click **[Your Initials]-DB** DB System.

Click the database name link **[Your Initials]DB** in the bottom table called Databases.

Access Backups table, and next to Manual-Backup click ⋮ > **Create Database**. On the Create Database from Backup dialog, enter the following values:

- Select **Create a new DB system** radio button
- Name your DB system: [Your Initials]-DBb (e.g. VLT-DBb)
- Change Shape: VM.Standard2.1
- Oracle Database software edition: Enterprise Edition Extreme Performance
- Choose Storage Management Software: Logical Volume Manager
- Upload SSH key files: id_rsa.pub
- Choose a license type: Bring Your Own License (BYOL)
- Virtual cloud network: [Your Initials]-VCN
- Client Subnet: Public Subnet
- Hostname prefix: [Your Initials]-hostb (small case, e.g. vlt-hostb)
- Database name: [Your Initials]DBB (e.g. VLTDBB)
- Password: DBlearnPTS#20_
- Enter the source database's TDE wallet or RMAN password: DBlearnPTS#20_

Click **Create Database**. Status is Provisioning...

When it becomes Available, click **Nodes** on the left menu, and copy Public IP Address in your notes.

## Step 6: Verify New Database Service Created from Backup

Connect to the new [Your Initials]-DBb Database node using SSH (the one you just created from backup).

````
ssh -C -i id_rsa opc@<DBb Node Public IP Address>
````

All Oracle software components are installed with **oracle** OS user. Use the substitute user command to start a session as **oracle** user.

````
sudo su - oracle
````
You can connect to the database instance specified by environment variables.

````
sqlplus / as sysdba
````

List all pluggable databases. This database has three pluggable databases, one seed PDB (system-supplied template that the CDB can use to create new PDBs) and two user-created PDBs (application data). Pluggable database PDB012 was created at the moment when you took the Manual Backup you used to create this new database system.

````
show pdbs

    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			  READ ONLY  NO
	 3 PDB011			  READ WRITE NO
	 4 PDB012			  READ WRITE NO
````

Set current container connection to the pluggable database **PDB012**.

````
alter session set container=PDB012;
````

Verify the application data stored in **HR** schema.

````
set linesize 130
col TABLE_NAME format a40

select TABLE_NAME, NUM_ROWS from DBA_TABLES where OWNER='HR';

TABLE_NAME				   NUM_ROWS
---------------------------------------- ----------
REGIONS 					  4
LOCATIONS					 23
DEPARTMENTS					 27
JOBS						 19
EMPLOYEES					107
JOB_HISTORY					 10
COUNTRIES					 25

7 rows selected.
````

Type **exit** command tree times followed by Enter to close all sessions (SQL*Plus, oracle user, and SSH).

````
exit

exit

exit
````

## Step 7: Terminate New Database Service to Release Resources

On Oracle cloud console, click on hamburger menu ≡, then **Bare Metal, VM, and Exadata** under Databases. Click **[Your Initials]-DBb** DB System.

Click **More Actions** > **Terminate**. Type in the DB System Name to confirm termination: **[Your Initials]-DBb**. Click **Terminate DB System**. Status becomes Terminating... If you want to see more details, click **Work Requests** in the lower left menu. Click on **Terminate DB System** operation. Here you can see Log Messages, Error Messages, Associated Resources.

## Acknowledgements

- **Author** - Valentin Leonard Tabacaru
- **Last Updated By/Date** - Valentin Leonard Tabacaru, Principal Product Manager, DB Product Management, Aug 2020

See an issue? Please open up a request [here](https://github.com/oracle/learning-library/issues). Please include the workshop name and lab in your request.

