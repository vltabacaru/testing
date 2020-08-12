# Memoptimized Rowstore Fast Ingest

## Introduction

The fast ingest functionality of Memoptimized Rowstore enables fast data inserts into an Oracle Database from applications, such as Internet of Things (IoT) applications that ingest small, high volume transactions with a minimal amount of transactional overhead. The insert operations that use fast ingest temporarily buffer the data in the large pool before writing it to disk in bulk in a deferred, asynchronous manner.

Using the rich analytical features of Oracle Database, you can now perform data analysis more effectively by easily integrating data from high-frequency data streaming applications with your existing application data.

Fast ingest uses `MEMOPTIMIZE_WRITE` to insert data into tables specified as `MEMOPTIMIZE FOR WRITE` hint. The database temporarily buffers these inserts in the large pool and automatically commits the changes at the time of writing these buffered inserts to disk. The changes cannot be rolled back.

The inserts using fast ingest are also known as deferred inserts, because they are initially buffered in the large pool and later written to disk asynchronously by background processes.

>**Note** : Some features in this lab are restricted to Database Cloud Service Enterprise Edition - Extreme Performance and Engineered Systems platforms only, Exadata and Exadata Cloud Service.

## Step 1: First Row Insert by Fast Ingest 

Connect to the Compute node using SSH.

````
ssh -C -i id_rsa opc@<Compute Public IP Address>
````

Connect to your pluggable database as HR user using SQL*Plus. 

````
sqlplus hr/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name>
````

Create a new table enabled for fast ingest by specifying the `MEMOPTIMIZE FOR WRITE` hint in the `CREATE TABLE` or `ALTER TABLE` statement. However, `MEMOPTIMIZE FOR WRITE` feature cannot be enabled on a table in encrypted tablespace.

````
create table MEMOPTWRITES
      ( C1 number, C2 varchar2(12)) 
memoptimize for write;

ORA-62172: MEMOPTIMIZE FOR WRITE feature cannot be enabled on table in encrypted tablespace.
````

Connect to your pluggable database as SYS user.

````
conn sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name> as sysdba
````

Verify the encryption on all tablespaces. All new tablespaces created on Oracle Database Cloud Service instances are encrypted by default, and this cannot be avoided.

````
col TABLESPACE_NAME format a20

select TABLESPACE_NAME, STATUS, ENCRYPTED, CONTENTS from DBA_TABLESPACES;
````

Create again a new table enabled for fast ingest, this time in `SYSAUX` tablespace.

````
create table HR.MEMOPTWRITES
      ( C1 number, C2 varchar2(12)) 
tablespace SYSAUX 
memoptimize for write;
````

It will not work because, by default, a table does not have a segment created until a first row is inserted. `MEMOPTIMIZE FOR WRITE` tables require a segment created before the first row is inserted. We need to modify a parameter called `deferred_segment_creation`.

````
show parameter deferred_segment_creation

NAME                            TYPE        VALUE
------------------------------- ----------- --------------------
deferred_segment_creation       Boolean     TRUE
````

Set `deferred_segment_creation` parameter to **FALSE**.

````
alter system set deferred_segment_creation=FALSE scope=BOTH;
````

Now, we can finally create the table enabled for fast ingest in the **HR** schema.

````
create table HR.MEMOPTWRITES
      ( C1 number, C2 varchar2(12)) 
tablespace SYSAUX 
memoptimize for write;
````

Connect to your pluggable database as HR user.

````
conn hr/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name>
````

Verify all tables and the tablespace they reside in.

````
col TABLE_NAME format a25
col TABLESPACE_NAME format a20

select TABLE_NAME, TABLESPACE_NAME from USER_TABLES;
````

Enable fast ingest for inserts by specifying the `MEMOPTIMIZE_WRITE` hint in the `INSERT` statement.

>**Note** : The following inserts is not how fast ingest is meant to be used in the real world, but demonstrates the mechanism. In the real world, the data inserted using fast ingest comes from IoT devices like sensors, smart meters, or traffic cameras.

````
insert /*+ MEMOPTIMIZE_WRITE */ into MEMOPTWRITES values (1, 'Memoptwrites');
````

Check if data has been written in your table.

````
select * from MEMOPTWRITES;
````

The result of the insert above is to write data to the ingest buffer in the large pool of the SGA. At some point, that data is flushed to the `MEMOPTWRITES` table. Until that happens, the data is not durable.

