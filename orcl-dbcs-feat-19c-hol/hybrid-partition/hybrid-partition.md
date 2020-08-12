# Managing Hybrid Partitioned Tables

## Introduction

You can use the `EXTERNAL PARTITION ATTRIBUTES` clause of the `CREATE TABLE` statement to determine hybrid partitioning for a table. The partitions of the table can be external and or internal.

A hybrid partitioned table enables partitions to reside both in database data files (internal partitions) and in external files and sources (external partitions). You can create and query a hybrid partitioned table to utilize the benefits of partitioning with classic partitioned tables, such as pruning, on data that is contained in both internal and external partitions.

The `EXTERNAL PARTITION ATTRIBUTES` clause of the `CREATE TABLE` statement is defined at the table level for specifying table level external parameters in the hybrid partitioned table, such as:

- The access driver type, such as `ORACLE_LOADER`, `ORACLE_DATAPUMP`, `ORACLE_HDFS`, `ORACLE_HIVE`
- The default directory for all external partitions files
- The access parameters

The `EXTERNAL` clause of the `PARTITION` clause defines the partition as an external partition. When there is no `EXTERNAL` clause, the partition is an internal partition. You can specify for each external partition different attributes than the default attributes defined at the table level, such the directory. 

## Step 1: Get Ready For Hybrid Partitioning

This workshop requires an Oracle Database 19c installation. The virtual machine received from the instructor contains Oracle Database 19c installed with Multitenant architecture, and one pluggable database that will be used for the execution of the following exercises.

Connect to the Database node using SSH.

````
ssh -C -i id_rsa opc@<DB Node Public IP Address>
````

Use the substitute user command to start a session as **oracle** user.

````
sudo su - oracle
````

For this lab we will use the Sales History (SH) sample schema. Create two new folders on disk. These folders will be used later as location for some external partitions.

````
mkdir -p /home/oracle/sales_1998

mkdir -p /home/oracle/sales_1999
````

Connect to your pluggable database as SYSDBA.

````
$ sqlplus sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name> as SYSDBA
````

Create the directory objects for the folders we will use as locations for external partitions.

````
CREATE DIRECTORY SALES_98 AS '/home/oracle/sales_1998';

GRANT READ, WRITE ON DIRECTORY SALES_98 TO sh;
````

Grant SH full access to both directories.

````
CREATE DIRECTORY SALES_99 AS '/home/oracle/sales_1999';

GRANT READ, WRITE ON DIRECTORY SALES_99 TO sh;
````

For the tasks we will execute in this lab, SH user needs some privileges.

````
GRANT CREATE ANY DIRECTORY TO sh;
GRANT EXECUTE ON dbms_sql TO sh;
GRANT EXECUTE ON utl_file TO sh;
````

The `DBMS_SQL` package provides an interface to use dynamic SQL to parse any data manipulation language (DML) or data definition language (DDL) statement using PL/SQL. Using the `UTL_FILE` package, PL/SQL programs can read and write operating system text files. Basically, `UTL_FILE` provides a restricted version of operating system stream file I/O. One use case is for exporting data into flat files, that will become external partitions.

## Step 2: Review Current Sales Table

From this point, all statements will be executed on your pluggable database as SH user, inside the Sales History schema.

````
conn sh/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name>
````

For queries we run in SQL*Plus, it is recommended to use some formatting statements. If we use SQL Developer, these statements may be skipped. 

````
SET LINESIZE 120
column TABLE_NAME format a27
column PARTITION_NAME format a25
column TABLESPACE_NAME format a15
column DIRECTORY_NAME format a20
column DIRECTORY_PATH format a80
````

This query will show all tables in SH schema, with their number of rows, if they are partitioned or not, and, for those partitioned, if they use hybrid partitioning.

````
select TABLE_NAME, NUM_ROWS, PARTITIONED, HYBRID from USER_TABLES;
````

Table `SALES` is partitioned, and does not use hybrid partitioning. We can use this table as an example, to see how hybrid partitioning works.

````
select unique to_char(TIME_ID, 'YYYY') from SALES order by 1;
````

