# Oracle 19c Automatic Indexing

## Introduction

Creating indexes manually requires deep knowledge of the data model, application, and data distribution. Often DBAs make choices about which indexes to create, and then never revise their choices. As a result, opportunities for improvement are lost, and unnecessary indexes can become a performance liability.

With automatic indexing the database monitors the application workload, creating and maintaining indexes automatically. The indexing feature is implemented as an automatic task that runs at a fixed interval. You can use the `DBMS_AUTO_INDEX` package to report on this automatic task and to set your preferences. 

The automatic indexing feature automates the index management tasks in an Oracle database. Automatic indexing automatically creates, rebuilds, and drops indexes in a database based on the changes in application workload, thus improving database performance. The automatically managed indexes are known as **auto indexes**.

Automatic indexing provides the following functionality:

- Runs the automatic indexing process in the background periodically at a predefined time interval.
- Analyzes application workload, and accordingly creates new indexes and drops the existing underperforming indexes to improve database performance.
- Rebuilds the indexes that are marked unusable due to table partitioning maintenance operations, such as ALTER TABLE MOVE.
- Provides PL/SQL APIs for configuring automatic indexing in a database and generating reports related to automatic indexing operations.

>**Note** : Some features in this lab are restricted to Database Cloud Service Enterprise Edition - Extreme Performance and Engineered Systems platforms only, Exadata and Exadata Cloud Service.

## Step 1: Check Existing Indexes

For these labs we will use the Sales History (SH) sample schema. Use SQL*Plus and connect to your pluggable database as SYS user. You may use for this lab the SQL Developer tool via remote desktop connection.

````
$ sqlplus sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name> as SYSDBA
````

It is recommended to format the output in SQL*Plus, for a better view of the information listed.

````
set linesize 120
column table_name format a30
column  index_name format a30
column  index_type format a25
column  last_analyzed format a25 
````

List all existing indexes created on tables in SH schema.

````
select table_name,index_name,index_type,last_analyzed from dba_indexes where table_owner = 'SH'; 
````

DBA_INDEXES describes all indexes in the database, and we use the owner of the indexed object as filter.

## Step 2: Drop Existing Indexes 

For the purpose of this exercise, we drop most of the existing indexes in SH schema.

````
drop index  SH.COSTS_PROD_BIX;
drop index  SH.COSTS_TIME_BIX;
drop index  SH.SALES_PROD_BIX;
drop index  SH.SALES_CUST_BIX;
drop index  SH.SALES_TIME_BIX;
drop index  SH.SALES_CHANNEL_BIX;
drop index  SH.SALES_PROMO_BIX;
drop index  SH.PRODUCTS_PROD_SUBCAT_IX;
drop index  SH.PRODUCTS_PROD_CAT_IX;
drop index  SH.PRODUCTS_PROD_STATUS_BIX;
drop index  SH.CUSTOMERS_GENDER_BIX;
drop index  SH.CUSTOMERS_MARITAL_BIX;
drop index  SH.CUSTOMERS_YOB_BIX;
drop index  SH.FW_PSC_S_MV_SUBCAT_BIX;
drop index  SH.FW_PSC_S_MV_CHAN_BIX;
drop index  SH.FW_PSC_S_MV_PROMO_BIX;
drop index  SH.FW_PSC_S_MV_WD_BIX; 
````

>**Note** : This is not an action you can perform on a production database. Please be careful before dropping indexes in your own environments.

Gather SH schema statistics.

````
exec dbms_stats.gather_schema_stats(ownname => 'SH', estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE, method_opt => 'FOR ALL COLUMNS SIZE  AUTO', degree => 4);
````

SH schema statistics are gathered.
 
## Step 3: Automatic Indexing Configuration

Automatic indexing requires little to no manual intervention, but a package called `DBMS_AUTO_INDEX` package is provided for changing a small number of defaults.

You can configure automatic indexing in an Oracle database using the `DBMS_AUTO_INDEX.CONFIGURE` procedure.

By default, the permanent tablespace specified during the database creation is used for storing auto indexes. However, we create a new tablespace to store all auto indexes, connected as SYS to your pluggable database.

````
CREATE TABLESPACE tbs_auto_idx;
````

