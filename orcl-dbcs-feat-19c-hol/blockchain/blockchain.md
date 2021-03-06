# Oracle 20c Preview Blockchain Tables

## Introduction

Blockchain tables are append-only tables in which only insert operations are allowed. Deleting rows is either prohibited or restricted based on time. Rows in a blockchain table are made tamper-resistant by special sequencing & chaining algorithms. Users can verify that rows have not been tampered. A hash value that is part of the row metadata is used to chain and validate rows.

Blockchain tables enable you to implement a centralized ledger model where all participants in the blockchain network have access to the same tamper-resistant ledger.

A centralized ledger model reduces administrative overheads of setting a up a decentralized ledger network, leads to a relatively lower latency compared to decentralized ledgers, enhances developer productivity, reduces the time to market, and leads to significant savings for the organization. Database users can continue to use the same tools and practices that they would use for other database application development.

## Step 1: Provision Oracle 20c Preview DB System

On the Oracle Cloud console, click on hamburger menu ≡, then **Bare Metal, VM, and Exadata** under Databases. **Create DB System**.

- Select a compartment: [Your Compartment]
- Name your DB system: [Your Initials]-20DB (e.g. VLT-20DB)
- Select a shape type: Virtual Machine
- Select a shape: VM.Standard2.1
- Choose Storage Management Software: Logical Volume Manager
- Add public SSH keys: Upload SSH key files > id_rsa.pub
- Choose a license type: Bring Your Own License (BYOL)

Specify the network information.

- Virtual cloud network: [Your Initials]-VCN
- Client Subnet: Public Subnet
- Hostname prefix: [Your Initials]-20host (small case, e.g. vlt-20host)

Next.

- Database name: [Your Initials]20DB (e.g. VLT20DB)
- Database version: 20c (Preview)
- PDB name: PDB011
- Password: DBlearnPTS#20_
- Select workload type: Transaction Processing

**Create DB System**.

## Step 2: Connect to Oracle 20c Preview Database

Wait for DB System to finish provisioning, and have status Available. Click on hamburger menu ≡, then **Bare Metal, VM, and Exadata**. Click **[Your Initials]-20DB** DB System. On the DB System Details page, copy Host Domain Name in your notes. In the table below, copy Database Unique Name in your notes. Click **Nodes** on the left menu, and copy Public IP Address in your notes.

The rest of the lab will be executed in SQL*Plus, using SSH connection to your 20c Preview DB System. 

Connect to the 20c Preview Database node using SSH.

````
ssh -C -i id_rsa opc@<20c DB Node Public IP Address>
````

Use the substitute user command to start a session as **oracle** user.

````
sudo su - oracle
````

Blockchain tables cannot be created in the root container and in an application root container, therefore you will use the pluggable database called PDB011.

````
sqlplus system/DBlearnPTS#20_@[Your Initials]-20host:1521/pdb011.<Host Domain Name>
````

## Step 3: Create Database User

Database users require no special privileges to create and work with blockchain tables or JSON data type. A simple user will do the job just fine. As a best practice, you can create a specific tablespace for storing JSON documents, but for this simple case we will use the existing one.

````
create user ooe identified by "DBlearnPTS#20_" default tablespace USERS quota unlimited on USERS;
````

````
GRANT connect, resource TO ooe;
````

The rest of the scenario is executed by OOE user we just created.

````
conn ooe/DBlearnPTS#20_@[Your Initials]-20host:1521/pdb011.<Host Domain Name>
````

Some formatting for SQL*Plus will help me understand the output better. Hit Enter one more time after pasting these lines.

````
set linesize 130
set serveroutput on
set pages 9999
set long 90000
column table_name format a40
column order_doc format a40
column name format a40
column shipTo format a40
````

## Step 4: Create Blockchain Table

Let's say we got bored of the old fashion relational table orders. We can create a new orders table, that is smart, secure, and cool at the same time. This table uses the new identity column type for the primary key (new in 12c), native JSON data type that allows our application to make changes in the order document at any time (e.g. adding new fields, separating billing from shipping addresses, etc.), and a special sequencing & algorithm (SHA-2 512-bit cryptographic hash) used to chain and validate rows making it tamper-resistant.

- `NO DROP` is used to specify the retention period for the table after no new inserts;
- `NO DELETE` is used to set the retention period for rows;
- `HASHING USING`, and `VERSION` clauses are mandatory in a `CREATE BLOCKCHAIN TABLE` statement. 