There are four years of data in this table: 1998, 1999, 2000, and 2001. The table is partitioned by quarters.

````
select TABLE_NAME, PARTITION_NAME, TABLESPACE_NAME, READ_ONLY, NUM_ROWS 
  from USER_TAB_PARTITIONS 
  where TABLE_NAME = 'SALES' and NUM_ROWS != 0
  order by substr(PARTITION_NAME, 10) || substr(PARTITION_NAME, 8, 1);
````

We can copy previous years data (1998) to generate more data, creating new partitions automatically.

````
insert into SALES (PROD_ID, CUST_ID, TIME_ID, CHANNEL_ID, PROMO_ID, QUANTITY_SOLD, AMOUNT_SOLD)
  select PROD_ID, CUST_ID, add_months(TIME_ID,48), CHANNEL_ID, PROMO_ID, QUANTITY_SOLD, AMOUNT_SOLD from 
   SALES where to_char(TIME_ID, 'YY') = '98';

commit;
````

Gather schema statistics for SH schema to refresh table information about rows and partitons.

````
BEGIN
  DBMS_STATS.gather_schema_stats (
    ownname          => 'SH',
    cascade          => TRUE,
    options          => 'GATHER AUTO');
END;
/
````

Now list again the years of data in the `SALES` table, and the table partitions.

````
select unique to_char(TIME_ID, 'YYYY') from SALES order by 1;

select TABLE_NAME, PARTITION_NAME, TABLESPACE_NAME, READ_ONLY, NUM_ROWS 
  from USER_TAB_PARTITIONS 
  where TABLE_NAME = 'SALES' and NUM_ROWS != 0
  order by substr(PARTITION_NAME, 10) || substr(PARTITION_NAME, 8, 1);
````

After generating new rows, we have sample data in quarter 3 of year 2002. Partition extension clause `PARTITION` (partition_name) specify the name of the partition within table from which you want to retrieve data.

````
select count(*) from SALES partition (SALES_Q3_2002);

50515
````

Now, there are more than 50k rows in this partition.

## Step 3: Move Old Data to Flat Files

We can convert a table with only internal partitions to a hybrid partitioned table. For example `SALES` table could be converted to a hybrid partitioned table by adding external partition attributes using `ALTER TABLE` command, then add external partitions. Note that at least one partition must be an internal partition. However, we will create a new hybrid partitioned table, as a copy of `SALES` table instead. 

The external partitions will be defined using comma-separated (CSV) data files stored in directories, that point to folders on disk in this case. These external CSV files are exported using existing internal partitions of the `SALES` table. The following procedure will be used to export individual partitions of a table into CSV files on disk.

````
create or replace PROCEDURE table_part_to_csv (
    p_tname in varchar2,
    p_pname in varchar2,
    p_dir in varchar2,
    p_filename in varchar2)
is
  l_output utl_file.file_type;
  l_theCursor integer default dbms_sql.open_cursor;
  l_columnValue varchar2(4000);
  l_status integer;
  l_query varchar2(1000) default 'select * from '|| p_tname || ' partition (' || p_pname || ') where 1=1';
  l_colCnt number := 0;
  l_separator varchar2(1);
  l_descTbl dbms_sql.desc_tab;
begin

--create an empty file on disk
  l_output := utl_file.fopen( p_dir, p_filename, 'w','32760');
  execute immediate 'alter session set nls_date_format=''dd-mon-yyyy hh24:mi:ss'' ';
  dbms_sql.parse( l_theCursor, l_query, dbms_sql.native );
  dbms_sql.describe_columns( l_theCursor, l_colCnt, l_descTbl );

--write column names into the new file
  for i in 1 .. l_colCnt loop
    utl_file.put( l_output, l_separator || '"' || l_descTbl(i).col_name || '"' );
    dbms_sql.define_column( l_theCursor, i, l_columnValue, 4000 );
    l_separator := ',';
  end loop;
  utl_file.new_line( l_output );
 