>**Note** : You may receive the folowing error when creating a new tablespace in a Database Coud Service instance:
>````
>ORA-28427: cannot create, import or restore unencrypted tablespace: TBS_AUTO_IDX in Oracle Cloud
>````
>Verify that parameter `ENCRYPT_NEW_TABLESPACES` has the value `CLOUD_ONLY`. If not, set this value, and retry creating the tablepace.
>````
>alter system set encrypt_new_tablespaces=CLOUD_ONLY scope=BOTH;
>````

This will avoid any conflicts for storage resources between application data and auto indexes created by the database.

## Step 4: Enable Auto Indexing Implementation 

You can use the `AUTO_INDEX_DEFAULT_TABLESPACE` configuration setting to specify a tablespace to store auto indexes.

````
exec DBMS_AUTO_INDEX.CONFIGURE('AUTO_INDEX_DEFAULT_TABLESPACE','TBS_AUTO_IDX');
````

You can use the `AUTO_INDEX_MODE` configuration setting to enable or disable automatic indexing in a database. The feature can be enabled as follows:

````
exec DBMS_AUTO_INDEX.CONFIGURE('AUTO_INDEX_MODE','IMPLEMENT');
````

There are three possible values for `AUTO_INDEX_MODE` configuration setting: `OFF` (default), `IMPLEMENT`, and `REPORT ONLY`. 

- `OFF` disables automatic indexing in a database, so that no new auto indexes are created, and the existing auto indexes are disabled. 
- `IMPLEMENT` enables automatic indexing in a database and creates any new auto indexes as visible indexes, so that they can be used in SQL statements. 
- `REPORT ONLY` enables automatic indexing in a database, but creates any new auto indexes as invisible indexes, so that they cannot be used in SQL statements.

## Step 5: View Configuration Details

These lines are used to format the output in SQL*Plus.

````
set linesize 120
column parameter_name format a35
column parameter_value format a30
column last_modified format a30
column modified_by format a20
````

Now we can check the configuration values. All parameter settings can be seen as follows:

````
select * from DBA_AUTO_INDEX_CONFIG;
````

When automatic indexing is enabled in a database, all the schemas in the database can use auto indexes by default.

## Step 6: Enable Auto Indexing for Specific Schemas 

It is possible to specify which schemas are subject to auto indexing. The following statement add the HR schema to the exclusion list, so that the HR schema cannot use auto indexes.

````
exec DBMS_AUTO_INDEX.CONFIGURE('AUTO_INDEX_SCHEMA', 'HR', FALSE);
````

The following statement removes all the schema from the exclusion list, so that all the schemas in the database can use auto indexes.

````
exec DBMS_AUTO_INDEX.CONFIGURE('AUTO_INDEX_SCHEMA', NULL, TRUE);
````

For our exercise, we will use automatic indexing only for OE and SH schemas. These statements add OE and SH schemas to the inclusion list, so that only OE and SH schemas can use auto indexes.

````
exec DBMS_AUTO_INDEX.CONFIGURE('AUTO_INDEX_SCHEMA', 'OE', TRUE);
exec DBMS_AUTO_INDEX.CONFIGURE('AUTO_INDEX_SCHEMA', 'SH', TRUE);
````

All parameter settings plus schemas that have been included and excluded can be seen as follows:

````
select * from DBA_AUTO_INDEX_CONFIG;
````

Sample output for automatic indexing configuration parameters.

````
PARAMETER_NAME                   PARAMETER_VALUE     LAST_MODIFIED          MODIFIED_BY
-------------------------------- ------------------- ---------------------------- -----
AUTO_INDEX_COMPRESSION           OFF
AUTO_INDEX_DEFAULT_TABLESPACE    TBS_AUTO_IDX        18-JUN-19  10.24.26.000000 AM  SYS
AUTO_INDEX_MODE                  IMPLEMENT           18-JUN-19  10.24.31.000000 AM  SYS
AUTO_INDEX_REPORT_RETENTION      31
AUTO_INDEX_RETENTION_FOR_AUTO    373
AUTO_INDEX_RETENTION_FOR_MANUAL
AUTO_INDEX_SCHEMA                schema IN (OE, SH)  19-JUN-19  08.49.21.000000 AM  SYS
AUTO_INDEX_SPACE_BUDGET          50
````

Among other settings we can configure, we have:

