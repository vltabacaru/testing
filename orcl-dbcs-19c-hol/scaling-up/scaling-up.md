# Performance, Scalability and Elasticity

## Introduction

Oracle Database technology used for the Database Cloud Service is the standard for scalability, robustness and enterprise strength. Virtual Machine VM.Standard2 Shapes provide a virtual machine DB system database provisioned on X7 machines, with six VM options available, having 1 to 24 CPU cores and 15 GB to 320 GB memory. You can scale up or down the number of OCPUs on the instance by changing the shape the DB System is running on.

Oracle Database on virtual machines uses remote block storage, and enables scaling storage from 256 GB to 40 TB, with no downtime when scaling up storage. You specify a storage size when you launch the DB system, and you can scale up the storage as needed at any time. You can easily scale up storage for a DB system by using the console, REST APIs, CLI and SDKs. 

>**Note** : The total storage attached to an instance will be a sum of available storage, reco storage, and software size. Available storage is selected by the customer, reco storage is automatically calculated based on available storage, and software size is a fixed size Oracle database cost.

## Step 1: Check CPU Resources

Connect to the Compute node using SSH.

````
ssh -C -i id_rsa opc@<Compute Public IP Address>
````

Connect to your DB System database as SYSDBA using SQL*Plus.

````
sqlplus sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name> as sysdba
````

Display the value of parameter **cpu_count**. Database Service is running currently on 2 CPUs.

````
show parameter cpu_count

NAME				     TYPE	 VALUE
------------------------------------ ----------- --------------------
cpu_count			     integer	 2
````

Connect to your pluggable database as SH user.

````
conn sh/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name>
````

Run a test query, and write down the time it takes to complete.

````
set timing on
set linesize 130

SELECT
  a.cust_id,
  a.cust_last_name || ', ' || a.cust_first_name as customer_name,
  a.cust_city || ', ' || a.cust_state_province || ', ' || a.country_id as city_id,
  a.cust_city as city_name,
  a.cust_state_province || ', ' || a.country_id as state_province_id,
  a.cust_state_province as state_province_name,
  b.country_id,
  b.country_name,
  b.country_subregion as subregion,
  b.country_region as region
FROM sh.customers a, sh.countries b
where a.country_id = b.country_id;

55500 rows selected.

Elapsed: 00:00:28.67
````

Exit SQL*Plus.

````
exit
````

## Step 2: Change Shape for More CPUs

On Oracle cloud console, click on hamburger menu ≡, then **Bare Metal, VM, and Exadata** under Databases. Click **[Your Initials]-DB** DB System.

On DB System Details page, click **Change Shape** button. Select VM.Standard2.2 shape. Click **Change Shape** to confirm.

Read the warning: *Changing shapes stops a running DB System and restarts it on the selected shape.
Are you sure you want to change the shape from VM.Standard2.1 to VM.Standard2.2?*
Click again **Change Shape** to confirm.

DB System Status will change to Updating... Wait for Status to become Available. Connect to your DB System database as SYSDBA using SQL*Plus.

````
sqlplus sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name> as sysdba
````

Display again the value of parameter **cpu_count**. Database Service is running now on 4 CPUs.

````
show parameter cpu_count

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
cpu_count			     integer	 4
````

After restarting the DB System on a new shape, your pluggable database is in mounted state. Open pluggable database PDB012.

````
alter pluggable database PDB012 open;
````

Connect to your pluggable database as SH user.

````
conn sh/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb012.<Host Domain Name>
````

Run again the test query, and compare the time it takes to complete with the previous run.

````
set timing on
set linesize 130

SELECT
  a.cust_id,
  a.cust_last_name || ', ' || a.cust_first_name as customer_name,
  a.cust_city || ', ' || a.cust_state_province || ', ' || a.country_id as city_id,
  a.cust_city as city_name,
  a.cust_state_province || ', ' || a.country_id as state_province_id,
  a.cust_state_province as state_province_name,
  b.country_id,
  b.country_name,
  b.country_subregion as subregion,
  b.country_region as region
FROM sh.customers a, sh.countries b
where a.country_id = b.country_id;

55500 rows selected.

Elapsed: 00:00:11.23
````

