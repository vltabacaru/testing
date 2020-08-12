# Resource Manager and SQL Quarantine

## Introduction

SQL Quarantine protects an Oracle Database from performance degradation by preventing execution of SQL statements that excessively consume CPU and I/O resources.

SQL statements that are terminated by Oracle Database Resource Manager due to their excessive consumption of CPU and I/O resources are automatically quarantined. The execution plans associated with the terminated SQL statements are quarantined to prevent them from being executed again.

>**Note** : Some features in this lab are restricted to Engineered Systems platforms only, Exadata and Exadata Cloud Service.

## Step 1: Prepare Database and User 

Connect to the Compute node using SSH.

````
ssh -C -i id_rsa opc@<Compute Public IP Address>
````

Connect to your pluggable database as SYS user.

````
conn sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name> as sysdba
````

Create a new database user, called **UNAUTH**.

````
create user UNAUTH identified by DBlearnPTS#20_ default tablespace USERS temporary tablespace TEMP;
````

This user will execute some unauthorized workload on this pluggable database, so it needs some basic privileges.

````
grant connect, resource to UNAUTH;
grant unlimited tablespace to UNAUTH; 
````

## Step 2: Resource Manager Setup 

Clear any eventual changes in an existing pending area and create a new pending area. All changes to the plan schema must be done within a pending area. The pending area can be thought of as a "scratch" area for plan schema changes. The administrator creates this pending area, makes changes as necessary, possibly validates these changes, and only when the submit is completed do these changes become active.

````
exec DBMS_RESOURCE_MANAGER.CLEAR_PENDING_AREA;
exec DBMS_RESOURCE_MANAGER.CREATE_PENDING_AREA;
````

Create a new Resource Manager consumer group.

````
begin
DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP(
  consumer_group => 'EXEC_LIMIT_GROUP', 
  comment => 'Resource manager consumer group with limits'
  );
end;
/
````

Map sessions to consumer groups, based on the session's login and runtime attributes, like Oracle Database user name in our case.

````
begin
DBMS_RESOURCE_MANAGER.SET_CONSUMER_GROUP_MAPPING( 
  attribute => 'ORACLE_USER', 
  value => 'UNAUTH', 
  consumer_group => 'EXEC_LIMIT_GROUP'
  );
end;
/
````

Define a new resource plan.

````
begin
DBMS_RESOURCE_MANAGER.CREATE_PLAN( 
  plan => 'CPU_TIME_LIMIT', 
  comment => 'Kill statement after exceeding CPU time limit'
  );
end;
/
````

Create a resource plan directive, having parameter `SWITCH_GROUP` value `CANCEL_SQL`. If group name is `CANCEL_SQL`, then the current call is canceled when the switch condition is met. If the group name is `KILL_SESSION`, then the session is killed when the switch condition is met. If the group name is `LOG_ONLY`, then no action is taken other than recording this event via SQL monitor. Parameter `SWITCH_TIME` specifies the time on CPU (not elapsed time) that a session can execute before the action is taken. We set it to 30 seconds. Parameter `SWITCH_ESTIMATE`, if TRUE, tells Oracle to use its execution time estimate to automatically switch the consumer group of an operation before beginning its execution.

````
begin
DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
  plan => 'CPU_TIME_LIMIT',
  group_or_subplan => 'EXEC_LIMIT_GROUP',
  comment => 'Kill statement after exceeding CPU time limit',
  switch_group => 'CANCEL_SQL',
  switch_time => 30,
  switch_estimate => false
  );
end;
/
````

Create another resource plan directive for `OTHER_GROUPS` - all sessions that aren't mapped to a consumer group. Parameter `MGMT_P1` specifies the resource percentage at the first level - resource allocation value for level 1. In this case it give 100% of CPU to other sessions not mapped to our `EXEC_LIMIT_GROUP` consumer group.

>**Note** : There must be a plan directive for `OTHER_GROUPS` somewhere in any active plan schema.This ensures that a session not covered by the currently active plan is allocated resources as specified by the `OTHER_GROUPS` directive.

````
begin
DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
  plan => 'CPU_TIME_LIMIT',
  group_or_subplan => 'OTHER_GROUPS',
  comment => 'Do not affect others',
  mgmt_p1 => 100
  );
end;
/
````

Validate and apply the resource plan in the current pending area.

````
exec DBMS_RESOURCE_MANAGER.VALIDATE_PENDING_AREA;
exec DBMS_RESOURCE_MANAGER.SUBMIT_PENDING_AREA;
````