- `AUTO_INDEX_RETENTION_FOR_AUTO` that sets the retention period for unused auto indexes, default is 373 days (one year plus one week). This means the unused auto indexes are deleted after 373 days;
- `AUTO_INDEX_RETENTION_FOR_MANUAL` configuration setting specifies a period for retaining unused non-auto indexes (manually created indexes) in a database;
- `AUTO_INDEX_SPACE_BUDGET` configuration setting can be used to specify percentage of tablespace to allocate for auto indexes. In our case, 50% of tablespace `TBS_AUTO_IDX` is allocated for this purpose.

## Step 7: Generate Auto Indexing Report

You can generate reports related to automatic indexing operations in an Oracle database using the `REPORT_ACTIVITY` and `REPORT_LAST_ACTIVITY` functions of the `DBMS_AUTO_INDEX` package. It is recommended to format the output in SQL*Plus, before running the report.

````
set linesize 120 trims on pagesize 1000 long 100000
column REPORT format a120
````

Here is an extract from an example report:

````
select DBMS_AUTO_INDEX.REPORT_ACTIVITY(sysdate-30,NULL,'text','all','all') REPORT from DUAL;
````

This example generates a report containing all information about the automatic indexing operations for the last 30 days. The report is generated in the text format and includes all automatic indexing operations. However, you may notice there is no information about automatic indexing operations because automatic indexing was OFF during this time, and there was no activity on the database.
 
## Step 8: Sample Data and Statistics

Automatic indexing can work with data for OLTP applications, which use large data sets and run millions of SQL statements a day, but also with data for data warehousing applications. For this simple exercise, we don’t have any application running on our database, so we will generate some synthetic workload manually.

We need new tables to store some dummy data that could easily modify. These new tables are created in our existing SH schema.

````
conn sh/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name>
````

Create a simple dummy table with two columns.

````
create table tempjfv (c number, d varchar2(1000));
````

Use the following code to insert dummy data into the new table.

````
begin
for i in 1..20000 loop
  insert into tempjfv values(-i,'aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa');
end loop;
commit;
end;
/
````

Now we have 20,000 rows in our dummy table. Create also a copy of the `SH.CUSTOMERS` table, using a CTAS statement.

````
create table customersjfv as select * from sh.customers;
````

Now we have some tables we can modify, and at the same time some OLTP workload was generated on our database, and we can check how automatic indexing feature reacts to this type of workload.

## Step 9: Gather Schema Statistics

As a recommendation and best practice, before we drill down into activity reviews for database performance advisors, it is important to gather optimizer statistics for SH schema, and check the current information about our advisor executions in the database.

For these tasks we need to connect with SYSDBA privileges, on your pluggable database.

````
conn sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name> as SYSDBA
````

The `DBMS_STATS` PL/SQL package collects and manages optimizer statistics. `GATHER_SCHEMA_STATS` procedure collects statistics for all objects in a schema.

````
execute dbms_stats.gather_schema_stats(ownname => 'SH', estimate_percent=> DBMS_STATS.AUTO_SAMPLE_SIZE, method_opt => 'FOR ALL COLUMNS SIZE  AUTO', degree => 4); 
````

- `AUTO_SAMPLE_SIZE` indicates that auto-sample size algorithms should be used. 
- `METHOD_OPT` statistics preference controls column statistics collection and histogram creation. 
- `DEGREE` determines degree of parallelism used for gathering statistics.

## Step 10: View Advisor Tasks 

Format the output in SQL*Plus.

````
column task_name format a32
column description format a28
column advisor_name format a28
column execution_start format a18
column execution_end format a18
column status format a12
````

`DBA_ADVISOR_TASKS` displays information about all tasks in the database. The view contains one row for each task. Each task has a name that is unique to the owner. We are interested in the `STATUS` of the task named `SYS_AUTO_INDEX_TASK`. This `STATUS` can be:

- INITIAL - Initial state of the task; no recommendations are present.
- EXECUTING - Task is currently running.
- INTERRUPTED - Task analysis was interrupted by the user. Recommendation data, if present, can be viewed and reported at this time.
- COMPLETED - Task successfully completed the analysis operation. Recommendation data can be viewed and reported.
- ERROR - An error occurred during the analysis operation. Recommendations, if present, can be viewed and reported at this time.

