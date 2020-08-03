# Data Guard Physical Standby Database

## Introduction

In this lab you will learn how to use the cloud console to manage Oracle Data Guard associations in your DB System. An Oracle Data Guard implementation requires two DB systems, one containing the primary database and one containing the standby database. When you enable Oracle Data Guard for a virtual machine DB system database, a new DB system with the standby database is created and associated with the primary database.

>**Note** : An Oracle Data Guard configuration on the Oracle Cloud Infrastructure is limited to one standby database for each primary database.

Basic requirements:

- Both DB systems must be in the same compartment.
- The DB systems must be the same shape type.
- The database versions and editions must be identical. 
- Oracle Data Guard does not support Oracle Database Standard Edition. 
- Active Data Guard requires Enterprise Edition - Extreme Performance.

## Step 1: Enable Data Guard

On Oracle cloud console, click on hamburger menu ≡, then **Bare Metal, VM, and Exadata** under Databases. Click **[Your Initials]-DB** DB System. 

Add to your notes the Availability Domain you see on the DB System Information page (e.g. IWcS:EU-FRANKFURT-1-AD-3).

On the DB System Details page, click the database name link **[Your Initials]DB** in the bottom table called Databases.

Observe Data Guard - Status: Not enabled, on the Database Information page.

Click **Data Guard Associations** in the lower left menu. Click **Enable Data Guard**. Provide the following details:

- Display name: [Your Initials]-DBSb
- Availability domain: AD2 (different from your DB System AD)
- Virtual cloud network: [Your Initials]-VCN
- Client Subnet: Public Subnet
- Hostname prefix: [Your Initials]-hostsb
- Password: DBlearnPTS#20_

**Enable Data Guard**.

Click ≡, then **Bare Metal, VM, and Exadata** under Databases. Click **[Your Initials]-DBSb** DB System. Status is Provisioning... If you want to see more details, click **Work Requests** in the lower left menu. Check Operation Create Data Guard having status In Progress... Wait until your Standby DB System is Available. On **[Your Initials]-DBSb** DB System details page, click **Nodes** in the lower left menu, and and copy Public IP Address and Private IP Address in your notes.

## Step 2: Connect to Standby DB System

Connect to the Standby DB System node using SSH.

````
ssh -C -i id_rsa opc@<Standby Node Public IP Address>
````

All Oracle software components are installed with **oracle** OS user. Use the substitute user command to start a session as **oracle** user.

````
sudo su - oracle
````

The Data Guard command-line interface (DGMGRL) enables you to manage a Data Guard broker configuration and its databases directly from the command line, or from batch programs or scripts.

````
dgmgrl
````

To run DGMGRL, you must have SYSDBA privileges. Log in as user SYSDG with the SYSDG administrative privilege to perform Data Guard operations. You can use this privilege with either Data Guard Broker or the DGMGRL command-line interface.

````
CONNECT sysdg;
Password: DBlearnPTS#20_
Connected to "DBS001_fra2qq"
Connected as SYSDBA.
````

The SHOW CONFIGURATION command displays a summary and status of the broker configuration.

The summary lists all members included in the broker configuration and other information pertaining to the broker configuration itself, including the fast-start failover status and the transport lag and apply lag of all standby databases. Write down in your notes the names of the two databases, Primary and Physical standby, and their role (e.g. DBS001_fra3hb - Primary, DBS001_fra2qq - Physical standby).

````
SHOW CONFIGURATION;

Configuration - DBS001_fra3hb_DBS001_fra2qq

  Protection Mode: MaxPerformance
  Members:
  DBS001_fra3hb - Primary database
    DBS001_fra2qq - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 27 seconds ago)
````

The SHOW DATABASE command displays information, property values, or initialization parameter values of the specified database and its instances. Adding VERBOSE keyword to the command, it shows properties of the database in addition to the brief summary. This is the Primary database.

````
SHOW DATABASE VERBOSE 'DBS001_fra3hb';

