# Oracle Cloud Infrastructure (OCI)

## Introduction

This is the first of several labs that are part of the Oracle Public Cloud Database Cloud Service workshop. These labs will give you a basic understanding of the Oracle Database Cloud Service and many of the capabilities around administration and database development.

This lab will walk you through creating a new Database Cloud Service instance. You will also connect into a Database image using the SSH private key and familiarize yourself with the image layout. 

## Step 1: Login to OCI Console

Oracle cloud console URL: [https://console.eu-frankfurt-1.oraclecloud.com](https://console.eu-frankfurt-1.oraclecloud.com)

- Tenant: oci-tenant
- Username: oci-username
- Password: oci-password

## Step 2: Create Network

Click on hamburger menu ≡, then Networking > **Virtual Cloud Networks**. Select your Region and Compartment assigned by the instructor. Click **Start VCN Wizard**.

Select **VCN with Internet Connectivity**. Start VCN Wizard.

- VCN Name: [Your Initials]-VCN (e.g. VLT-VCN)
- Compartment: [Your Compartment]

Next. Create.

## Step 3: Create Database

Click on hamburger menu ≡, then **Bare Metal, VM, and Exadata** under Databases. **Create DB System**.

- Select a compartment: [Your Compartment]
- Name your DB system: [Your Initials]-DB (e.g. VLT-DB)
- Select a shape type: Virtual Machine
- Select a shape: VM.Standard2.1
- Oracle Database software edition: Enterprise Edition Extreme Performance
- Choose Storage Management Software: Logical Volume Manager
- Add public SSH keys: Upload SSH key files > id_rsa.pub
- Choose a license type: Bring Your Own License (BYOL)

Specify the network information.

- Virtual cloud network: [Your Initials]-VCN
- Client Subnet: Public Subnet
- Hostname prefix: [Your Initials]-host (small case, e.g. vlt-host)

Next.

- Database name: [Your Initials]DB (e.g. VLTDB)
- Database version: 19c
- PDB name: PDB011
- Password: DBlearnPTS#20_
- Select workload type: Transaction Processing
- Configure database backups: Enable automatic backups

**Create DB System**.

## Step 4: Create Compute Instance

Click on hamburger menu ≡, then Compute > **Instances**. Click **Create Instance**.

- Name: [Your Initials]-VM (e.g. VLT-VM)
- Image or operating system: Change Image > Oracle Images > Oracle Cloud Developer Image
- Virtual cloud network: [Your Initials]-VCN
- Subnet: Public Subnet
- Assign a public IP address
- Add SSH keys: Choose SSH key files > id_rsa.pub

**Create**.

## Step 5: Compute SSH Connection

Wait for both Compute Instance and DB System to finish provisioning, and have status Available.

### Compute Instance Details

Click on hamburger menu ≡, then Compute > **Instances**. Click **[Your Initials]-VM** Compute instance. On the Instance Details page, copy Public IP Address in your notes. 

### DB System Details

Click on hamburger menu ≡, then **Bare Metal, VM, and Exadata**. Click **[Your Initials]-DB** DB System. On the DB System Details page, copy Host Domain Name in your notes. In the table below, copy Database Unique Name in your notes. Click **Nodes** on the left menu, and copy Private IP Address in your notes.

Connect to the Compute node using SSH.

````
ssh -C -i id_rsa opc@<Compute Public IP Address>
````

Try to connect to your DB System database using SQL*Plus.

````
sqlplus sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/<Database Unique Name>.<Host Domain Name> as sysdba
````

It is not possible to connect because port 1521 is closed. It requires a new Security Rule to open this port.

````
ORA-12170: TNS:Connect timeout occurred
````

Press Ctrl-C and Enter to abort SQL*Plus.


## Step 6: Create Security Rule

When complete, under Networking > **Virtual Cloud Networks**. Click **[Your Initials]-VCN** for details. 

Click **Public Subnet-[Your Initials]-VCN**. Click **Default Security List for [Your Initials]-VCN**. Click **Add Ingress Rules**.

- CIDR Block: 10.0.0.0/24
- Destination Port Range: 1521
- Description: Database connection

**Save Changes**.

## Step 7: Verify Connectivity

Use the same Compute node SSH connection. Try to connect to your DB System database using SQL*Plus.

````
sqlplus sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/<Database Unique Name>.<Host Domain Name> as sysdba
````

List pluggable databases.

````
show pdbs
````

Exit SQL*Plus.

````
exit
````

Connect to the pluggable database.

````
sqlplus sys/DBlearnPTS#20_@<DB Node Private IP Address>:1521/pdb011.<Host Domain Name> as sysdba
````

Display the current container name.

````
show con_name
````

List all users in PDB011.

````
select username from all_users order by 1;
````

This pluggable database doesn't have Oracle Sample Schemas. Exit SQL*Plus.

````
exit
````

## Acknowledgements

- **Author** - Valentin Leonard Tabacaru
- **Last Updated By/Date** - Valentin Leonard Tabacaru, Principal Product Manager, DB Product Management, Aug 2020

See an issue? Please open up a request [here](https://github.com/oracle/learning-library/issues). Please include the workshop name and lab in your request.