Run the following query:

````
select TASK_NAME,DESCRIPTION, ADVISOR_NAME, EXECUTION_START, STATUS from DBA_ADVISOR_TASKS;
````

`DBA_ADVISOR_EXECUTIONS` displays metadata information for task executions.

````
select to_char(execution_start, 'DD-MM-YY HH24:MI:SS') execution_start,
         to_char(execution_end, 'DD-MM-YY HH24:MI:SS') execution_end,
         status 
    from dba_advisor_executions where task_name='SYS_AUTO_INDEX_TASK';
````

For this query, we filter the information in `DBA_ADVISOR_EXECUTIONS` by `TASK_NAME`, to display only the rows corresponding to `SYS_AUTO_INDEX_TASK`. Automatic Indexing task, `SYS_AUTO_INDEX_TASK`, creates a new execution for each tuning run,  and it runs by default every 15 minutes. 

>**Note** : Re-execute these two queries until you get at least one execution, and write down the time when the last execution or automatic indexing task started.

````
TASK_NAME           DESCRIPTION  ADVISOR_NAME        EXECUTION_START  STATUS
------------------- ------------ ------------------- ---------------- ---------
SYS_AUTO_INDEX_TASK              SQL Access Advisor	27-JUN-19        COMPLETED

EXECUTION_START    EXECUTION_END      STATUS
------------------ ------------------ ------------
27-06-19 16:03:34  27-06-19 16:03:36  COMPLETED
````
 
## Step 11: Generate Database Workload

Automatic indexing improves database performance by managing indexes automatically and dynamically in an Oracle database based on changes in the application workload. This is why we need more workload on your pluggable database, to simulate changes in the application workload. This synthetic workload is generated using SH schema.

````
conn sh/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name>
````

`SERVEROUTPUT` controls whether to display the output. The `TIMING` command is used to see the elapsed time for individual queries in SQL*Plus. These settings are useful to understand the workload we generate.

````
set serveroutput on
set timing on
````

Queries are executed multiple times, in a `FOR LOOP` cycle.

````
begin
for i in 1..20 loop
    FOR sales_data IN (
SELECT ch.channel_class, c.cust_city, t.calendar_quarter_desc, SUM(s.amount_sold) sales_amount 
  FROM sh.sales s, sh.times t, sh.customers c, sh.channels ch 
  WHERE s.time_id = t.time_id AND s.cust_id = c.cust_id AND s.channel_id = ch.channel_id 
        AND c.cust_state_province = 'CA' AND ch.channel_desc in ('Internet','Catalog') 
        AND t.calendar_quarter_desc IN ('1999-01','1999-02') 
  GROUP BY ch.channel_class, c.cust_city, t.calendar_quarter_desc)
    LOOP  
      DBMS_OUTPUT.PUT_LINE('Sales data: ' || sales_data.channel_class || ' ' || sales_data.cust_city || 
                  ' '  || sales_data.calendar_quarter_desc || ' ' || sales_data.sales_amount);
    END LOOP;
end loop;
end;
/
````

These are simple queries we can find in any OLTP application.

````
begin
for i in 1..20 loop
    FOR sales_data IN (
SELECT ch.channel_class, c.cust_city, t.calendar_quarter_desc, SUM(s.amount_sold) sales_amount 
  FROM sh.sales s, sh.times t, sh.customers c, sh.channels ch 
  WHERE s.time_id = t.time_id AND s.cust_id = c.cust_id AND s.channel_id = ch.channel_id 
        AND c.cust_state_province = 'CA' AND ch.channel_desc in ('Internet','Catalog') 
        AND t.calendar_quarter_desc IN ('1999-03','1999-04') 
  GROUP BY ch.channel_class, c.cust_city, t.calendar_quarter_desc)
    LOOP  
      DBMS_OUTPUT.PUT_LINE('Sales data: ' || sales_data.channel_class || ' ' || sales_data.cust_city || 
                  ' '  || sales_data.calendar_quarter_desc || ' ' || sales_data.sales_amount);
    END LOOP;
end loop;
end;
/
````

Run these code blocks one by one in SQL*Plus.