The first time an insert is run, the fast ingest area is allocated from the large pool. The amount of memory allocated is written to the alert.log. 

You can manually flush all the fast ingest data from the large pool to disk for the current session using `DBMS_MEMOPTIMIZE.WRITE_END` procedure.

````
exec DBMS_MEMOPTIMIZE.WRITE_END
````

Verify the data has been written in your table.

````
select * from MEMOPTWRITES;
````

## Step 2: Investigate Fast Ingest Operations

Create a new connection to the Compute node using SSH.

````
ssh -C -i id_rsa opc@<Compute Public IP Address>
````

Connect to your pluggable database as SYS user using SQL*Plus. Let's call this **Connection 2**.

### Connection 2: SYS 

````
sqlplus sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name> as sysdba
````

As SYSDBA, you can see some administrative information about the fast ingest buffers by querying the view `V$MEMOPTIMIZE_WRITE_AREA`.

````
select * from V$MEMOPTIMIZE_WRITE_AREA;
````

There are two new columns in `DBA_TABLES` view: `MEMOPTIMIZE_READ` - indicates whether the table is enabled for Fast Key Based Access, and `MEMOPTIMIZE_WRITE` - indicates whether the fast ingest `MEMOPTIMIZE FOR WRITE` attribute is set.

````
select MEMOPTIMIZE_READ Mem_read,
       MEMOPTIMIZE_WRITE Mem_write
 from DBA_TABLES
 where TABLE_NAME = 'MEMOPTWRITES';
````

### Connection 1: HR

Use the first connection to your pluggable database as HR user. Insert another row using fast ingest. As you can see, fast ingest does not require to commit rows, as with Memoptimized Rowstore Fast Ingest the normal Oracle transaction mechanisms are bypassed. Since the application doesn’t have to wait for the data to be written to disk, the inserts statement returns extremely quickly.

````
insert /*+ MEMOPTIMIZE_WRITE */ into MEMOPTWRITES values (2, 'Memoptwrites');
````

### Connection 2: SYS

Use the second connection to your pluggable database as SYS user. Verify the data in `HR.MEMOPTWRITES` table.

````
select * from HR.MEMOPTWRITES;
````

If the deferred, asynchronous process has not written to disk the content of the buffer, which occurs periodically, then you may not see the last row inserted. As SYS user, you can use `DBMS_MEMOPTIMIZE_ADMIN` package to control the fast ingest data in the large pool, with `WRITES_FLUSH` procedure that flushes all the fast ingest data from the large pool to disk for all the sessions.

````
exec DBMS_MEMOPTIMIZE_ADMIN.WRITES_FLUSH
````

Verify again the data in `HR.MEMOPTWRITES` table.

````
select * from HR.MEMOPTWRITES;
````

List the statistics about memoptimized rows written and flushed. This query shows the number of rows written via the space allocated for fast ingest writes in the large pool.

````
select DISPLAY_NAME, VALUE from V$MYSTAT m, V$STATNAME n
  where m.STATISTIC# = n.STATISTIC#
  and DISPLAY_NAME in ('memopt w rows written',
                       'memopt w rows flushed');
````

### Connection 1: HR 

Use the first connection to your pluggable database as HR user. Verify the data in `MEMOPTWRITES` table.

````
select * from MEMOPTWRITES;
````

Delete all rows in `MEMOPTWRITES` table.

````
delete from MEMOPTWRITES;
````

The delete statement does not use fast ingest, so it does require to commit.

````
commit;
````

## Step 3: Compare Fast Ingest with Transactional Inserts

As HR user, create a new transactional table, with the same structure as the table enabled for fast ingest.

````
create table NORMALWRITES
      ( C1 number, C2 varchar2(12));
````

We measure and display the time it takes to insert 2 million rows in the transactional table. These rows include a column that is generated as a random string.

````
set serveroutput on

declare
n number;
begin
n := DBMS_UTILITY.GET_TIME;
for i in 1..2000000 loop
insert into NORMALWRITES values(i,DBMS_RANDOM.STRING('x',10));
end loop;
commit;
n := (DBMS_UTILITY.GET_TIME - n)/100;
DBMS_OUTPUT.PUT_LINE('Execution time: ' || n || ' sec');
end;
/
````

We also measure and display the time it takes to insert 2 million rows in the fast ingest enabled table. Compare the two values.