## Step 3: Prepare Sample Data and Workload 

Create a new connection to the Compute node using SSH.

````
ssh -C -i id_rsa opc@<Compute Public IP Address>
````

Connect to your pluggable database as UNAUTH user using SQL*Plus. Let's call this **Connection 2**.

### Connection 2: UNAUTH

````
sqlplus unauth/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name>
````

Create a simple table with two columns.

````
create table MY_REPORT(C number, D varchar2(500));
````

Insert 200K rows that will be used to generate a long running query simulating the unauthorized workload, for example a complex report.

````
begin
for i in 1..200000 loop
insert into MY_REPORT values(1,'longlonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglonglong');
end loop;
commit;
end;
/
````

Create an index on the sample table.

````
create index MY_REPORT_C_INDX on MY_REPORT(C);
````

Start a long running query that simulates a report, generating an unauthorized workload on the database. We also time the execute of this query, to compare the elapsed time for each case.

````
set timing on

SELECT count(*) FROM my_report r1, my_report r2 WHERE r1.c=r2.c AND r1.c=1;
````

At this point there is no mechanism in the database to prevent such a long running query to be executed. This query takes approximately 25 minutes to complete. The only way to abort it is by pressing Ctrl-C. Press Ctrl-C after a couple of minutes to abort the execution.

````
ERROR at line 1:
ORA-01013: user requested cancel of current operation

Elapsed: 00:01:58.01
````

## Step 4: Set Execution Limits for Unauthorized User

Now we will set some limits for the queries that user UNAUTH is allowed to execute.

### Connection 1: SYS

Use the first connection to the pluggable database as SYS user. `V$SESSION` view displays session information for each current session. Observe the value of the `RESOURCE_CONSUMER_GROUP` column.

````
set linesize 130
col USERNAME format a30
col RESOURCE_CONSUMER_GROUP format a30

select  SID, SERIAL#, USERNAME, RESOURCE_CONSUMER_GROUP from V$SESSION where USERNAME='UNAUTH';

       SID    SERIAL# USERNAME			     RESOURCE_CONSUMER_GROUP
---------- ---------- ------------------------------ ------------------------------
	36	11196 UNAUTH			     OTHER_GROUPS
````

`DBA_RSRC_GROUP_MAPPINGS` view displays the mapping between session attributes and consumer groups in the database. UNAUTH is mapped to `EXEC_LIMIT_GROUP` consumer group.

````
col ATTRIBUTE format a25
col VALUE format a25
col CONSUMER_GROUP format a25
col STATUS format a10

select ATTRIBUTE, VALUE, CONSUMER_GROUP, STATUS from DBA_RSRC_GROUP_MAPPINGS;

ATTRIBUTE		  VALUE 		    CONSUMER_GROUP	      STATUS
------------------------- ------------------------- ------------------------- -----------
ORACLE_USER		  UNAUTH		    EXEC_LIMIT_GROUP
````

`RESOURCE_MANAGER_PLAN` initialization parameter specifies the resource plan to use for a database. Set this parameter to the plan we create previously.

````
alter system set resource_manager_plan = 'CPU_TIME_LIMIT';
````

## Step 5: Test Execution Limits 

Now we have Oracle Database Resource Manager to prevent such a long running query to be executed. Resource plan directive, having parameter `SWITCH_GROUP` value `CANCEL_SQL` will cancel the current call after 30 seconds.

### Connection 2: UNAUTH

Use the second connection to the pluggable database as UNAUTH user. Start a long running query that simulates the unauthorized workload again. See what happens. After approximately 30 seconds the long running query is canceled.

````
set timing on

SELECT count(*) FROM my_report r1, my_report r2 WHERE r1.c=r2.c AND r1.c=1;

ERROR at line 1:
ORA-00040: active time limit exceeded - call aborted

Elapsed: 00:00:31.96
````

### Connection 1: SYS

Use the first connection to the pluggable database as SYS user. Observe again the `RESOURCE_CONSUMER_GROUP` column in `V$SESSION` view.

````
select SID, SERIAL#, USERNAME, RESOURCE_CONSUMER_GROUP from V$SESSION where USERNAME='UNAUTH';

       SID    SERIAL# USERNAME			     RESOURCE_CONSUMER_GROUP
---------- ---------- ------------------------------ ------------------------------
	36	47806 UNAUTH			     EXEC_LIMIT_GROUP
````

It confirms user UNAUTH was mapped correctly with the corresponding consumer group.

## Step 6: SQL Quarantine Setup 

