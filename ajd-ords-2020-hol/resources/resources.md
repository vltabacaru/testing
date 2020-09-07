# Development Project Resources

## Introduction

Wikidata is a central storage repository that can be accessed by others, such as the wikis maintained by the Wikimedia Foundation. The Wikidata repository consists mainly of items, each one having a label, a description and any number of aliases. Items are uniquely identified by a *Q* followed by a number, such as Douglas Adams (Q42). Statements describe detailed characteristics of an Item and consist of a property and a value. Properties in Wikidata have a *P* followed by a number, such as with educated at (P69).

The Wikidata Query Service provides a SPARQL endpoint. SPARQL is an RDF query language, that is, a semantic query language for databases. With SPARQL you can extract any kind of data, with a query composed of logical combinations of triples. RDF is a directed, labeled graph data format for representing information in the Web.

## Step 1: Login to OCI Console

Oracle cloud console URL: [https://console.eu-frankfurt-1.oraclecloud.com](https://console.eu-frankfurt-1.oraclecloud.com)

- Tenant: oci-tenant
- Username: oci-username
- Password: oci-password

## Step 2: Test Wikidata API

Wikidata is part of the non-profit, multilingual, free-content Wikimedia family. The data in Wikidata is published under the Creative Commons Public Domain Dedication 1.0, allowing the reuse of the data in many different scenarios. You can copy, modify, distribute and perform the data, even for commercial purposes, without asking for permission.

Let’s go through a simple example demonstrating how to get a list of all known cats in the world, information available on [Wikidata:WikiProject Cat breeds](https://www.wikidata.org/wiki/Wikidata:WikiProject_Cat_breeds). Use the Cloud Shell on OCI to run the following **curl** command. 

![](./images/cloudShell.jpg "")

````
curl --header "Accept: application/sparql-results+json"  -G 'https://query.wikidata.org/sparql' --data-urlencode query='
SELECT ?item ?itemLabel 
WHERE 
{
  ?item wdt:P31 wd:Q146.
  SERVICE wikibase:label { bd:serviceParam wikibase:language "[AUTO_LANGUAGE],en". }
}'
````

You will receive a long JSON document from Wikidata with all the cats in the world.

````
{
  "head" : {
    "vars" : [ "item", "itemLabel" ]
  },
  "results" : {
    "bindings" : [ {
      "item" : {
        "type" : "uri",
        "value" : "http://www.wikidata.org/entity/Q28792126"
      },
      "itemLabel" : {
        "xml:lang" : "en",
        "type" : "literal",
        "value" : "Gli"
      }
    }, {
      "item" : {
        "type" : "uri",
        "value" : "http://www.wikidata.org/entity/Q30600575"
      },
      "itemLabel" : {
        "xml:lang" : "en",
        "type" : "literal",
        "value" : "Orlando"
      }
    }, {
      "item" : {
        "type" : "uri",
...
````

>**Note** : If you want to see the entire JSON document, add '` | more `' at the end of the command.

In order to run the same SPARQL from our Autonomous JSON Database, we will convert it to URL format:

````
SELECT%20?item%20?itemLabel%20%0AWHERE%20%0A%7B%0A%20%20?item%20wdt:P31%20wd:Q14
6.%0A%20%20SERVICE%20wikibase:label%20%7B%20bd:serviceParam%20wikibase:language%
20%22[AUTO_LANGUAGE],en%22.%20%7D%0A%7D
````

## Step 3: Provision Autonomous JSON Database

Click on hamburger menu ≡, then **Autonomous JSON Database** under Oracle Database. **Create Autonomous Database**.

- Select a compartment: [Your Compartment]
- Display name: [Your Initials]-AJD (e.g. VLT-AJD)
- Database name: [Your Initials]AJD (e.g. VLTAJD)
- Choose a workload type: JSON
- Choose a deployment type: Shared Infrastructure
- Choose database version: 19c
- OCPU count: 1
- Storage (TB): 1
- Auto scaling: enabled

Under Create administrator credentials:

- Password: DBlearnPTS#20_

Under Choose network access:

- Access Type: Allow secure access from everywhere

**Create Autonomous Database**.

Wait for Lifecycle State to become Available.

## Step 4: Create Apex Workspace DEMO

On Tools tab, under Oracle Application Express, click **Open APEX**.

On Administration Services login page, use password for ADMIN.

- Password: DBlearnPTS#20_

**Sing In to Administration**.

Click **Create Workspace**.

- Database User: DEMO
- Password: DBlearnPTS#20_
- Workspace Name: DEMO

Click **Create Workspace**.

Click AD on upper right corner, **Sign out**. Click **Return to Sign In Page**.

- Workspace: demo
- Username: demo
- Pasword: DBlearnPTS#20_

**Sign In**.

## Step 5: Enable SQL Developer Web for DEMO user

On Oracle Cloud Infrastrucuture Console, on Tools tab, under SQL Developer Web, click **Open SQL Developer Web**.

Copy link from browser in your notes:

https://kndl0dsxmmt29t1-vltajd.adb.eu-frankfurt-1.oraclecloudapps.com/ords/admin/_sdw/?nav=worksheet

Use ADMIN user credentials.

- Username: admin
- Password: DBlearnPTS#20_

On SQL Dev Web Worksheet as ADMIN user, run the following code:

````
BEGIN 
   ords_admin.enable_schema (
      p_enabled => TRUE,
      p_schema => 'DEMO',
      p_url_mapping_type => 'BASE_PATH',
      p_url_mapping_pattern => 'demo',
      p_auto_rest_auth => NULL
   ) ;
  commit ;
END ; 
/
````

For all code you run in SQL Developer Web, make sure you receive a success message:

````
PL/SQL procedure successfully completed.
````

## Step 6:  Create ACL for DEMO to WikiData

The `DBMS_NETWORK_ACL_ADMIN` package provides the interface to administer the network Access Control List (ACL). Grant the **http**, **connect** and **resolve** privileges for host **query.wikidata.org** to DEMO user.

````
BEGIN
  DBMS_NETWORK_ACL_ADMIN.append_host_ace (
    host       => 'query.wikidata.org',
    ace        => xs$ace_type(privilege_list => xs$name_list('http','connect','resolve'),
                              principal_name => 'DEMO',
                              principal_type => xs_acl.ptype_db));
END;
/
````

Grant **SODA_APP** to DEMO user. This role provides privileges to use the SODA APIs, in particular, to create, drop, and list document collections.

````
GRANT SODA_APP TO demo;
````

Click **ADMIN** upper right corner, and **Sign Out**.

## Acknowledgements

- **Author** - Valentin Leonard Tabacaru
- **Last Updated By/Date** - Valentin Leonard Tabacaru, Principal Product Manager, DB Product Management, Sep 2020

See an issue? Please open up a request [here](https://github.com/oracle/learning-library/issues). Please include the workshop name and lab in your request.