````
begin
for i in 1..20 loop
    FOR sales_data IN (
SELECT c.country_id, c.cust_city, c.cust_last_name 
  FROM sh.customers c 
  WHERE c.country_id in (52790, 52798) 
  ORDER BY c.country_id, c.cust_city, c.cust_last_name)
    LOOP  
      DBMS_OUTPUT.PUT_LINE('Sales data: ' || sales_data.country_id || ' ' || sales_data.cust_city || 
                  ' '  || sales_data.cust_last_name);
    END LOOP;
end loop;
end;
/
````

There are also some queries on the new tables we created on top of the original SH sample data.

````
begin
for i in 1..20 loop
    FOR sales_data IN (
select /* func_indx */ count(*) howmany from tempjfv where abs(c)=5)
    LOOP  
      DBMS_OUTPUT.PUT_LINE('Sales data: ' || sales_data.howmany);
    END LOOP;
end loop;
end;
/
````

These tables will be also used to identify candidate indexes, and we gathered optimizer statistics after their creation.

````
begin
for i in 1..20 loop
    FOR sales_data IN (
SELECT * FROM customersjfv WHERE cust_state_province = 'CA')
    LOOP  
      DBMS_OUTPUT.PUT_LINE('Sales data: ' || sales_data.CUST_FIRST_NAME || ' ' || sales_data.CUST_LAST_NAME || 
                  ' '  || sales_data.CUST_EMAIL);
    END LOOP;
end loop;
end;
/ 
````

Please make sure these queries execute successfully on the SH schema, and no errors are returned.

## Step 12: Run Analytics Queries

In addition to the OLTP synthetic workload, we can generate also some advanced analytical SQL workload. In this section we want to verify that automatic indexing is also capable of handling advanced business intelligence queries. For these queries, use SQL Developer, available on the remote desktop connection. Create and save a new database connection as user SH. 

Here is an example of advanced analytical SQL statement. This query returns the percent change in market share for a grouping of SH top 20% of products for the current three-month period versus same period one year ago for accounts that grew by more than 20 percent in revenue.

````
WITH prod_list AS ( SELECT prod_id prod_subset, cume_dist_prod FROM ( SELECT s.prod_id, SUM(amount_sold),
         CUME_DIST() OVER (ORDER BY SUM(amount_sold)) cume_dist_prod
      FROM sales s, customers c, channels ch,  products p, times t
      WHERE s.prod_id = p.prod_id AND p.prod_total_id = 1 AND
            s.channel_id = ch.channel_id AND ch.channel_total_id = 1 AND
            s.cust_id = c.cust_id AND
            s.promo_id = 999 AND
            s.time_id  = t.time_id AND t.calendar_quarter_id = 1776 AND
            c.cust_city_id IN
       (SELECT cust_city_id FROM
         (SELECT cust_city_id, ((new_cust_sales - old_cust_sales) / old_cust_sales ) pct_change, old_cust_sales
           FROM
           (SELECT cust_city_id, new_cust_sales, old_cust_sales
              FROM
              ( SELECT cust_city_id,
                  SUM(CASE WHEN t.calendar_quarter_id = 1776
                     THEN amount_sold  ELSE  0  END ) new_cust_sales,
                  SUM(CASE WHEN t.calendar_quarter_id = 1772
                     THEN amount_sold ELSE 0 END) old_cust_sales
                FROM sales s, customers c, channels ch,
                     products p, times t
                WHERE s.prod_id = p.prod_id AND p.prod_total_id = 1 AND
                      s.channel_id = ch.channel_id AND ch.channel_total_id = 1 AND
                      s.cust_id = c.cust_id AND c.country_id = 52790 AND
                      s.promo_id = 999 AND
                      s.time_id  = t.time_id AND
                     (t.calendar_quarter_id = 1776 OR t.calendar_quarter_id =1772)
                GROUP BY cust_city_id) cust_sales_wzeroes
              WHERE old_cust_sales > 0)  cust_sales_woutzeroes)
         WHERE old_cust_sales > 0 AND  pct_change >= 0.20)
GROUP BY s.prod_id )  prod_sales                    
    WHERE cume_dist_prod > 0.8 ) SELECT  prod_id, ( (new_subset_sales/new_tot_sales) - (old_subset_sales/old_tot_sales) ) *100  share_changes