In our case, orders blockchain table cannot be dropped if the newest row is less than 60 days old, and rows cannot be deleted until 16 days after the most recent row was added.

````
CREATE BLOCKCHAIN TABLE orders
( order_id NUMBER GENERATED BY DEFAULT ON NULL AS IDENTITY
    MINVALUE 1 MAXVALUE 9999999999999999999999999999
    INCREMENT BY 1 START WITH 1 CACHE 20 NOT NULL ENABLE,
  order_doc JSON,
  created DATE,
  created_by VARCHAR2(20),
  currency VARCHAR2(3),
  channel VARCHAR2(20),
  CONSTRAINT orders_pk PRIMARY KEY (order_id)
    USING INDEX ENABLE
)
NO DROP UNTIL 60 DAYS IDLE
NO DELETE UNTIL 16 DAYS AFTER INSERT
HASHING USING "SHA2_512" VERSION "v1";
````

On top of that, this table saves the date when each order is created, and the database user that created that order.

````
CREATE OR REPLACE EDITIONABLE TRIGGER orders_bi
    before insert on orders
    for each row
begin
    :new.created := SYSDATE;
    :new.created_by := SYS_CONTEXT('USERENV','SESSION_USER');
    if :new.currency is null then
        :new.currency := 'USD';
    end if;
end;
/
````

````
ALTER TRIGGER orders_bi ENABLE;
````

## Step 5: View Blockchain Table Details

Describe the blockchain tables owned by the current OOE user with this query.

````
SELECT * FROM USER_BLOCKCHAIN_TABLES;

TABLE_NAME       ROW_RETENTION ROW TABLE_INACTIVITY_RETENTION HASH_ALG
---------------- ------------- --- -------------------------- --------
ORDERS                      16 NO                          60 SHA2_512
````

We can modify, more exactly increase, the retention period for a blockchain table and for rows within a blockchain table, but never decrease. Modify the definition of orders blockchain table so rows cannot be deleted until 31 days after they were created. By adding a `LOCKED` clause we indicate that this setting can never be modified.

````
ALTER TABLE orders NO DELETE UNTIL 31 DAYS AFTER INSERT LOCKED;
````

Review again blockchain tables details.

````
SELECT * FROM USER_BLOCKCHAIN_TABLES;

TABLE_NAME       ROW_RETENTION ROW TABLE_INACTIVITY_RETENTION HASH_ALG
---------------- ------------- --- -------------------------- --------
ORDERS                      31 YES                         60 SHA2_512
````

## Step 6: Insert Records in Blockchain Table

JSON native data type automatically validates the format, and does not require a IS JSON check constraint like in previous Oracle versions.

````
INSERT INTO orders (order_doc) VALUES ('++{} "This is not a valid", "JSON" ::; "document !"');

ORA-40441: JSON syntax error
````

Insert some valid rows into the table.

````
INSERT INTO orders (order_doc, currency) VALUES (
'{ "name"  : "Joe Bravo",
  "sku"    : "5",
  "price"  : 23.95,
  "shipTo" : { "name" : "Eva Bravo",
              "address" : "Via Calimala, 23",
              "city" : "Firenze",
              "country" : "IT",
              "zip"  : "50123" },
  "billTo" : { "name" : "Joe Bravo",
              "address" : "Via Calimala, 23",
              "city" : "Firenze",
              "country" : "IT",
              "zip"  : "50123" }
}',
'EUR');
````

And another valid row.

````
INSERT INTO orders (order_doc, channel) VALUES (
'{ "name"  : "Carmen Flores",
  "sku"    : "8",
  "price"  : 199.95,
  "shipTo" : { "name" : "Mario Flores",
              "address" : "Gran Via, 25",
              "city" : "Salamanca",
              "country" : "ES",
              "zip"  : "37001" },
  "billTo" : { "name" : "Carmen Flores",
              "address" : "Gran Via, 25",
              "city" : "Salamanca",
              "country" : "ES",
              "zip"  : "37001" }
}',
'online');
````

JSON documents can be inserted in any shape as long as we keep the structure, we don't have to use a clear human readable shape.

````
INSERT INTO orders (order_doc, currency, channel) VALUES ('{ "name" : "Francis Picard", "sku" : "10", "price" : 65.95, "shipTo" : { "name" : "Maria Picard", "address" : "25 Avenue Jean Jaures", "city" : "Lyon", "country" : "FR", "zip"  : "69007" }, "billTo" : { "name" : "Francis Picard", "address" : "25 Avenue Jean Jaures", "city" : "Lyon", "country" : "FR", "zip"  : "69007" }}', 'EUR', 'direct');
````