Type **exit** command two times followed by Enter to close all sessions (SQL*Plus, and SSH).

````
exit

exit
````

## Step 3: Scale Up Storage Volumes

On Oracle cloud console, click on hamburger menu ≡, then **Bare Metal, VM, and Exadata** under Databases. Click **[Your Initials]-DB** DB System.

Under DB System Information review the allocated storage resources:

- Available Data Storage: 256 GB
- Total Storage Size: 712 GB

Connect to the Database node using SSH.

````
ssh -C -i Keys/KeysDry/id_rsa opc@<DB Node Public IP Address>
````

Check file system disk space usage on the database node, using the **df** command. Flag **-h** is fo *human-readable* output. It uses unit suffixes: Byte, Kilobyte, Megabyte, and so on. Total Storage Size value on DB System Information is the sum of all these filesystems. Write down in your notes the value of **/dev/mapper/DATA_GRP-DATA** filesystem.

````
df -h

Filesystem                           Size  Used Avail Use% Mounted on
devtmpfs                             7.2G     0  7.2G   0% /dev
tmpfs                                7.3G     0  7.3G   0% /dev/shm
tmpfs                                7.3G  193M  7.1G   3% /run
tmpfs                                7.3G     0  7.3G   0% /sys/fs/cgroup
/dev/mapper/VolGroupSys0-LogVolRoot   45G  8.7G   34G  21% /
tmpfs                                7.3G  4.3M  7.3G   1% /tmp
/dev/sda2                            1.4G  102M  1.2G   8% /boot
/dev/sda1                            486M  9.7M  476M   2% /boot/efi
/dev/mapper/DATA_GRP-DATA            252G   21G  219G   9% /u02
/dev/mapper/RECO_GRP-RECO            252G   23G  217G  10% /u03
/dev/mapper/BITS_GRP-BITS            197G   14G  174G   7% /u01
tmpfs                                1.5G     0  1.5G   0% /run/user/101
tmpfs                                1.5G     0  1.5G   0% /run/user/54322
````

On Oracle cloud console, click on hamburger menu ≡, then **Bare Metal, VM, and Exadata** under Databases. Click **[Your Initials]-DB** DB System.

On DB System Details page, click **Scale Storage Up** button. Set Available Data Storage (GB): 512, and click **Update**.

DB System Status will change to Updating... Wait for Status to become Available.

On Oracle cloud console, under DB System Information, review again the allocated storage resources:

- Available Data Storage: 512 GB
- Total Storage Size: 968 GB

Check again the file system disk space usage, and compare the value of **/dev/mapper/DATA_GRP-DATA** filesystem with the previous one.

````
df -h

Filesystem                           Size  Used Avail Use% Mounted on
devtmpfs                             7.2G     0  7.2G   0% /dev
tmpfs                                7.3G     0  7.3G   0% /dev/shm
tmpfs                                7.3G  193M  7.1G   3% /run
tmpfs                                7.3G     0  7.3G   0% /sys/fs/cgroup
/dev/mapper/VolGroupSys0-LogVolRoot   45G  8.7G   34G  21% /
tmpfs                                7.3G  4.3M  7.3G   1% /tmp
/dev/sda2                            1.4G  102M  1.2G   8% /boot
/dev/sda1                            486M  9.7M  476M   2% /boot/efi
/dev/mapper/DATA_GRP-DATA            504G   21G  460G   5% /u02
/dev/mapper/RECO_GRP-RECO            252G   23G  217G  10% /u03
/dev/mapper/BITS_GRP-BITS            197G   14G  174G   7% /u01
tmpfs                                1.5G     0  1.5G   0% /run/user/101
tmpfs                                1.5G     0  1.5G   0% /run/user/54322
````

>**Note** : You noticed the button on Oracle cloud console is called **Scale Storage Up**. Because you can always scale up, in other words increase, the database service storage, and never scale down, or decrease.

## Acknowledgements

- **Author** - Valentin Leonard Tabacaru
- **Last Updated By/Date** - Valentin Leonard Tabacaru, Principal Product Manager, DB Product Management, Aug 2020

See an issue? Please open up a request [here](https://github.com/oracle/learning-library/issues). Please include the workshop name and lab in your request.