````
declare
n number;
begin
n := DBMS_UTILITY.GET_TIME;
for i in 1..2000000 loop
insert /*+ MEMOPTIMIZE_WRITE */ into MEMOPTWRITES values(i,DBMS_RANDOM.string('x',10));
end loop;
n := (DBMS_UTILITY.GET_TIME - n)/100;
DBMS_OUTPUT.PUT_LINE('Execution time: ' || n || ' sec');
end;
/
````

We can also run the same test, but instead of a random string, we can use a fixed value. First, insert 2 millions rows more into the transactional table.

````
declare
n number;
begin
n := DBMS_UTILITY.GET_TIME;
for i in 1..2000000 loop
insert into NORMALWRITES values(i,'Normalwrites');
end loop;
commit;
n := (DBMS_UTILITY.GET_TIME - n)/100;
DBMS_OUTPUT.PUT_LINE('Execution time: ' || n || ' sec');
end;
/
````

After, insert 2 millions rows more into the fast ingest enabled table. Compare the two values. What do you observe?

````
declare
n number;
begin
n := DBMS_UTILITY.GET_TIME;
for i in 1..2000000 loop
insert /*+ MEMOPTIMIZE_WRITE */ into MEMOPTWRITES values(i,'Memoptwrites');
end loop;
n := (DBMS_UTILITY.GET_TIME - n)/100;
DBMS_OUTPUT.PUT_LINE('Execution time: ' || n || ' sec');
end;
/
````

### Connection 2: SYS

Use the second connection to your pluggable database as SYS user. Verify the data in `HR.MEMOPTWRITES` table.

````
select count(*) from HR.MEMOPTWRITES;
````

Run `WRITES_FLUSH` procedure that flushes all the fast ingest data from the large pool to disk for all sessions.

````
exec DBMS_MEMOPTIMIZE_ADMIN.WRITES_FLUSH
````

List the statistics about memoptimized rows written and flushed.

````
select DISPLAY_NAME, VALUE from V$MYSTAT m, V$STATNAME n
  where m.STATISTIC# = n.STATISTIC#
  and DISPLAY_NAME in ('memopt w rows written',
                       'memopt w rows flushed');
````

Clean the environment.

````
drop table HR.NORMALWRITES;

truncate table HR.MEMOPTWRITES;
````

## Step 4: Some Fast Ingest Particularities

As SYS user (or HR user) try to run a multi-row insert using fast ingest for inserts by specifying the `MEMOPTIMIZE_WRITE` hint in the `INSERT` statement. It doesn't work, as this is one of the limitations of fast ingest feature.

````
insert /*+ MEMOPTIMIZE_WRITE */ all 
  into HR.MEMOPTWRITES values (1, 'Memoptwrites')
  into HR.MEMOPTWRITES values (2, 'Memoptwrites')
 select 1 from DUAL;

ORA-62139: MEMOPTIMIZE_WRITE hint disallowed for this operation
````

Drop `HR.MEMOPTWRITES` table.

````
drop table HR.MEMOPTWRITES;
````

Create a new `HR.MEMOPTWRITES` table, this time with a `PRIMARY KEY` constraint.

````
create table HR.MEMOPTWRITES
      ( C1 number primary key, C2 varchar2(12)) 
tablespace SYSAUX 
memoptimize for write;
````

### Connection 1: HR

Use the first connection to your pluggable database as HR user. Insert 10 rows in the new `MEMOPTWRITES` table using fast ingest, having the same value for the `PRIMARY KEY` column.

````
begin
for i in 1..10 loop
insert /*+ MEMOPTIMIZE_WRITE */ into MEMOPTWRITES values(1,DBMS_RANDOM.STRING('x',10));
end loop;
end;
/
````

Manually flush all the fast ingest data from the large pool to disk for the current session.

````
exec DBMS_MEMOPTIMIZE.WRITE_END
````

Verify the data in your table.

````
select * from MEMOPTWRITES;
````

Index operations and constraint checking is done only when the data is written from the fast ingest area in the large pool to disk. If primary key violations occur when the background processes write data to disk, then the database will not write those rows to the database.

## Acknowledgements

- **Author** - Valentin Leonard Tabacaru
- **Last Updated By/Date** - Valentin Leonard Tabacaru, Principal Product Manager, DB Product Management, Aug 2020

See an issue? Please open up a request [here](https://github.com/oracle/learning-library/issues). Please include the workshop name and lab in your request.