--write data into the new file and close
  l_status := dbms_sql.execute(l_theCursor);
 
  while ( dbms_sql.fetch_rows(l_theCursor) > 0 ) loop
    l_separator := '';
    for i in 1 .. l_colCnt loop
      dbms_sql.column_value( l_theCursor, i, l_columnValue );
      utl_file.put( l_output, l_separator || '"' || l_columnValue || '"');
      l_separator := ',';
    end loop;
    utl_file.new_line( l_output );
  end loop;
  dbms_sql.close_cursor(l_theCursor);
  utl_file.fclose( l_output );


/*exception
when others then
execute immediate 'alter session set nls_date_format=''dd-mon-yyyy hh24:mi:ss'' ';
raise;*/

END table_part_to_csv;
/
````

Some formatting lines for SQL*Plus.

````
SET LINESIZE 120
column DIRECTORY_NAME format a20
column DIRECTORY_PATH format a55
````

Check directory objects that will be used as destination to export `SALES` table partitions as CSV external files.

````
select DIRECTORY_NAME, DIRECTORY_PATH from ALL_DIRECTORIES where DIRECTORY_NAME like 'SALES%';
````

If all directories are prepared, we can export partition `SALES_Q1_1998` from table `SALES`, as external file `SALES_Q1_1998.csv` located in directory SALES_98.

````
exec table_part_to_csv('SALES','SALES_Q1_1998','SALES_98','SALES_Q1_1998.csv');
````

Using your favorite text editor, check the contents of `SALES_Q1_1998.csv` file. Run this command in a new tab of the Terminal window on the remote desktop connection. You can also use vim editor in the command line, or `cat` (list entire file) or `head -50` (list first 50 rows) commands from SQL*Plus session. 

The last option is the easiest. First line contains the names of the columns, and all data fields are separated by double quotes. This format can be customized. 

````
!head -50 /home/oracle/sales_1998/SALES_Q1_1998.csv

"PROD_ID","CUST_ID","TIME_ID","CHANNEL_ID","PROMO_ID","QUANTITY_SOLD","AMOUNT_SOLD"
"13","987","10-jan-1998 00:00:00","3","999","1","1232.16"
"13","1660","10-jan-1998 00:00:00","3","999","1","1232.16"
"13","1762","10-jan-1998 00:00:00","3","999","1","1232.16"
"13","1843","10-jan-1998 00:00:00","3","999","1","1232.16"
"13","1948","10-jan-1998 00:00:00","3","999","1","1232.16"
"13","2273","10-jan-1998 00:00:00","3","999","1","1232.16"
"13","2380","10-jan-1998 00:00:00","3","999","1","1232.16"
"13","2683","10-jan-1998 00:00:00","3","999","1","1232.16"
"13","2865","10-jan-1998 00:00:00","3","999","1","1232.16"
"13","4663","10-jan-1998 00:00:00","3","999","1","1232.16"
"13","5203","10-jan-1998 00:00:00","3","999","1","1232.16"
...
````

or

````
$ gedit /home/oracle/sales_1998/SALES_Q1_1998.csv
````

or

````
$ vi /home/oracle/sales_1998/SALES_Q1_1998.csv
````

For this workshop we will go ahead and use this same format for the rest of the `SALES` table partitions that store data for the years 1998 and 1999.

````
begin
table_part_to_csv('SALES','SALES_Q2_1998','SALES_98','SALES_Q2_1998.csv');
table_part_to_csv('SALES','SALES_Q3_1998','SALES_98','SALES_Q3_1998.csv');
table_part_to_csv('SALES','SALES_Q4_1998','SALES_98','SALES_Q4_1998.csv');
table_part_to_csv('SALES','SALES_Q1_1999','SALES_99','SALES_Q1_1999.csv');
table_part_to_csv('SALES','SALES_Q2_1999','SALES_99','SALES_Q2_1999.csv');
table_part_to_csv('SALES','SALES_Q3_1999','SALES_99','SALES_Q3_1999.csv');
table_part_to_csv('SALES','SALES_Q4_1999','SALES_99','SALES_Q4_1999.csv');
end;
/
````