Database - DBS001_fra3hb

  Role:               PRIMARY
  Intended State:     TRANSPORT-ON
  Instance(s):
    DBS001

  Properties:
    DGConnectIdentifier             = 'DBS001_fra3hb'
    ObserverConnectIdentifier       = ''
    FastStartFailoverTarget         = ''
    PreferredObserverHosts          = ''
    LogShipping                     = 'ON'
    RedoRoutes                      = ''
    LogXptMode                      = 'ASYNC'
    DelayMins                       = '0'
    Binding                         = 'optional'
    MaxFailure                      = '0'
    ReopenSecs                      = '300'
    NetTimeout                      = '30'
    RedoCompression                 = 'DISABLE'
    PreferredApplyInstance          = ''
    ApplyInstanceTimeout            = '0'
    ApplyLagThreshold               = '30'
    TransportLagThreshold           = '30'
    TransportDisconnectedThreshold  = '30'
    ApplyParallel                   = 'AUTO'
    ApplyInstances                  = '0'
    StandbyFileManagement           = ''
    ArchiveLagTarget                = '0'
    LogArchiveMaxProcesses          = '0'
    LogArchiveMinSucceedDest        = '0'
    DataGuardSyncLatency            = '0'
    LogArchiveTrace                 = '0'
    LogArchiveFormat                = ''
    DbFileNameConvert               = ''
    LogFileNameConvert              = ''
    ArchiveLocation                 = ''
    AlternateLocation               = ''
    StandbyArchiveLocation          = ''
    StandbyAlternateLocation        = ''
    InconsistentProperties          = '(monitor)'
    InconsistentLogXptProps         = '(monitor)'
    LogXptStatus                    = '(monitor)'
    SendQEntries                    = '(monitor)'
    RecvQEntries                    = '(monitor)'
    HostName                        = 'dbs01h'
    StaticConnectIdentifier         = '(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=dbs01h)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=DBS001_fra3hb_DGMGRL.sub07141037280.rehevcn.oraclevcn.com)(INSTANCE_NAME=DBS001)(SERVER=DEDICATED)))'
    TopWaitEvents                   = '(monitor)'
    SidName                         = '(monitor)'

  Log file locations:
    Alert log               : /u01/app/oracle/diag/rdbms/dbs001_fra3hb/DBS001/trace/alert_DBS001.log
    Data Guard Broker log   : /u01/app/oracle/diag/rdbms/dbs001_fra3hb/DBS001/trace/drcDBS001.log

Database Status:
SUCCESS
````

Show the brief summary of the Physical stadnby database.

````
SHOW DATABASE 'DBS001_fra2qq';

Database - DBS001_fra2qq

  Role:               PHYSICAL STANDBY
  Intended State:     APPLY-ON
  Transport Lag:      0 seconds (computed 7 seconds ago)
  Apply Lag:          9 seconds (computed 7 seconds ago)
  Average Apply Rate: 0 Byte/s
  Real Time Query:    ON
  Instance(s):
    DBS001

Database Status:
SUCCESS
````

The SHOW FAST_START FAILOVER command displays all fast-start failover related information. 

````
SHOW FAST_START FAILOVER;

Fast-Start Failover:  Disabled

  Protection Mode:    MaxPerformance
  Lag Limit:          30 seconds

  Threshold:          30 seconds
  Active Target:      (none)
  Potential Targets:  (none)
  Observer:           (none)
  Shutdown Primary:   TRUE
  Auto-reinstate:     TRUE
  Observer Reconnect: (none)
  Observer Override:  FALSE

Configurable Failover Conditions
  Health Conditions:
    Corrupted Controlfile          YES
    Corrupted Dictionary           YES
    Inaccessible Logfile            NO
    Stuck Archiver                  NO
    Datafile Write Errors          YES

  Oracle Error Conditions:
    (none)
````

Exit DGMGRL.

````
EXIT
````

## Step 3: Perform Switchover Operation

Click ≡, then **Bare Metal, VM, and Exadata** under Databases. Click **[Your Initials]-DB** DB System. 

On the DB System Details page, click the database name link **[Your Initials]DB** in the bottom table called Databases.

Click **Data Guard Associations** in the lower left menu. Observe Peer DB System and it's details like Peer Role, Shape, Availability Domain. Click **⋮** > **Switchover**. Enter the database admin password: DBlearnPTS#20_.

Status will change to Updating... Click **Work Requests** in the lower left menu, and the Operation name link. Here you can see Log Messages, Error Messages, Associated Resources.

You can use the breadcrumbs links in the upper section of the page to navigate to superior levels: Overview > Bare Metal, VM and Exadata > DB Systems > DB System Details > Database Home Details > Database Details. Click **Database Details**, wait for Status to become Available.

Launch DGMGRL.

````
dgmgrl
````

Log in as user SYSDG with the SYSDG administrative privilege.

