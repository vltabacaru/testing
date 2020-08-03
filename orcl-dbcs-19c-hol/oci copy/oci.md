# Oracle Cloud Infrastructure (OCI)

## Introduction

Development .

## Step 1: Genera

On a 

````
ssh-keygen
````

No passphrase

>**Note** : If you 

Some 

## Step 2: Login 

Oracle cloud console URL: [https://console.eu-frankfurt-1.oraclecloud.com](https://console.eu-frankfurt-1.oraclecloud.com)

- Tenant: oci-tenant
- Username: oci-username
- Password: oci-password

## Step 3: Create Network

Click on hamburger menu ≡, then Networking > **Virtual Cloud Networks**. Click **Start VCN Wizard**.

Select **VCN with Internet Connectivity**. Start VCN Wizard.

- VCN Name: [Your Initials]-VCN (e.g. VLT-VCN)
- Compartment: [Your Compartment]

Next. Create.

### Step 3b: Create Subnet

When complete, under Networking > **Virtual Cloud Networks**. Click **[Your Initials]-VCN** for details. Click **Create Subnet**.

- Name: Public LB Subnet-[Your Initials]-VCN (e.g. Public LB Subnet-VLT-VCN)
- Subnet Type:  Regional
- CIDR Block: 10.0.2.0/24
- Route Table: Default Route Table for [Your Initials]-VCN
- Subnet Access: Public Subnet
- DHCP Options: Default DHCP Options for [Your Initials]-VCN
- Security List: Default Security List for [Your Initials]-VCN

**Create Subnet**.

## Step 4: Create Database

Click on hamburger menu ≡, then **Bare Metal, VM, and Exadata** under Databases. **Create DB System**.

- Select a compartment: [Your Compartment]
- Name your DB system: [Your Initials]-DB (e.g. VLT-DB)
- Select a shape type: Virtual Machine
- Select a shape: VM.Standard2.1
- Add public SSH keys: Upload SSH key files > id_rsa.pub
- Choose a license type: Bring Your Own License (BYOL)

Specify the network information.

- Virtual cloud network: [Your Initials]-VCN
- Client Subnet: Public Subnet
- Hostname prefix: [Your Initials]-host (small case, e.g. vlt-host)

Next.

- Database name: [Your Initials]DB (e.g. VLTDB)
- Database version: 19c
- PDB name: PDB01
- Password: WelCom3#2020_
- Select workload type: Transaction Processing

**Create DB System**.

## Step 5: Create Compute Instance

Click on hamburger menu ≡, then Compute > **Instances**. Click **Create Instance**.

- Name: [Your Initials]-VM (e.g. VLT-VM)
- Image or operating system: Change Image > Oracle Images > Oracle Cloud Developer Image
- Virtual cloud network: [Your Initials]-VCN
- Subnet: Public Subnet
- Assign a public IP address
- Add SSH keys: Choose SSH key files > id_rsa.pub

Create.

## Acknowledgements

- **Author** - Valentin Leonard Tabacaru
- **Last Updated By/Date** - Valentin Leonard Tabacaru, Principal Product Manager, DB Product Management, Aug 2020

See an issue? Please open up a request [here](https://github.com/oracle/learning-library/issues). Please include the workshop name and lab in your request.

