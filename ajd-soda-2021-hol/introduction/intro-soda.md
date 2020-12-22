# SODA for Python Workshop Introduction

## Introduction

SODA for Python is a Python API that implements Simple Oracle Document Access (SODA). It is part of the Oracle Python driver, cx_Oracle — no additional installation is needed.

You can use SODA for Python to perform create, read (retrieve), update, and delete (CRUD) operations on documents of any kind, and you can use it to query JSON documents.

SODA is a set of NoSQL-style APIs that let you create and store collections of documents (in particular JSON) in Oracle Database, retrieve them, and query them, without needing to know Structured Query Language (SQL) or how the documents are stored in the database.

Estimated Lab Time: 60 minutes

### Objectives
In this lab, you will:
* Create network and compute resources on Oracle Cloud
* Prepare cloud infrastructure for development
* Provision Oracle Autonomous JSON Database
* Create MongoDB document store on MongoDB Cloud

### Prerequisites
* An Oracle Free Tier, Always Free, Paid or LiveLabs Cloud Account
* SSH Keys for Putty or OpenSSH (id_rsa.ppk or id_rsa, and id_rsa.pub)


## **STEP 1:** Create Virtual Cloud Network (VCN)

1. Login to Oracle cloud console using the URL: [https://console.eu-frankfurt-1.oraclecloud.com](https://console.eu-frankfurt-1.oraclecloud.com)

- Tenant: oci-tenant
- Username: oci-username
- Password: oci-password

2. Click on main menu ≡, then Networking > **Virtual Cloud Networks**. Select your Region and Compartment assigned by the instructor. 

3. Click **Start VCN Wizard**.

4. Select **VCN with Internet Connectivity**. Start VCN Wizard.

- VCN Name: [Your Initials]-VCN (e.g. VLT-VCN)
- Compartment: [Your Compartment]

5. Click Next and Create.

6. Click **[Your Initials]-VCN** for details. 

7. Click **Public Subnet-[Your Initials]-VCN**. Click **Default Security List for [Your Initials]-VCN**. Click **Add Ingress Rules**.

- CIDR Block: 0.0.0.0/0
- Destination Port Range: 5000
- Description: Python Flask

8. Click **Save Changes**.


## **STEP 2:** Provision Compute Node for development

1. Click on main menu ≡, then Compute > **Instances**. Click **Create Instance**.

- Name: [Your Initials]-ClientVM (e.g. VLT-ClientVM)
- Image or operating system: Change Image > Oracle Images > Oracle Cloud Developer Image
- Virtual cloud network: [Your Initials]-VCN
- Subnet: Public Subnet
- Assign a public IP address
- Add SSH keys: Choose SSH key files > id_rsa.pub

2. Click **Create**. Wait for Compute Instance to finish provisioning, and have status Available.

3. Click **[Your Initials]-ClientVM** Compute instance. On the Instance Details page, copy Public IP Address in your notes.

4. Connect to the Compute node using SSH. In OpenSSH, local port forwarding is configured using the -L option. Use this option to forward any connection to port 3389 on the local machine to port 3389 on your Compute node. (Mac/Linux only)

    ````
    ssh -C -i id_rsa -L 3389:localhost:3389 opc@[ClientVM Public IP Address]
    ````

5. Connect to the Compute node using SSH Connection From a Windows Client. Connect to Compute Public IP Address port 22. (Windows only)

    ![](./images/putty1.png "")

6. Use the id_rsa.ppk private key. (Windows only)

    ![](./images/putty2.png "")

    ![](./images/putty3.png "")

7. Create a SSH tunnel from source port 5001 to localhost:3389. (Windows only)

    ![](./images/putty4.png "")


## **STEP 3:** Configure Compute Node for development

For some of the labs we need graphical user interface, and this can be achieved using a Remote Desktop connection.

1. Use the substitute user command to start a session as **root** user.

    ````
    sudo su -
    ````

2. Create a new script that will install and configure all the components required for the Remote Desktop connection.

    ````
    vi xRDP_config.sh
    ````

3. Press **i** to insert text, and paste the following lines:

    ````
    #!/bin/bash

    yum -y groupinstall "Server with GUI"

    yum -y install xrdp tigervnc-server terminus-fonts terminus-fonts-console cabextract

    yum -y update sqldeveloper.noarch

    yum -y localinstall https://downloads.sourceforge.net/project/mscorefonts2/rpms/msttcore-fonts-installer-2.6-1.noarch.rpm

    yum -y update sqldeveloper.noarch

    sed -i 's/max_bpp=24/max_bpp=128\nuse_compression=yes/g' /etc/xrdp/xrdp.ini

    systemctl enable xrdp

    firewall-cmd --permanent --add-port=3389/tcp
    firewall-cmd --permanent --add-port=5000/tcp
    firewall-cmd --reload

    chcon --type=bin_t /usr/sbin/xrdp
    chcon --type=bin_t /usr/sbin/xrdp-sesman

    systemctl start xrdp

    echo -e "DBlearnPTS#21_\nDBlearnPTS#21_" | passwd oracle

    sed -i -e 's/^/#/' /etc/profile.d/oracle-instantclient18.5.sh

    printf "\nORACLE_HOME=/opt/oracle/product/19c/dbhome_1\nLD_LIBRARY_PATH=\$ORACLE_HOME/lib\nPATH=\$PATH:\$ORACLE_HOME/bin\nexport ORACLE_HOME LD_LIBRARY_PATH PATH\n" >> /etc/profile
    ````

4. Press **Esc**, type **:wq** and hit **Enter** to save the file and close. Make this script executable.

    ````
    chmod u+x xRDP_config.sh 
    ````

5. Run the script and check that all goes well.

    ````
    ./xRDP_config.sh
    ````

6. Close session as **root** user.

    ````
    exit
    ````

7. Use Microsoft Remote Desktop to open a connection to **localhost**. (Mac/Linux only)

8. If you are using Putty on Windows, connect to **localhost:5001**. (Windows only)

    ![](./images/putty5.png "")

9. When asked about username and password, use **oracle** and **DBlearnPTS#21_**.

10. After setting your language and keyboard layout, open a Terminal window using **Right-Click** and **Open Terminal**. Check if your keyboard works. If you need to select another keyboard layout, click the **On-Off** button in the upper right corner, and **Settings** button. You will find the options under Region & Language.

## **STEP 4:** Provision Oracle Autonomous JSON Database (AJD)

1. Click on main menu ≡, then **Autonomous JSON Database** under Oracle Database. **Create Autonomous Database**.

- Select a compartment: [Your Compartment]
- Display name: [Your Initials]-AJD (e.g. VLT-AJD)
- Database name: [Your Initials]AJD (e.g. VLTAJD)
- Choose a workload type: JSON
- Choose a deployment type: Shared Infrastructure
- Choose database version: 19c
- OCPU count: 1
- Storage (TB): 1
- Auto scaling: enabled

2. Under Create administrator credentials:

- Password: DBlearnPTS#21_

3. Under Choose network access:

- Access Type: Allow secure access from everywhere

4. Click **Create Autonomous Database**. Wait for Lifecycle State to become Available.

5. On Tools tab, under Oracle Application Express, click **Open APEX**. On Administration Services login page, use password for ADMIN.

- Password: DBlearnPTS#21_

6. Click **Sing In to Administration**. Click **Create Workspace**.

- Database User: DEMO
- Password: DBlearnPTS#21_
- Workspace Name: DEMO

7. Click **Create Workspace**. Click AD on upper right corner, **Sign out**. Click **Return to Sign In Page**.

- Workspace: demo
- Username: demo
- Pasword: DBlearnPTS#21_

8. Click **Sign In**. Oracle APEX uses low-code development to let you build data-driven apps quickly without having to learn complex web technologies. This also gives you access to Oracle REST Data Services, that allows developers to readily expose and/or consume RESTful Web Services by defining REST end points.

9. On Oracle Cloud Infrastrucuture Console, on Tools tab, under SQL Developer Web, click **Open SQL Developer Web**. Copy the URL from browser in your notes:

    https://kndl0dsxmmt29t1-vltajd.adb.eu-frankfurt-1.oraclecloudapps.com/ords/admin/_sdw/?nav=worksheet

10. Use ADMIN user credentials to login.

- Username: admin
- Password: DBlearnPTS#21_

11. On SQL Dev Web Worksheet as ADMIN user, run the following code:

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

    >**Note** : For all code you run in SQL Developer Web, make sure you receive a success message:

    ````
    PL/SQL procedure successfully completed.
    ````

12. Grant **SODA_APP** to DEMO user. This role provides privileges to use the SODA APIs, in particular, to create, drop, and list document collections.

    ````
    GRANT SODA_APP TO demo;
    ````

13. Click **ADMIN** upper right corner, and **Sign Out**. Login using DEMO user credentials.

- Username: demo
- Password: DBlearnPTS#21_

14. Save in your notes the URL of SQL Developer Web for DEMO user, by changing '**admin**' with '**demo**' in the URL you saved for ADMIN user:

    https://kndl0dsxmmt29t1-vltajd.adb.eu-frankfurt-1.oraclecloudapps.com/ords/demo/_sdw/?nav=worksheet


## **STEP 5:** Deploy Atlas document store on MongoDB Cloud

One of the objectives of this workshop is to show the integration of Oracle Autonomous JSON Database with existing document stores like MongoDB. This is why you need an existing MongoDB database, and if you don't have one, you can provision it quickly on MondoDB Cloud.

1. Access MongoDB Cloud at [https://cloud.mongodb.com](https://cloud.mongodb.com), and create an account. You can login using your Google account.

2. Create a new Cluster using the default settings.

3. Once your Cluster is up and running, on the overview page, click Connect.

4. Click Add a Different IP Address, and use the Public IP address of your ClientVM Compute Node.

5. Create a Database User: mongoUser/DBlearnPTS#21_

    Save the username and the password in your notes.

6. Click Connect Your Application: Python 3.6 or later. You will receive a connection string like this:

    ````
    mongodb+srv://mongoUser:[password]@cluster_name.dsbwl.mongodb.net/[dbname]?retryWrites=true&w=majority
    ````

    Save this string in your notes.

7. Under Collections, use Load a Sample Dataset wizard to generate some JSON documents for different use cases in your MongoDB database. Navigate these sample datasets and familiarize yourself with JSON documents, if this is your first experience.

8. Click Create Database, and name it SimpleDatabase, and the collection SimpleCollection. This will be used for our Python application development in the next lab.


## Acknowledgements
* **Author** - Valentin Leonard Tabacaru, PTS
* **Contributors** -  Kay Malcolm, Database Product Management
* **Last Updated By/Date** -  Valentin Leonard Tabacaru, December 2020

## Need Help?
Please submit feedback or ask for help using our [LiveLabs Support Forum](https://community.oracle.com/tech/developers/categories/livelabsdiscussions). Please click the **Log In** button and login using your Oracle Account. Click the **Ask A Question** button to the left to start a *New Discussion* or *Ask a Question*.  Please include your workshop name and lab name.  You can also include screenshots and attach files.  Engage directly with the author of the workshop.

If you do not have an Oracle Account, click [here](https://profile.oracle.com/myprofile/account/create-account.jspx) to create one.
