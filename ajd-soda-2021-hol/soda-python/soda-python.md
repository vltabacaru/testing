# JSON SODA in Oracle Autonomous JSON Database (AJD) using Python

## Introduction

Oracle Autonomous JSON Database (AJD) is a new service in Autonomous Database family for JSON-centric development. Autonomous JSON Database is built for developers who are looking for an easy to use, cost-effective JSON database with native Document API support. Autonomous JSON Database provides all the core capabilities of a document store along with high performance, simple elasticity, full ACID support and complete SQL functionality. Autonomous JSON Database is available only on Shared Infrastructure. Also, customers can one-click upgrade from Autonomous JSON Database (AJD) to full featured Autonomous Transaction Processing Database (ATP) any time.

Estimated Lab Time: 60 minutes

### Objectives
In this lab, you will:
* Create the blockchain table
* Insert and delete rows
* Drop the blockchain table
* View blockchain tables hidden columns
* Check the validity of rows in the blockchain table

### Prerequisites
* OCI resources (Introduction)
* MongoDB Cloud Atlas (Introduction)

## **STEP 1:** Create Python application for MongoDB document store

1. Connect to **[Your Initials]-ClientVM** Compute instance using SSH, if not connected, as user **opc**.

2. Use the substitute user command to start a session as **oracle** user. 

    ````
    sudo su - oracle
    ````

    >**Note** : This step is not required if you prefer to use the Remote Desktop connection for this lab.

3. Create a new folder under `/home/oracle` as the location of our new Python application.

    ````
    mkdir python-simple-project
    cd python-simple-project
    ````

touch simple-app.py
touch requirements.pip

pip3 install --user virtualenv
# Create virtualenv in repo
virtualenv .
# Activate Virtualenv
source ./bin/activate   (deactivate)

requirements.pip:
Flask
pymongo
dnspython

simple-app.py:
import os
import json
import pymongo
from flask import Flask
from flask import request
app = Flask(__name__)

usr = os.environ['MONGO_DB_USER']
pwd = os.environ['MONGO_DB_PASS']
dbn = os.environ['MONGO_DB_NAME']

client = pymongo.MongoClient("mongodb+srv://" + usr + ":" + pwd + "@cluster0.dsbwl.mongodb.net/" + dbn + "?retryWrites=true&w=majority")
db = client[dbn]
collection = db['SimpleCollection']

@app.route("/", methods=['POST'])
def insert_mongo_doc():
    req_data = request.get_json()
    collection.insert_one(req_data).inserted_id
    return ('', 204)

@app.route('/')
def get_mongo_doc():
    documents = collection.find()
    response = []
    for document in documents:
        document['_id'] = str(document['_id'])
        response.append(document)
    return json.dumps(response)

if __name__ == '__main__':
    app.run(host= '0.0.0.0')


export MONGO_DB_USER=mongoUser
export MONGO_DB_PASS=DBlearnPTS#21_
export MONGO_DB_NAME=SimpleDatabase

pip3 install -r requirements.pip

python3.6 simple-app.py

http://130.61.234.226:5000/

curl --request POST \
  --url http://localhost:5000/ \
  --header 'content-type: application/json' \
  --data '{
 "company":"Capital One",
 "location":"McLean",
 "state":"VA",
 "country":"US"
}'

curl --request POST \
  --url http://130.61.234.226:5000/ \
  --header 'content-type: application/json' \
  --data '{
 "company":"Capital Two",
 "location":"Braila",
 "state":"BR",
 "country":"RO"
}'

curl --request POST \
  --url http://localhost:5000/ \
  --header 'content-type: application/json' \
  --data '{
 "company":"Capital Three",
 "location":"Botosani",
 "state":"BT",
 "country":"MD"
}'

## **STEP 2:** Improve Python application with AJD document store

Download and unzip wallet

export TNS_ADMIN=/home/oracle/Wallet_AJD0

requirements.pip:
cx_Oracle

pip3 install -r requirements.pip

sqlplus demo/DBlearnPTS#20_@ajd0_tp

export ORCL_DB_USER=demo
export ORCL_DB_PASS=DBlearnPTS#20_
export ORCL_DB_SERV=ajd0_tp

https://vltabacaru.github.io/pts-ws-cicd/workshops/cicd-complete/?lab=lab-4-oracle-database-for-python
https://blogs.oracle.com/opal/python-cx_oracle-7-introduces-soda-document-storage

simple-app.py:
import os
import json
import pymongo
import cx_Oracle
from flask import Flask
from flask import request
app = Flask(__name__)

m_usr = os.environ['MONGO_DB_USER']
m_pwd = os.environ['MONGO_DB_PASS']
m_dbn = os.environ['MONGO_DB_NAME']

o_usr = os.environ['ORCL_DB_USER']
o_pwd = os.environ['ORCL_DB_PASS']
o_srv = os.environ['ORCL_DB_SERV']

client = pymongo.MongoClient("mongodb+srv://" + m_usr + ":" + m_pwd + "@cluster0.dsbwl.mongodb.net/" + m_dbn + "?retryWrites=true&w=majority")
mdb = client[m_dbn]
mcollection = mdb['SimpleCollection']

conn_string = o_usr + '/' + o_pwd + '@' + o_srv
connection = cx_Oracle.connect(conn_string)
connection.autocommit = True
soda = connection.getSodaDatabase()
ocollection = soda.createCollection("SimpleCollection")

@app.route("/m/", methods=['POST'])
def insert_mongo_doc():
    req_data = request.get_json()
    mcollection.insert_one(req_data).inserted_id
    return ('', 204)