Check all files and make sure the export operation was successful. This commands will display the total size on disk, file permissions and owner, file size, date of creation, and name of the file.

````
!ls -lah /home/oracle/sales_1998/

total 9.9M
drwxr-xr-x 2 oracle oinstall 4.0K Aug 12 08:02 .
drwx------ 6 oracle oinstall 4.0K Aug 12 07:34 ..
-rw-r--r-- 1 oracle oinstall 2.4M Aug 12 07:53 SALES_Q1_1998.csv
-rw-r--r-- 1 oracle oinstall 2.0M Aug 12 08:02 SALES_Q2_1998.csv
-rw-r--r-- 1 oracle oinstall 2.8M Aug 12 08:02 SALES_Q3_1998.csv
-rw-r--r-- 1 oracle oinstall 2.7M Aug 12 08:02 SALES_Q4_1998.csv

!ls -lah /home/oracle/sales_1999/

total 14M
drwxr-xr-x 2 oracle oinstall 4.0K Aug 12 08:03 .
drwx------ 6 oracle oinstall 4.0K Aug 12 07:34 ..
-rw-r--r-- 1 oracle oinstall 3.6M Aug 12 08:03 SALES_Q1_1999.csv
-rw-r--r-- 1 oracle oinstall 3.0M Aug 12 08:03 SALES_Q2_1999.csv
-rw-r--r-- 1 oracle oinstall 3.7M Aug 12 08:03 SALES_Q3_1999.csv
-rw-r--r-- 1 oracle oinstall 3.5M Aug 12 08:03 SALES_Q4_1999.csv
````

Now we have the external files required to create a new Hybrid Partitioned Table.

## Step 4: Implement Hybrid Partitioned Tables

In this example we assume our OLTP application will continue to run on the original `SALES` table, and we can drop the partitions containing old data, for example years 1998 and 1999. For reporting and compliancy we will store old data outside the database, on a cheaper storage solution.

For the new table, we want to keep the same table structure as the original `SALES` table. However, there are some restrictions we need to consider for Hybrid Partitioned Tables. For example, only single level `LIST` and `RANGE` partitioning are supported, no column default value is allowed, and only `RELY` constraints are allowed. We check the original table, and make sure all these restrictions are applied to the new table.

Connected to your pluggable database as SH user, format the output in SQL*Plus.

````
set pages 999
set long 90000
col definition format a120
````

Use the following statement to fetch the DDL for `SALES` table under the SH schema.

````
select dbms_metadata.get_ddl('TABLE','SALES','SH') definition from dual;
````

With all that in mind, using the DDL retrieved for `SALES` table, this is the new Hybrid Partitioned that will be called `HYBRID_SALES`. Our hybrid range-partitioned table is created with eight external partitions and sixteen internal partitions. You can specify for each external partition different attributes than the default attributes defined at the table level, such as directory.

Range partitioning maps data to partitions based on ranges of values of the partitioning key that you establish for each partition. Range partitioning is the most common type of partitioning and is often used with dates. For a table with a date column as the partitioning key, the January-2017 partition would contain rows with partitioning key values from 01-Jan-2017 to 31-Jan-2017.

Each partition has a `VALUES LESS THAN` clause, that specifies a non-inclusive upper bound for the partitions. Any values of the partitioning key equal to or higher than this literal are added to the next higher partition. All partitions, except the first, have an implicit lower bound specified by the `VALUES LESS THAN` clause of the previous partition.
You can use SQL Developer to execute this large statement, and make sure there are no errors produced by the copy-paste operation.