`V$SQL` view lists statistics on shared SQL areas. `SQL_ID` is the SQL identifier of the parent cursor in the library cache. `PLAN_HASH_VALUE` is the numeric representation of the current SQL plan for this cursor. We need these two values to configure SQL Quarantine.

````
select SQL_ID, PLAN_HASH_VALUE from V$SQL where SQL_TEXT = 'SELECT count(*) FROM my_report r1, my_report r2 WHERE r1.c=r2.c AND r1.c=1';

SQL_ID	      PLAN_HASH_VALUE
------------- ---------------
ant1c7jnqf8yu	   1413800128
````

You can create a quarantine configuration for an execution plan of a SQL statement using any of these `DBMS_SQLQ` package functions – `CREATE_QUARANTINE_BY_SQL_ID` or `CREATE_QUARANTINE_BY_SQL_TEXT`. 

````
declare
  unauth_quarantine varchar2(30);
begin
  unauth_quarantine := DBMS_SQLQ.CREATE_QUARANTINE_BY_SQL_ID(SQL_ID => 'ant1c7jnqf8yu');
end;
/
````

Query the `DBA_SQL_QUARANTINE` view to get details of all the quarantine configurations.

````
col SQL_TEXT format a30
col NAME format a40

select NAME, SQL_TEXT, PLAN_HASH_VALUE, ENABLED from DBA_SQL_QUARANTINE;

NAME					 SQL_TEXT			PLAN_HASH_VALUE ENA
---------------------------------------- ------------------------------ --------------- ---
SQL_QUARANTINE_7quzzzhxrb2hg		 SELECT count(*) FROM my_report 		YES
					  r1, my_report r2 WHERE r1.c=r
					 2.c AND r1.c=1
````

We have one SQL Quarantine defined for the long running query.

## Step 7: Test SQL Quarantine 

Now we have SQL Quarantine to prevent such a long running query to be executed. If a SQL statement attempts to use the execution plan that is quarantined, is not allowed to run, being cancelled immediately, thus preventing database performance degradation, even for 30 seconds.

### Connection 2: UNAUTH

Use the second connection to the pluggable database as UNAUTH user. Start the long running query that simulates the unauthorized workload again.

````
set timing on

SELECT count(*) FROM my_report r1, my_report r2 WHERE r1.c=r2.c AND r1.c=1;

ERROR at line 1:
ORA-56955: quarantined plan used

Elapsed: 00:00:00.01
````

It is terminated in .01 seconds.

### Connection 1: SYS

Use the first connection to the pluggable database as SYS user.

You can query the `V$SQL` and `GV$SQL` views to get details about the quarantined execution plans of SQL statements. `SQL_QUARANTINE` column contains the name of the SQL quarantine configuration, and `AVOIDED_EXECUTIONS` the number of times this cursor was prevented from being used due to the plan being quarantined.

````
col SQL_QUARANTINE format a40

select SQL_QUARANTINE, AVOIDED_EXECUTIONS from V$SQL where SQL_ID='ant1c7jnqf8yu';

SQL_QUARANTINE				 AVOIDED_EXECUTIONS
---------------------------------------- ------------------
SQL_QUARANTINE_7quzzzhxrb2hg				  1
````

You can query a quarantine threshold for a quarantine configuration using the `DBMS_SQLQ.GET_PARAM_VALUE_QUARANTINE` function. The following example returns the quarantine threshold for elapsed time for the quarantine configuration. When a specific value is not specified, quarantines have thresholds set to `ALWAYS`.

````
declare
    quarantine_elapsed_time_limit varchar2(30);
begin
    quarantine_elapsed_time_limit := 
        DBMS_SQLQ.GET_PARAM_VALUE_QUARANTINE(
              quarantine_name => 'SQL_QUARANTINE_7quzzzhxrb2hg',
              parameter_name  => 'ELAPSED_TIME');
    DBMS_OUTPUT.PUT_LINE('SQL quarantined parameter is: ' || quarantine_elapsed_time_limit); 
end;
/
SQL quarantined parameter is: ALWAYS
````

You can enable or disable a quarantine configuration using the `DBMS_SQLQ.ALTER_QUARANTINE` procedure. A quarantine configuration is enabled by default when it is created. 

## Acknowledgements

- **Author** - Valentin Leonard Tabacaru
- **Last Updated By/Date** - Valentin Leonard Tabacaru, Principal Product Manager, DB Product Management, Aug 2020

See an issue? Please open up a request [here](https://github.com/oracle/learning-library/issues). Please include the workshop name and lab in your request.