````
CONNECT sysdg;
Password: DBlearnPTS#20_
Connected to "DBS001_fra2qq"
Connected as SYSDBA.
````

Show the current configuration. Observe the roles of the two databases are swapped.

````
SHOW CONFIGURATION;

Configuration - DBS001_fra3hb_DBS001_fra2qq

  Protection Mode: MaxPerformance
  Members:
  DBS001_fra2qq - Primary database
    DBS001_fra3hb - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 56 seconds ago)
````

View the Primary database information.

````
SHOW DATABASE 'DBS001_fra2qq';

Database - DBS001_fra2qq

  Role:               PRIMARY
  Intended State:     TRANSPORT-ON
  Instance(s):
    DBS001

Database Status:
SUCCESS
````

## Step 4: Enable Fast-Start Failover

Fast-start failover allows the broker to automatically fail over to a previously chosen standby database in the event of loss of the primary database. Fast-start failover quickly and reliably fails over the target standby database to the primary database role, without requiring you to perform any manual steps to invoke the failover. Fast-start failover can be used only in a broker configuration and can be configured only through DGMGRL or Enterprise Manager Cloud Control.

Use DGMGRL to enable fast-start failover.

````
ENABLE FAST_START FAILOVER;
Enabled in Potential Data Loss Mode.
````

Because our Protection Mode is MaxPerformance and LogXptMode database property is ASYNC, Fast-Start Failover is Enabled in Potential Data Loss Mode.

>**Note** : Enabling fast-start failover in a configuration operating in maximum performance mode provides better overall performance on the primary database because redo data is sent asynchronously to the target standby database. Note that this does not guarantee no data will be lost. 

````
SHOW FAST_START FAILOVER;

Fast-Start Failover: Enabled in Potential Data Loss Mode

  Protection Mode:    MaxPerformance
  Lag Limit:          30 seconds

  Threshold:          30 seconds
  Active Target:      DBS001_fra3hb
  Potential Targets:  "DBS001_fra3hb"
    DBS001_fra3hb valid
  Observer:           (none)
  Shutdown Primary:   TRUE
  Auto-reinstate:     TRUE
  Observer Reconnect: (none)
  Observer Override:  FALSE

Configurable Failover Conditions
  Health Conditions:
    Corrupted Controlfile          YES
    Corrupted Dictionary           YES
    Inaccessible Logfile            NO
    Stuck Archiver                  NO
    Datafile Write Errors          YES

  Oracle Error Conditions:
    (none)
````

By default, the fast-start failover target for the Primary database is the Physical standby database. 

````
SHOW DATABASE 'DBS001_fra2qq' FastStartFailoverTarget;
  FastStartFailoverTarget = 'DBS001_fra3hb'
````

And vice versa.

````
SHOW DATABASE 'DBS001_fra3hb' FastStartFailoverTarget;
  FastStartFailoverTarget = 'DBS001_fra2qq'
````

## Step 5: Change Protection Mode

Oracle Data Guard provides three protection modes: maximum availability, maximum performance, and maximum protection.

Maximum Performance mode provides the highest level of data protection that is possible without affecting the performance of a primary database. This is accomplished by allowing transactions to commit as soon as all redo data generated by those transactions has been written to the online log. Redo data is also written to one or more standby databases, but this is done asynchronously with respect to transaction commitment, so primary database performance is unaffected by the time required to transmit redo data and receive acknowledgment from a standby database. 

Maximum Availability mode provides the highest level of data protection that is possible without compromising the availability of a primary database. Under normal operations, transactions do not commit until all redo data needed to recover those transactions has been written to the online redo log AND based on user configuration, one of the following is true:
- redo has been received at the standby, I/O to the standby redo log has been initiated, and acknowledgement sent back to primary;
- redo has been received and written to standby redo log at the standby and acknowledgement sent back to primary.

Use DGMGRL to disable fast-start failover.

````
DISABLE FAST_START FAILOVER;
Disabled.
````

Configure the redo transport service to SYNC on Primary database.

````
EDIT DATABASE 'DBS001_fra2qq' SET PROPERTY LogXptMode='SYNC';
Property "logxptmode" updated
````

Configure also the redo transport service to SYNC on Physical standby database.

````
EDIT DATABASE 'DBS001_fra3hb' SET PROPERTY LogXptMode='SYNC';
Property "logxptmode" updated
````

Upgrade the broker configuration to the MAXAVAILABILITY protection mode.