````
CREATE TABLE "SH"."HYBRID_SALES"
   ("PROD_ID" NUMBER NOT NULL ENABLE,
	"CUST_ID" NUMBER NOT NULL ENABLE,
	"TIME_ID" DATE NOT NULL ENABLE,
	"CHANNEL_ID" NUMBER NOT NULL ENABLE,
	"PROMO_ID" NUMBER NOT NULL ENABLE,
	"QUANTITY_SOLD" NUMBER(10,2) NOT NULL ENABLE,
	"AMOUNT_SOLD" NUMBER(10,2) NOT NULL ENABLE
   ) PCTFREE 5 PCTUSED 40 INITRANS 1 MAXTRANS 255 NOCOMPRESS  NOLOGGING
  EXTERNAL PARTITION ATTRIBUTES (
    TYPE ORACLE_LOADER 
    DEFAULT DIRECTORY sales_98
     ACCESS PARAMETERS(
       FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"'
       (prod_id,cust_id,time_id DATE 'DD-MON-YYYY HH24:MI:SS',channel_id,promo_id,quantity_sold,amount_sold)
     ) 
    REJECT LIMIT UNLIMITED
   ) 
  PARTITION BY RANGE ("TIME_ID")
 (
 PARTITION "SALES_Q1_1998"  VALUES LESS THAN (TO_DATE(' 1998-04-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS')) EXTERNAL 
  DEFAULT DIRECTORY SALES_98 LOCATION ('SALES_Q1_1998.csv'),
 PARTITION "SALES_Q2_1998"  VALUES LESS THAN (TO_DATE(' 1998-07-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS')) EXTERNAL 
  DEFAULT DIRECTORY SALES_98 LOCATION ('SALES_Q2_1998.csv'),
 PARTITION "SALES_Q3_1998"  VALUES LESS THAN (TO_DATE(' 1998-10-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS')) EXTERNAL 
  DEFAULT DIRECTORY SALES_98 LOCATION ('SALES_Q3_1998.csv'),
 PARTITION "SALES_Q4_1998"  VALUES LESS THAN (TO_DATE(' 1999-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS')) EXTERNAL 
  DEFAULT DIRECTORY SALES_98 LOCATION ('SALES_Q4_1998.csv'),
 PARTITION "SALES_Q1_1999"  VALUES LESS THAN (TO_DATE(' 1999-04-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS')) EXTERNAL 
  DEFAULT DIRECTORY SALES_99 LOCATION ('SALES_Q1_1999.csv'),
 PARTITION "SALES_Q2_1999"  VALUES LESS THAN (TO_DATE(' 1999-07-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS')) EXTERNAL 
  DEFAULT DIRECTORY SALES_99 LOCATION ('SALES_Q2_1999.csv'),
 PARTITION "SALES_Q3_1999"  VALUES LESS THAN (TO_DATE(' 1999-10-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS')) EXTERNAL 
  DEFAULT DIRECTORY SALES_99 LOCATION ('SALES_Q3_1999.csv'),
 PARTITION "SALES_Q4_1999"  VALUES LESS THAN (TO_DATE(' 2000-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS')) EXTERNAL 
  DEFAULT DIRECTORY SALES_99 LOCATION ('SALES_Q4_1999.csv'),
 PARTITION "SALES_Q1_2000"  VALUES LESS THAN (TO_DATE(' 2000-04-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN')) SEGMENT CREATION IMMEDIATE PCTFREE 0 PCTUSED 40 INITRANS 1 MAXTRANS 255
  COMPRESS BASIC NOLOGGING ,
 PARTITION "SALES_Q2_2000"  VALUES LESS THAN (TO_DATE(' 2000-07-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN')) SEGMENT CREATION IMMEDIATE PCTFREE 0 PCTUSED 40 INITRANS 1 MAXTRANS 255
  COMPRESS BASIC NOLOGGING ,
 PARTITION "SALES_Q3_2000"  VALUES LESS THAN (TO_DATE(' 2000-10-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN')) SEGMENT CREATION IMMEDIATE PCTFREE 0 PCTUSED 40 INITRANS 1 MAXTRANS 255 
  COMPRESS BASIC NOLOGGING ,
 PARTITION "SALES_Q4_2000"  VALUES LESS THAN (TO_DATE(' 2001-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN')) SEGMENT CREATION IMMEDIATE PCTFREE 0 PCTUSED 40 INITRANS 1 MAXTRANS 255
  COMPRESS BASIC NOLOGGING ,
 PARTITION "SALES_Q1_2001"  VALUES LESS THAN (TO_DATE(' 2001-04-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN')) SEGMENT CREATION IMMEDIATE PCTFREE 0 PCTUSED 40 INITRANS 1 MAXTRANS 255
  COMPRESS BASIC NOLOGGING ,
 PARTITION "SALES_Q2_2001"  VALUES LESS THAN (TO_DATE(' 2001-07-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN')) SEGMENT CREATION IMMEDIATE PCTFREE 0 PCTUSED 40 INITRANS 1 MAXTRANS 255
  COMPRESS BASIC NOLOGGING ,
 PARTITION "SALES_Q3_2001"  VALUES LESS THAN (TO_DATE(' 2001-10-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN')) SEGMENT CREATION IMMEDIATE PCTFREE 0 PCTUSED 40 INITRANS 1 MAXTRANS 255
  COMPRESS BASIC NOLOGGING ,
 PARTITION "SALES_Q4_2001"  VALUES LESS THAN (TO_DATE(' 2002-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN')) SEGMENT CREATION IMMEDIATE PCTFREE 0 PCTUSED 40 INITRANS 1 MAXTRANS 255
  COMPRESS BASIC NOLOGGING ,
 PARTITION "SALES_Q1_2002"  VALUES LESS THAN (TO_DATE(' 2002-04-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN')) SEGMENT CREATION IMMEDIATE PCTFREE 0 PCTUSED 40 INITRANS 1 MAXTRANS 255
  COMPRESS BASIC NOLOGGING ,
 PARTITION "SALES_Q2_2002"  VALUES LESS THAN (TO_DATE(' 2002-07-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN')) SEGMENT CREATION IMMEDIATE PCTFREE 0 PCTUSED 40 INITRANS 1 MAXTRANS 255
  COMPRESS BASIC NOLOGGING ,
 PARTITION "SALES_Q3_2002"  VALUES LESS THAN (TO_DATE(' 2002-10-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN')) SEGMENT CREATION IMMEDIATE PCTFREE 0 PCTUSED 40 INITRANS 1 MAXTRANS 255
  COMPRESS BASIC NOLOGGING ,
 PARTITION "SALES_Q4_2002"  VALUES LESS THAN (TO_DATE(' 2003-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN')) SEGMENT CREATION IMMEDIATE PCTFREE 0 PCTUSED 40 INITRANS 1 MAXTRANS 255
  COMPRESS BASIC NOLOGGING ,
 PARTITION "SALES_Q1_2003"  VALUES LESS THAN (TO_DATE(' 2003-04-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN')) SEGMENT CREATION IMMEDIATE PCTFREE 0 PCTUSED 40 INITRANS 1 MAXTRANS 255
  COMPRESS BASIC NOLOGGING ,
 PARTITION "SALES_Q2_2003"  VALUES LESS THAN (TO_DATE(' 2003-07-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN')) SEGMENT CREATION IMMEDIATE PCTFREE 0 PCTUSED 40 INITRANS 1 MAXTRANS 255
  COMPRESS BASIC NOLOGGING ,
 PARTITION "SALES_Q3_2003"  VALUES LESS THAN (TO_DATE(' 2003-10-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN')) SEGMENT CREATION IMMEDIATE PCTFREE 0 PCTUSED 40 INITRANS 1 MAXTRANS 255
  COMPRESS BASIC NOLOGGING ,
 PARTITION "SALES_Q4_2003"  VALUES LESS THAN (TO_DATE(' 2004-01-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS',
  'NLS_CALENDAR=GREGORIAN')) SEGMENT CREATION IMMEDIATE PCTFREE 5 PCTUSED 40 INITRANS 1 MAXTRANS 255
  NOCOMPRESS NOLOGGING );