FROM ( SELECT  prod_id,
     SUM(CASE WHEN t.calendar_quarter_id = 1776
                   THEN amount_sold  ELSE  0  END )  new_subset_sales,
        (SELECT SUM(amount_sold) FROM sales s, times t, channels ch,
                customers c, countries co, products p
          WHERE s.time_id  = t.time_id AND t.calendar_quarter_id = 1776 AND
                s.channel_id = ch.channel_id AND ch.channel_total_id = 1 AND
                s.cust_id = c.cust_id AND 
                c.country_id = co.country_id AND co.country_total_id = 52806 AND
                s.prod_id = p.prod_id AND p.prod_total_id = 1 AND
                s.promo_id = 999
        )   new_tot_sales,
     SUM(CASE WHEN t.calendar_quarter_id = 1772
                   THEN amount_sold  ELSE  0  END)  old_subset_sales,
        (SELECT SUM(amount_sold) FROM sales s, times t, channels ch, 
                customers c, countries co, products p
          WHERE s.time_id  = t.time_id AND t.calendar_quarter_id = 1772 AND
                s.channel_id = ch.channel_id AND ch.channel_total_id = 1 AND
                s.cust_id = c.cust_id AND
                c.country_id = co.country_id AND co.country_total_id = 52806 AND
                       s.prod_id = p.prod_id AND p.prod_total_id = 1 AND
                s.promo_id = 999
        )   old_tot_sales
 FROM sales s, customers c, countries co, channels ch, times t
 WHERE s.channel_id = ch.channel_id AND ch.channel_total_id = 1 AND
       s.cust_id = c.cust_id AND
       c.country_id = co.country_id AND co.country_total_id = 52806 AND
       s.promo_id = 999 AND
       s.time_id  = t.time_id AND 
      (t.calendar_quarter_id = 1776 OR t.calendar_quarter_id = 1772)
         AND s.prod_id IN
     (SELECT prod_subset FROM prod_list)
  GROUP BY prod_id);
````

The way this example proceeds is to begin by creating a reference table of currency conversion factors.

````
CREATE TABLE currency (
   country         VARCHAR2(20),
   year            NUMBER,
   month           NUMBER,
   to_us           NUMBER);
````

The table will hold conversion factors for each month for each country.

````
INSERT INTO currency
(SELECT distinct
SUBSTR(country_name,1,20), calendar_year, calendar_month_number, 1
FROM countries
CROSS JOIN times t
WHERE calendar_year IN (2000,2001,2002)
);
````

For our purposes, we only set the conversion factor for one country, Canada.

````
UPDATE currency set to_us=.74 WHERE country='Canada';
````

This query projects sales for 2002 based on the sales of 2000 and 2001. It finds the most percentage changes in sales from 2000 to 2001 and then adds that to the sales of 2002. The first subclause finds the monthly sales per product by country for the years 2000, 2001, and 2002. The second subclause finds a list of distinct times at the month level.

````
WITH  prod_sales_mo AS ( SELECT country_name c, prod_id p, calendar_year  y,
   calendar_month_number  m, SUM(amount_sold) s
FROM sales s, customers c, times t, countries cn, promotions p, channels ch
WHERE  s.promo_id = p.promo_id AND p.promo_total_id = 1 AND
       s.channel_id = ch.channel_id AND ch.channel_total_id = 1 AND
       s.cust_id=c.cust_id  AND 
       c.country_id=cn.country_id AND country_name='France' AND
       s.time_id=t.time_id  AND t.calendar_year IN  (2000, 2001,2002)
GROUP BY cn.country_name, prod_id, calendar_year, calendar_month_number
), time_summary AS(  SELECT DISTINCT calendar_year cal_y, calendar_month_number cal_m
  FROM times
  WHERE  calendar_year IN  (2000, 2001, 2002) )