Commit all three valid rows.

````
COMMIT;
````

## Step 7: Retrieve Records from Blockchain Table

As you can see, all JSON documents in the orders table have the same shape, even though we inserted the first two in a clear format, and the last one as a continuous string.

````
SELECT * FROM orders;

  ORDER_ID ORDER_DOC                                CREATED   CREATED_BY  CUR CHANNEL
---------- ---------------------------------------- --------- ----------- --- -------
         2 {"name":"Joe Bravo","sku":"5","price":23 04-JUN-20 OOE         EUR
           .95,"shipTo":{"name":"Eva Bravo","addres
           s":"Via Calimala, 23","city":"Firenze","
           country":"IT","zip":"50123"},"billTo":{"
           name":"Joe Bravo","address":"Via Calimal
           a, 23","city":"Firenze","country":"IT","
           zip":"50123"}}

         3 {"name":"Carmen Flores","sku":"8","price 04-JUN-20 OOE         USD online
           ":199.95,"shipTo":{"name":"Mario Flores"
           ,"address":"Gran Via, 25","city":"Salama
           nca","country":"ES","zip":"37001"},"bill
           To":{"name":"Carmen Flores","address":"G
           ran Via, 25","city":"Salamanca","country
           ":"ES","zip":"37001"}}

         4 {"name":"Francis Picard","sku":"10","pri 04-JUN-20 OOE         EUR direct
           ce":65.95,"shipTo":{"name":"Maria Picard
           ","address":"25 Avenue Jean Jaures","cit
           y":"Lyon","country":"FR","zip":"69007"},
           "billTo":{"name":"Francis Picard","addre
           ss":"25 Avenue Jean Jaures","city":"Lyon
           ","country":"FR","zip":"69007"}}
````

There are ways of retrieving the information in a clear human readable format.

````
SELECT o.order_id,
      JSON_SERIALIZE(o.order_doc PRETTY) order_doc,
      to_char(created, 'HH24:MM:SS DD-MON-YY') created
FROM orders o;

  ORDER_ID ORDER_DOC                                CREATED
---------- ---------------------------------------- ------------------
         2 {                                        18:06:48 04-JUN-20
             "name" : "Joe Bravo",
             "sku" : "5",   
             "price" : 23.95,
             "shipTo" :
             {
               "name" : "Eva Bravo",
               "address" : "Via Calimala, 23",
               "city" : "Firenze",
               "country" : "IT",
               "zip" : "50123"
             },
             "billTo" :
             {
               "name" : "Joe Bravo",
               "address" : "Via Calimala, 23",
               "city" : "Firenze",
               "country" : "IT",
               "zip" : "50123"
             }
           }

          3 {                                        18:06:59 04-JUN-20
             "name" : "Carmen Flores",
             "sku" : "8",
             "price" : 199.95,
             "shipTo" :
             {
               "name" : "Mario Flores",
               "address" : "Gran Via, 25",
               "city" : "Salamanca",
               "country" : "ES",
               "zip" : "37001"
              },
             "billTo" :
             {
               "name" : "Carmen Flores",
               "address" : "Gran Via, 25",
               "city" : "Salamanca",
               "country" : "ES",
               "zip" : "37001"
             }
           }

         4 {                                        18:06:11 04-JUN-20
             "name" : "Francis Picard",
             "sku" : "10",
              "price" : 65.95,
              "shipTo" :
             {
               "name" : "Maria Picard",
               "address" : "25 Avenue Jean Jaures",
               "city" : "Lyon",
                "country" : "FR",
               "zip" : "69007"
             },
             "billTo" :
             {
               "name" : "Francis Picard",
               "address" : "25 Avenue Jean Jaures",
               "city" : "Lyon",
               "country" : "FR",
               "zip" : "69007"
             }
           }
````

We can even retrieve individual fields from the orders stored as JSON documents.

````
SELECT o.order_id, o.order_doc."name", o.order_doc."shipTo"."address" FROM orders o;

  ORDER_ID name                                     shipTo
---------- ---------------------------------------- -----------------------
         2 "Joe Bravo"                              "Via Calimala, 23"
         3 "Carmen Flores"                          "Gran Via, 25"
         4 "Francis Picard"                         "25 Avenue Jean Jaures"
````

## Step 8: Permitted Operations on Blockchain Tables

However, we cannot update rows, delete rows, truncate or drop the blockchain table. We cannot even drop the tablespace containing a blockchain table (go ahead and try it, but connecting with SYSDBA privileges).

Try to delete rows from `ORDERS` blockchain table.

````
DELETE FROM orders WHERE order_id > 2;

ORA-05715: operation not allowed on the blockchain table
````

Try to truncate `ORDERS` blockchain table.

````
TRUNCATE TABLE orders;

ORA-05715: operation not allowed on the blockchain table
````

Try to update a row in `ORDERS` blockchain table.

````
UPDATE orders SET currency = 'GBP' WHERE order_id > 2;

ORA-05715: operation not allowed on the blockchain table
````

Try to drop `ORDERS` blockchain table.

````
DROP TABLE orders;

ORA-05723: drop blockchain table ORDERS not allowed
````

## Step 9: Blockchain Tables Hidden Columns

For management purposes it is important to understand the hidden columns in blockchain tables. Here are a few of them: 

- `ORABCTAB_INST_ID$` is the database instance ID;
- `ORABCTAB_CHAIN_ID$` represents the ID of the chain with values between 0 and 31;
- `ORABCTAB_USER_NUMBER$` is the ID of the database user who inserted the row;
- `ORABCTAB_HASH$` contains the hash value of the row computed based on current row content and hash value of the previous row.

Hash values on your table will be different from the ones in the example, because these are unique. Same may be true for the database instance ID and the ID of the database user in your database, these values ay have different values from the example.

````
SELECT o.order_id, o.order_doc."name", ORABCTAB_INST_ID$, ORABCTAB_CHAIN_ID$, ORABCTAB_USER_NUMBER$, ORABCTAB_HASH$ FROM orders o;

  ORDER_ID name                                     ORABCTAB_INST_ID$ ORABCTAB_CHAIN_ID$ ORABCTAB_USER_NUMBER$
---------- ---------------------------------------- ----------------- ------------------ ---------------------
ORABCTAB_HASH$
----------------------------------------------------------------------------------------------------------------------------------
         2 "Joe Bravo"                              1                 12                 113
ACC12150F672BA9E5491EADE2C290BA2EB4AD23B3F13D2C919015B0D2633CCCAC035241ED4019329DC445969B0B8488704F2A7DD08C090C0A454A1252E458553
         3 "Carmen Flores"                          1                 12                 113
93F697F51245F7AD4778328D891F04F1432896B244B915A803AFBD9178BDA6388AC1792CB2B64C631F11D419AB64620A40CB3D470CD54926BAD85B554A55BB03
         4 "Francis Picard"                         1                 12                 113
9018522AFA8E50A3CB477FD110F333F18EBBB1B46733E472ADCC8429D4170A3D6E3607B70722764C63B1E9D93876079EC718D5A064CCF91670CC9609EDB8DE36
````

It is easy to verify `ORABCTAB_USER_NUMBER$` for example.

````
select SYS_CONTEXT('USERENV','SESSION_USERID') from dual;

SYS_CONTEXT('USERENV','SESSION_USERID')
-----------------------------------------------------------------------
113
````

## Step 10: Validate Records in Blockchain Table

Last but not least, I want to mention the `DBMS_BLOCKCHAIN_TABLE` package that can be used to manage records in blockchain tables. Some of the tasks we can perform are: 

- Delete rows that are beyond the retention period;
- Sign a row you inserted after it is added to a chain; 
- Verify the hashes and signatures on rows. 

Here is a simple example of verifying or validating a row. Use the value of `ORABCTAB_INST_ID$` hidden column for **instance_id** variable.

````
DECLARE
      verify_rows NUMBER;
      instance_id NUMBER;
BEGIN
      instance_id := 1;
      DBMS_BLOCKCHAIN_TABLE.VERIFY_ROWS('OOE','ORDERS', NULL, NULL, instance_id, NULL, verify_rows);
      DBMS_OUTPUT.PUT_LINE(' Number of rows verified in instance Id '|| instance_id || ' = '|| verify_rows);
END;
/

 Number of rows verified in instance Id 1 = 3
````

All rows in your `ORDERS` table should be verified as valid.

## Acknowledgements

- **Author** - Valentin Leonard Tabacaru
- **Last Updated By/Date** - Valentin Leonard Tabacaru, Principal Product Manager, DB Product Management, Aug 2020

See an issue? Please open up a request [here](https://github.com/oracle/learning-library/issues). Please include the workshop name and lab in your request.