````

If the statement executes successfully, our new `HYBRID_SALES` table is created. As you have noticed already, we removed all Foreign Keys from the original statement, because of the restrictions on Hybrid Partitioned Tables, only `RELY` constraints are allowed.

## Step 5: Add Foreign Keys

Foreign Key constraints are constantly validated for all data coming into the `SALES` table. Using `RELY` constraints, it means that you can trust the original `SALES` table to provide clean data, instead of implementing constraints in the `HYBRID_SALES` table used for reporting.

`RELY` constraints, even though they are not used for data validation, can enable more sophisticated query rewrites for materialized views, or enable other data warehousing tools to retrieve information regarding constraints directly from the Oracle data dictionary. Creating a `RELY` constraint is inexpensive and does not impose any overhead during DML or load. Because the constraint is not being validated, no data processing is necessary to create it.

````
alter table "SH"."HYBRID_SALES" add CONSTRAINT "HYBRID_SALES_CHANNEL_FK" FOREIGN KEY ("CHANNEL_ID")
 REFERENCES "SH"."CHANNELS" ("CHANNEL_ID") RELY DISABLE NOVALIDATE;

ORA-25158: Cannot specify RELY for foreign key if the associated primary key is NORELY
````

This statement assumes that the Primary Key is in the `RELY` state. If not sure, check the reference `CHANNELS` table.