SELECT c, p, y, m, s,  nr FROM (
SELECT c, p, y, m, s,  nr
FROM prod_sales_mo s
  PARTITION BY (s.c, s.p)
     RIGHT OUTER JOIN time_summary ts ON
        (s.m = ts.cal_m
         AND s.y = ts.cal_y )
MODEL REFERENCE curr_conversion ON
      (SELECT country, year, month, to_us
      FROM currency)
      DIMENSION BY (country, year y,month m) MEASURES (to_us)
   PARTITION BY (s.c c)
   DIMENSION BY (s.p p, ts.cal_y y, ts.cal_m m)
   MEASURES (s.s s, CAST(NULL AS NUMBER) nr, s.c cc )
   RULES ( nr[ANY, ANY, ANY]
         = CASE WHEN s[CV(), CV(), CV()] IS NOT NULL
              THEN s[CV(), CV(), CV()]
              ELSE ROUND(AVG(s)[CV(), CV(), m BETWEEN 1 AND 12],2)
           END, nr[ANY, 2002, ANY] = ROUND(
         ((nr[CV(),2001,CV()] - nr[CV(),2000, CV()])
          / nr[CV(),2000, CV()]) * nr[CV(),2001, CV()]
         + nr[CV(),2001, CV()],2),
      nr[ANY,y != 2002,ANY]
         = ROUND(nr[CV(),CV(),CV()]
           * curr_conversion.to_us[ cc[CV(),CV(),CV()], CV(y), CV(m)], 2) )
ORDER BY c, p, y, m)
WHERE y = '2002'
ORDER BY c, p, y, m;
````

Make sure these statements run successfully. When finished, close SQL Developer, and commit changes when asked. 

## Step 13: Verify Automatic Indexing Results

Now it is time to see what automatic indexing has prepared for us, to help improve the performance of our queries. For this section, use the maximized Terminal window, connecting with SYSDBA privileges on your pluggable database.

````
conn sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name> as SYSDBA
````

First step is to query `DBA_ADVISOR_EXECUTIONS` for information about automatic indexing task executions.

````
select to_char(execution_start, 'DD-MM-YY HH24:MI:SS') execution_start,
         to_char(execution_end, 'DD-MM-YY HH24:MI:SS') execution_end,
         status 
    from dba_advisor_executions where task_name='SYS_AUTO_INDEX_TASK';
````

>**Note** : Wait and make sure another automatic indexing task was executed since the last time we queried this view, by comparing the starting time of the last entry with the starting time we saved in the previous section.

````
EXECUTION_START    EXECUTION_END      STATUS
------------------ ------------------ ------------
27-06-19 16:03:34  27-06-19 16:03:36  COMPLETED
27-06-19 16:18:38  27-06-19 16:18:53  COMPLETED 
````

Automatic indexing process runs in the background periodically at a predefined time interval, default is 15 minutes. It analyzes application workload, and accordingly creates new indexes and drops the existing underperforming indexes to improve database performance. It also rebuilds the indexes that are marked unusable due to table partitioning maintenance operations, such as `ALTER TABLE MOVE`.

Count all `VALID` and `VISIBLE` indexes in the SH schema, that are auto indexes (`AUTO = ‘YES’`). `AUTO` column is available starting with Oracle Database release 19c, version 19.1.

````
select index_name, index_type, status, visibility
    from dba_indexes
    where auto = 'YES'
      and visibility = 'VISIBLE'
      and status = 'VALID'
      and table_owner = 'SH';
````

If everything was executed accurately following this guide, there are new auto indexes in our SH schema. 

>**Note** : If there are no valid visible new indexes, please run the workload again as SH user, and re-run the previous task as SYSDBA. Copy all queries from Step 11 and Step 12 into SQL Developer and use Run Script command to run them multiple times.

## Step 14: List Automatic Indexes 

Format the output in SQL*Plus, for a better view of the information listed.

````
set linesize 120
column table_name format a30
column  index_name format a30
column  index_type format a25
column  last_analyzed format a25 
````

All indexes, including the new auto indexes, are listed using the following query.

````
select TABLE_NAME, INDEX_NAME, INDEX_TYPE, LAST_ANALYZED from DBA_INDEXES where TABLE_OWNER = 'SH';
````

Can you identify the auto indexes for the list? Please observe the tables on which these indexes have been created, and what types.

As we have noticed already, automatic indexing provides PL/SQL APIs for configuring automatic indexing in a database and generating reports related to automatic indexing operations.

````
set linesize 120 trims on pagesize 1000 long 100000
column REPORT format a120
````

Run again the activity report with the same parameters.

````
select DBMS_AUTO_INDEX.REPORT_ACTIVITY(sysdate-30, NULL, 'text', 'all', 'all') REPORT from DUAL;
````

Here is a sample output of the activity report. From this report we can understand more about the auto indexes, including the tables on which they were created, and the index types selected.