````
EDIT CONFIGURATION SET PROTECTION MODE AS MaxAvailability;
Succeeded.
````

Display the configuration information. If this command is executed immediately, it will display a warning message. The information was not fully updated, and you need to wait a few seconds to verify the configuration.

````
SHOW CONFIGURATION;

Configuration - DBS001_fra3hb_DBS001_fra2qq

  Protection Mode: MaxAvailability
  Members:
  DBS001_fra2qq - Primary database
    Warning: ORA-16629: database reports a different protection level from the protection mode

    DBS001_fra3hb - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
WARNING   (status updated 54 seconds ago)
````

After a minute, display again the configuration information. As you can see, the protection mode was updated.

````
SHOW CONFIGURATION;

Configuration - DBS001_fra3hb_DBS001_fra2qq

  Protection Mode: MaxAvailability
  Members:
  DBS001_fra2qq - Primary database
    DBS001_fra3hb - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 56 seconds ago)
````

## Step 6: Stop Redo Apply on Standby

It is possible to temporarily stop the Redo Apply on a Physical standby database. To change the state of the standby database to APPLY-OFF, enter the EDIT DATABASE command as shown in the following line.

````
EDIT DATABASE 'DBS001_fra3hb' SET STATE='APPLY-OFF';
Succeeded.
````

Use SHOW DATABASE command to display information, property values, or initialization parameter values of the Physical standby database. Redo data is still being received when you put the physical standby database in the APPLY-OFF state. But redo data is not applied, and you will notice the increasing values of Apply Lag. Also Average Apply Rate has an unknown value.

````
SHOW DATABASE 'DBS001_fra3hb';

Database - DBS001_fra3hb

  Role:               PHYSICAL STANDBY
  Intended State:     APPLY-OFF
  Transport Lag:      0 seconds (computed 1 second ago)
  Apply Lag:          1 minute 57 seconds (computed 1 second ago)
  Average Apply Rate: (unknown)
  Real Time Query:    OFF
  Instance(s):
    DBS001

Database Status:
SUCCESS
````
Use again the EDIT DATABASE command to change the state of the Physical standby database, so it resumes the apply of redo data.

````
EDIT DATABASE 'DBS001_fra3hb' SET STATE='APPLY-ON';
Succeeded.
````

Verify the APPLY-ON state and Apply Lag that has to be as low as possible, ideally 0. Check the Average Apply Rate.

````
SHOW DATABASE 'DBS001_fra3hb';

Database - DBS001_fra3hb

  Role:               PHYSICAL STANDBY
  Intended State:     APPLY-ON
  Transport Lag:      0 seconds (computed 1 second ago)
  Apply Lag:          0 seconds (computed 1 second ago)
  Average Apply Rate: 8.00 KByte/s
  Real Time Query:    ON
  Instance(s):
    DBS001

Database Status:
SUCCESS
````

Exit DGMGRL.

````
EXIT
````

## Step 7: Test Fast-Start Failover with Maximum Availability 

On Oracle cloud console, click ≡, then **Bare Metal, VM, and Exadata** under Databases. Click **[Your Initials]-DBSb** DB System. 

On the DB System Details page, click the database name link **[Your Initials]DB** in the bottom table called Databases.

Click **Data Guard Associations** in the lower left menu. Click **⋮** > **Switchover**. Enter the database admin password: DBlearnPTS#20_. Wait for the status to become Available.

Launch DGMGRL.

````
dgmgrl
````

Log in as user SYSDG with the SYSDG administrative privilege.

````
CONNECT sysdg;
Password: DBlearnPTS#20_
Connected to "DBS001_fra2qq"
Connected as SYSDBA.
````

Now, with MAXAVAILABILITY protection mode, let's test again enabling fast-start failover. This time it is enabled in Zero Data Loss Mode.

````
ENABLE FAST_START FAILOVER;
Enabled in Zero Data Loss Mode.
````

Type **exit** command tree times followed by Enter to close all sessions (DGMGRL, oracle user, and SSH). 

````
EXIT

exit

exit
````

## Acknowledgements

- **Author** - Valentin Leonard Tabacaru
- **Last Updated By/Date** - Valentin Leonard Tabacaru, Principal Product Manager, DB Product Management, Aug 2020

See an issue? Please open up a request [here](https://github.com/oracle/learning-library/issues). Please include the workshop name and lab in your request.