````
SET LINESIZE 120
column CONSTRAINT_NAME format a27
column TABLE_NAME format a27

select CONSTRAINT_NAME, CONSTRAINT_TYPE, TABLE_NAME, RELY
  from USER_CONSTRAINTS where TABLE_NAME='CHANNELS';
````

We can see that the Primary Key `CHANNELS_PK` is not in `RELY` state.

````
CONSTRAINT_NAME	CONSTRAINT_TYPE	TABLE_NAME	RELY
CHANNELS_PK		P			CHANNELS	(null)
````

Fix the issue with the following `ALTER TABLE` command.

````
alter table "SH"."CHANNELS" modify constraint CHANNELS_PK rely;
````

Check again the state of the Primary Key.

````
select CONSTRAINT_NAME, CONSTRAINT_TYPE, TABLE_NAME, RELY
  from USER_CONSTRAINTS where TABLE_NAME='CHANNELS';
````

Now it is in `RELY` state.

````
CONSTRAINT_NAME	CONSTRAINT_TYPE	TABLE_NAME	RELY
CHANNELS_PK		P			CHANNELS	RELY
````

At this moment we can add the `RELY` Foreign Key constraint on `HYBRID_SALES` table.

````
alter table "SH"."HYBRID_SALES" add CONSTRAINT "HYBRID_SALES_CHANNEL_FK" FOREIGN KEY ("CHANNEL_ID")
 REFERENCES "SH"."CHANNELS" ("CHANNEL_ID") RELY DISABLE NOVALIDATE;
````

Optionally add more Foreign Keys from the original `SALES` table to the `HYBRID_SALES`, using the same method.

## Step 6: Review Major Differences

With a hybrid partitioned table created, we can test how it works, and some differences from traditional partitioned tables. There is a long list of supported operations on hybrid partitioned tables. Here are just a few:

-	Using `ALTER TABLE` DDLs such as `ADD`, `DROP`, and `RENAME` partitions
-	Modifying for external partitions the location of the external data sources at the partition level
-	Changing the existing location to an empty location resulting in an empty external partition
-	Creating materialized views that include external partitions in `QUERY_REWRITE_INTEGRITY` stale tolerated mode only
-	Full partition wise refreshing on external partitions

First thing we can notice is that data is already stored in the external partitions. These partitions are actually the CSV files we exported from the database.

````
select count(*) from HYBRID_SALES partition (SALES_Q1_1998);

43687

select count(*) from HYBRID_SALES partition (SALES_Q3_1998);

50515

select count(*) from HYBRID_SALES partition (SALES_Q4_1999);

62388
````

In the internal partitions there are no records yet.

````
select count(*) from HYBRID_SALES partition (SALES_Q1_2000);

0

select count(*) from HYBRID_SALES partition (SALES_Q3_2002);

0
````

We can insert rows into internal partitions from the original `SALES` table. Using the partition extension clause `PARTITION` (partition_name), we can specify the name of the partition within `SALES` table from which we want to retrieve data, in order to populate one internal partition in particular.