````
REPORT
-------------------------------------------------------------------------------
GENERAL INFORMATION
-------------------------------------------------------------------------------
 Activity start 	      : 20-MAY-2019 15:01:30
 Activity end		      : 19-JUN-2019 15:01:30
 Executions completed	      : 115
 Executions interrupted       : 0
 Executions with fatal error  : 2
-------------------------------------------------------------------------------

SUMMARY (AUTO INDEXES)
-------------------------------------------------------------------------------
 Index candidates			       : 18
 Indexes created (visible / invisible)	       : 9 (5 / 4)
 Space used (visible / invisible)	       : 85 MB (6.49 MB / 78.51 MB)
 Indexes dropped			       : 0
 SQL statements verified		       : 34
 SQL statements improved (improvement factor)  : 4 (2.7x)
 SQL plan baselines created		       : 0
 Overall improvement factor		       : 1.8x
-------------------------------------------------------------------------------

SUMMARY (MANUAL INDEXES)
-------------------------------------------------------------------------------
 Unused indexes    : 0
 Space used	   : 0 B
 Unusable indexes  : 0
-------------------------------------------------------------------------------

INDEX DETAILS
-------------------------------------------------------------------------------
1. The following indexes were created:
*: invisible
-------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------
| Owner   | Table        | Index                  | Key                           | Type   | Properties|
--------------------------------------------------------------------------------------------------------
| AUTOIDX | CUSTOMERSJFV | * SYS_AI_cj7tna2ack2r6 | CUST_STATE_PROVINCE           | B-TREE | NONE      |
| SH      | CUSTOMERS    | SYS_AI_8urpdk6h1ujkh   | COUNTRY_ID                    | B-TREE | NONE      |
| SH      | CUSTOMERS    | SYS_AI_92r3r69avtpnj   | CUST_STATE_PROVINCE           | B-TREE | NONE      |
| SH      | CUSTOMERS    | SYS_AI_fz1fyu74a1f87   | CUST_CITY_ID                  | B-TREE | NONE      |
| SH      | SALES        | * SYS_AI_5bcpzx4s8ts9j | CUST_ID                       | B-TREE | LOCAL     |
| SH      | SALES        | * SYS_AI_fkvdkj56j2q0a | TIME_ID,CHANNEL_ID,CUST_ID    | B-TREE | LOCAL     |
| SH      | TIMES        | SYS_AI_7gxc48px5b52h   | CALENDAR_QUARTER_ID           | B-TREE | NONE      |
| SH      | TIMES        | * SYS_AI_829bv4n435ysa | TIME_ID,CALENDAR_QUARTER_DESC | B-TREE | NONE      |
| SH      | TIMES        | SYS_AI_c45r9rthxz31w   | CALENDAR_QUARTER_DESC         | B-TREE | NONE      |
--------------------------------------------------------------------------------------------------------
-------------------------------------------------------------------------------

VERIFICATION DETAILS
...
````

In this example report, automatic indexing identified 18 candidates, and created 9 indexes, 5 visible and 4 invisible.

Auto index candidates are identified based on the usage of table columns in SQL statements. The auto index candidates are created as invisible auto indexes, that is, these auto indexes cannot be used in SQL statements. The invisible auto indexes are validated against SQL statements.

If the performance of SQL statements is improved by using these indexes, then the indexes are configured as visible indexes, so that they can be used in SQL statements.

If the performance of SQL statements is not improved by using these indexes, then the indexes are configured as unusable indexes and the SQL statements are blacklisted. The unusable indexes are later deleted by the automatic indexing process. The blacklisted SQL statements are not allowed to use auto indexes in future.

The auto indexes that are not used for a long period are dropped. No indexes were dropped in this example report. 
Please review also the information in the Original Plan vs. Auto Index Plan comparison section. This comparison is performed using different metrics like Elapsed Time (s), CPU Time (s), or Optimizer Cost.

## Acknowledgements

- **Author** - Valentin Leonard Tabacaru
- **Last Updated By/Date** - Valentin Leonard Tabacaru, Principal Product Manager, DB Product Management, Aug 2020

See an issue? Please open up a request [here](https://github.com/oracle/learning-library/issues). Please include the workshop name and lab in your request.