@app.route('/m/')
def get_mongo_doc():
    documents = mcollection.find()
    response = []
    for document in documents:
        document['_id'] = str(document['_id'])
        response.append(document)
    return json.dumps(response)

@app.route("/o/", methods=['POST'])
def insert_orcl_doc():
    req_data = request.get_json()
    ocollection.insertOne(req_data)
    return ('', 204)

@app.route('/o/')
def get_orcl_doc():
    documents = ocollection.find().getDocuments()
    response = []
    for document in documents:
        content = document.getContent()
        content['key'] = str(document.key)
        response.append(content)
    return json.dumps(response)

if __name__ == '__main__':
    app.run(host= '0.0.0.0')

  ########### %%%%%%%%%%%%%%% ############ >> Env variables

variables.sh:
#!/bin/bash
export MONGO_DB_PASS=DBlearnPTS#21_
export MONGO_DB_USER=mongoUser
export MONGO_DB_NAME=SimpleDatabase
export TNS_ADMIN=/home/oracle/Wallet_AJD0
export ORCL_DB_USER=demo
export ORCL_DB_PASS=DBlearnPTS#20_
export ORCL_DB_SERV=ajd0_tp

chmod a+x variables.sh

. variables.sh

  ########### %%%%%%%%%%%%%%% ############ >> 

curl --request POST \
  --url http://130.61.234.226:5000/m/ \
  --header 'content-type: application/json' \
  --data '{
 "company":"Capital Four",
 "location":"Vaslui",
 "state":"VS",
 "country":"UK"
}'


curl --request POST \
  --url http://130.61.234.226:5000/o/ \
  --header 'content-type: application/json' \
  --data '{
 "company":"Capital Five",
 "location":"Malaga",
 "state":"MA",
 "country":"ES"
}'

curl --request POST \
  --url http://130.61.234.226:5000/o/ \
  --header 'content-type: application/json' \
  --data '{
 "company":"Capital Six",
 "location":"Barcelona",
 "state":"CA",
 "country":"ES"
}'


  ########### %%%%%%%%%%%%%%% ############ >> Pagination

Mongo:
documents = mcollection.find().limit(2).skip(2)

Oracle:
documents = ocollection.find().limit(2).skip(2).getDocuments()

  ########### %%%%%%%%%%%%%%% ############ >> Migration

## **STEP 3:** Migrate documents between document stores

simple-migrate.py 
import os
import json
import pymongo
import cx_Oracle

m_usr = os.environ['MONGO_DB_USER']
m_pwd = os.environ['MONGO_DB_PASS']
m_dbn = os.environ['MONGO_DB_NAME']

o_usr = os.environ['ORCL_DB_USER']
o_pwd = os.environ['ORCL_DB_PASS']
o_srv = os.environ['ORCL_DB_SERV']

client = pymongo.MongoClient("mongodb+srv://" + m_usr + ":" + m_pwd + "@cluster0.dsbwl.mongodb.net/" + m_dbn + "?retryWrites=true&w=majority")
mdb = client[m_dbn]
mcollection = mdb['SimpleCollection']

conn_string = o_usr + '/' + o_pwd + '@' + o_srv
connection = cx_Oracle.connect(conn_string)
connection.autocommit = True
soda = connection.getSodaDatabase()
ocollection = soda.createCollection("SimpleCollection")

if __name__ == '__main__':
    documents = mcollection.find()
    for document in documents:
        document['_id'] = str(document['_id'])
        doc = ocollection.insertOneAndGet(document)
        key = doc.key
        print('Migrated SODA document key: ', key)
        doc = ocollection.find().key(key).getOne() 
        content = doc.getContent()
        print('Migrated SODA document: ')
        print(content)









1. Connect to PDB1 pluggable database as SYSTEM user.

    ````
    $  sqlplus system/WElcome123##@Hostname-Prefix:1521/pdb1.Host-Domain-Name
    ````

    **Hostname Prefix** and **Host Domain Name** values can be found on Database System details page, under Network.

2. Create the OOE user, owner of the blockchain table.

    Users require no special privileges to create and work with blockchain tables or JSON data type. A simple user will do the job just fine. As a best practice, you can create a specific tablespace for storing JSON documents, but for this simple case we will use the existing one.

	````
	create user ooe identified by "WElcome123##" default tablespace USERS quota unlimited on USERS;
	````

	````
	GRANT connect, resource TO ooe;
	````

3. Connect to PDB1 pluggable database as OOE user.

    The rest of the scenario is executed by OOE user we just created.

	````
	conn ooe/WElcome123##@Hostname-Prefix:1521/pdb1.Host-Domain-Name
	````

4. Some formatting for SQL*Plus will help me understand the output better. Hit Enter one more time after pasting these lines.

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


## Acknowledgements
* **Author** - Valentin Leonard Tabacaru, PTS
* **Contributors** -  Kay Malcolm, Database Product Management
* **Last Updated By/Date** -  Valentin Leonard Tabacaru, December 2020

## Need Help?
Please submit feedback or ask for help using our [LiveLabs Support Forum](https://community.oracle.com/tech/developers/categories/livelabsdiscussions). Please click the **Log In** button and login using your Oracle Account. Click the **Ask A Question** button to the left to start a *New Discussion* or *Ask a Question*.  Please include your workshop name and lab name.  You can also include screenshots and attach files.  Engage directly with the author of the workshop.

If you do not have an Oracle Account, click [here](https://profile.oracle.com/myprofile/account/create-account.jspx) to create one.