````
insert into HYBRID_SALES (PROD_ID, CUST_ID, TIME_ID, CHANNEL_ID, PROMO_ID, QUANTITY_SOLD, AMOUNT_SOLD)
  select PROD_ID,CUST_ID,TIME_ID,CHANNEL_ID,PROMO_ID,QUANTITY_SOLD,AMOUNT_SOLD from 
   SALES partition (SALES_Q1_2000);

commit;
````

Now we can confirm that internal partition has all records available in the original `SALES` table.

````
select count(*) from HYBRID_SALES partition (SALES_Q1_2000);

62197
````

## Step 7: Insert Records In New Table

Verify individual records from the new table, filtering on product ID `PROD_ID` and customer ID `CUST_ID`.

````
select * from HYBRID_SALES where PROD_ID = 136 and CUST_ID = 6033;
````

Try to insert some data into the new table. Notice this record is inserted into an internal partition.

````
insert into HYBRID_SALES (PROD_ID, CUST_ID, TIME_ID, CHANNEL_ID, PROMO_ID, QUANTITY_SOLD, AMOUNT_SOLD) 
  values (136, 6033, TO_DATE('2000-02-01 00:00:00', 'YYYY-MM-DD HH24:MI:SS'), 3, 999, 2, 16.58);

commit;
````

Retrieve the record to check is was properly inserted.

````
select * from HYBRID_SALES where PROD_ID = 136 and CUST_ID = 6033;
````

We can check also some records that are located on the external partitions.

````
select * from HYBRID_SALES where PROD_ID = 13 and CUST_ID = 987;
````

However, if we try to insert or update data in an external partition, we get an error. Data in external partitions cannot be modified using traditional commands, as if we have specified the `READ ONLY` clause at the partition level in the `CREATE TABLE` statement.

````
insert into HYBRID_SALES (PROD_ID, CUST_ID, TIME_ID, CHANNEL_ID, PROMO_ID, QUANTITY_SOLD, AMOUNT_SOLD) 
  values (13, 987, TO_DATE('1998-04-15 00:00:00', 'YYYY-MM-DD HH24:MI:SS'), 3, 999, 2, 232.16);

ORA-14466: Data in a read-only partition or subpartition cannot be modified.
````

Trying to switch the partition using the `ALTER TABLE` statement to read write mode is not successful either.

````
alter table HYBRID_SALES modify partition SALES_Q2_1998 read write;

ORA-14354: operation not supported for a hybrid-partitioned table
````

In conclusion, data in external partitions I read only, and we can insert or modify only the data in internal partitions.

````
insert into HYBRID_SALES (PROD_ID, CUST_ID, TIME_ID, CHANNEL_ID, PROMO_ID, QUANTITY_SOLD, AMOUNT_SOLD) 
  values (13, 987, TO_DATE('2003-04-15 00:00:00', 'YYYY-MM-DD HH24:MI:SS'), 3, 999, 2, 232.16);

commit;
````

All queries issued on the Hybrid Partitioned Table will retrieve data from both internal and external partitions.

````
select * from HYBRID_SALES where PROD_ID = 13 and CUST_ID = 987;
````

Gathering schema statistics for schemas with Hybrid Partitioned Tables is performed in the same way as usual.

````
BEGIN
  DBMS_STATS.gather_schema_stats (
    ownname          => 'SH',
    cascade          => TRUE,
    options          => 'GATHER AUTO');
END;
/
````

Format the output in SQL*Plus.

````
SET LINESIZE 120
column TABLE_NAME format a27
````

Verify again all tables in SH schema, with their number of rows, if they are partitioned or not, and, for those partitioned, if they use hybrid partitioning.

````
select TABLE_NAME, NUM_ROWS, PARTITIONED, HYBRID from USER_TABLES;
````

## Acknowledgements

- **Author** - Valentin Leonard Tabacaru
- **Last Updated By/Date** - Valentin Leonard Tabacaru, Principal Product Manager, DB Product Management, Aug 2020

See an issue? Please open up a request [here](https://github.com/oracle/learning-library/issues). Please include the workshop name and lab in your request.

