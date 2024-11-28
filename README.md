![MambaETL.png](_markdown%2FMambaETL.png)

# openmrs-module-mamba-etl

## **Background**

MambaETL (or simply known as Mamba) is an OpenMRS (Open Electronic Medical Records System) implementation for data Extraction, Loading and Transforming (ETL).
From highly normalised data such as the Observeration (Obs) data into a more denormalised format for faster data retrieval and analysis.

OpenMRS stores patient observational data in a long format. Essentially, for each encounter type for a given patient, multiple rows are saved into the OpenMRS Obs table. Sometimes as many as 50 or more rows saved for a single encounter in just the Obs table.

This means that the Obs table quickly grows to millions of records in fairly sized facilities making reporting and any analysis on such data incredibly difficult and slow.

## **Purpose of this module**

The `openmrs-module-mamba-etl` module found [here](https://github.com/UCSF-IGHS/openmrs-module-mamba-etl) or simply the MambaETL module is an OpenMRS example module of an ETL implementation. The module can also be used as is or alternatively an implementer can
can use it as a starter module to build on a more robust ETL solution. 

Out of the box, this module is a collection of familiar artefacts/tooling that collectively offer out-of-the box database flattening/transposing and abstraction of repetitive reporting tasks 
so that implementers, analysts, data scientists or teams building reports focus on building without worrying about system performance bottlenecks or bothering too much about how the data is extracted from the primary data source into the reporting destination.

This module specifically has a dependency on the MambaETL core library `openmrs-module-mamba-core` found [here](https://github.com/UCSF-IGHS/openmrs-module-mamba-core/releases).

## **A Quick start-up setup guide**

<span style='color: red'>Pre-Requisites</span>

* A running instance of the OpenMRS server
* The OpenMRS database user used by the running OpenMRs server must have elevated rights to be able to create databases (i.e. the analysis (ETL) db), Stored Procedures and Functions.
* A second database user with full access to the ETL database to drop and create Tables (should be configured in the openmrs-runtime-properties, more on this later)
* MySQL v5.7.8 and above running
* Java functionality (JAVA 7 & above)
* Access to the OpenMRS admin interface to be able to re-configure the ETL task

<span style='color: red'>How to setup and deploy the ETL module</span>

MambaETL table flattening comes bundled with MambaETL out of the box.

However, for this to work a few steps need to be taken:

1. Check the OpenMRS database user privileges and confirm that the user has the privileges to create new databases, stored procedures and Functions in this database. 
   
   **Note**:
   If you created your OpenMRs instance using the default `Simple` option in the wizard, MambaETL will not be able to work since the wizard creates a database user with fewer privileges than is required for MambaETL.
   However, you can solve this by elevating the user rights to be able to create databases, stored procedures and functions. 
   
   To elevate database user privileges, run the commands to grant them.

   
      -- Check to see the privileges user has, assuming user name is 'openmrs_user' 
   
      `SHOW GRANTS FOR 'openmrs_user'@'localhost';`

   
      -- Grant privileges to create databases 
   
      `GRANT CREATE ON *.* TO 'openmrs_user'@'localhost';`

   
      -- Grant privileges to create and delete/alter stored procedures and functions

      `GRANT ALTER ROUTINE ON *.* TO 'openmrs_user'@'localhost';`
   
      `GRANT CREATE ROUTINE ON *.* TO 'openmrs_user'@'localhost';`
   
      `GRANT SUPER ON *.* TO 'openmrs_user'@'localhost';`

   
      -- update the privileges
   
      `FLUSH PRIVILEGES;`
   
   This user is needed with the right privileges because once the MambaETL module has been deployed, at starts up, the ETL module needs to run and do two things:
   -  Create the specified ETL database (`analysis_db` by default)   
   -  Drop and Create a number of MambaETL stored procedures and Functions in your specified `analysis database`
   ![routines.png](_markdown%2Froutines.png)


   **Note**:

   In MySQL 8, when binary logging is enabled, the ability to create and drop stored procedures and functions requires the CREATE ROUTINE and ALTER ROUTINE privileges, respectively. However, these privileges are not sufficient for non-SUPER users to drop stored procedures and functions due to security concerns related to binary logging.

   Meaning you will run into this Error when the ETL module tries to execute:

   `... exception.DatabaseException: You do not have the SUPER privilege and binary logging is enabled (you *might* want to use the less safe log_bin_trust_function_creators variable) [Failed SQL: (1419) CREATE FUNCTION..`

   If you want to allow non-SUPER users to drop stored procedures and functions while binary logging is enabled, you have a couple of options:
   
   Grant SUPER Privilege: Granting the SUPER privilege to the user would allow them to create and drop stored procedures and functions even with binary logging enabled. However, this privilege is very powerful and allows the user to perform administrative tasks beyond just creating and dropping routines. Granting SUPER should be done cautiously due to security implications.
   
   `GRANT SUPER ON *.* TO 'openmrs_user'@'localhost';`
   
   sometimes you might need this other permission just incase the error persists: `GRANT SYSTEM_USER ON *.* TO 'openmrs_user'@'localhost';`

   Replace 'openmrs_user'@'localhost' with the appropriate username and host.
   Set log_bin_trust_function_creators: This option is less secure but might be acceptable depending on your environment. It allows non-SUPER users to create and drop routines without requiring the SUPER privilege. However, enabling this option may pose security risks.
   
   `SET GLOBAL log_bin_trust_function_creators = 1;`
   
   Keep in mind that changing global variables like this might require SUPER privileges or appropriate administrative permissions.
   Choose the option that best fits your security requirements and administrative constraints. If you're concerned about granting the SUPER privilege or enabling log_bin_trust_function_creators, you might want to consult with your database administrator or review your security policies.

2. Add a database user configurations to the OpenMRS runtime properties file.

   ![openmrs-runtime-properties.png](_markdown%2Fopenmrs-runtime-properties.png)

   Adding this connection information is not mandatory as the system will default to using the same user connection information as your distribution (if you have provided none). 
   
   Other-wise Create a separate database user with enough privileges to the analysis_db (ETL) database.
   
   The user should be able to create and drop Tables in the analysis_db (ETL) database.

   For-example given the openmrs properties file below:
   
        cat /Users/smallgod/openmrs/mambaetl-ref-app/openmrs-runtime.properties 

   Add the following database user configurations to that file


        mambaetl.analysis.db.driver=com.mysql.cj.jdbc.Driver

        mambaetl.analysis.db.url=jdbc\:mysql\://localhost\:3306/analysis_db?useSSL\=false&autoReconnect\=true

        mambaetl.analysis.db.username=mamba

        mambaetl.analysis.db.password=iopdRmgaphk

        mambaetl.analysis.db.openmrs_database=openmrs

        mambaetl.analysis.db.etl_database=analysis_db
        
        #preferred concepts locale
        mambaetl.analysis.locale=en
        
        #MySQL fails when Columns become too large, we can partition tables nad have other columns in other subsequent tables
        mambaetl.analysis.columns=65
        
        #Whether the ETL should delete and recreate all ETL tables or only add/modify what has changed
        mambaetl.analysis.incremental_mode=1
        
        #MambaETL will automatically transpose all encounters in the OpenMRS database if set to 1
        mambaetl.analysis.automated_flattening=1
        
        #The ETL interval in seconds. ETL will run every number of seconds specified here
        mambaetl.analysis.etl_interval=180


        Incremental Mode (`_mambaetl.analysis.incremental_mode_`):
        
            0: The ETL process will drop and recreate all ETL tables every time it runs. Essentially, a fresh start for each run.
            1: The ETL will run in incremental mode, meaning it will only update (add/modify) data that has changed since the last run, avoiding the need to recreate all tables.
        
        Automated Flattening (`_mambaetl.analysis.automated_flattening_`):
        
            1: This setting will flatten encounter data only for encounters where the corresponding JSON files are provided, whether the files are generated automatically or manually.
            0: No encounters will be flattened.
        
        Scheduler Interval (`_mambaetl.analysis.etl_interval_`):
        
            Specifies the interval, in seconds, at which the ETL should run. For example, setting _mambaetl.analysis.etl_interval=180_ means the ETL will run every 3 minutes.

3. Upload/Deploy the MambaETL module to your OpenMRS instance
   Locate the omod(mamba-etl-1.0.1-SNAPSHOT.omod) file from mambaetl project in the target folder 

   ![location-of-omod.png](_markdown%2Flocation-of-omod.png)
    
   Then copy the file and add/paste it to the modules subfolder in openmrs server folder 

   ![add mamba omod.png](_markdown%2Fadd%20mamba%20omod.png)
    
   You can also use the command prompt

   <span style='color: red'>step 1: Navigate to the Project Directory:</span> 

   Open a terminal and navigate to the root of your project, for example: -

            `cd ~/openmrs-module-mamba-etl`

   <span style='color: red'>step 2: Build the project using Maven:</span>
   
   Ensure that you have Maven installed on your system. To generate the .omod file, run the following Maven command:

            `mvn clean install`
   
   This will:

   Clean the previous build outputs.
   Compile the code.
   Package the module into a .omod file.

   <span style='color: red'>step 3: Navigate to the OMOD directory</span>
   
   After the build is complete, navigate to the omod module’s target directory, where the .omod file is located

            `cd omod/target`

   <span style='color: red'>step 4: Locate the .omod file</span>
   
   In the target directory, you will find the mamba-etl-1.0.1-SNAPSHOT.omod file. You can list the files in this directory using

            `ls`
   
   You should see the .omod file (mamba-etl-1.0.1-SNAPSHOT.omod) listed, along with other files such as .jar files and test outputs.

   <span style='color: red'>step 5: Deploy the .omod file</span>
   
   Now that you've located the .omod file, it’s ready for deployment into the OpenMRS system by uploading it through the OpenMRS admin interface or placing it in the OpenMRS modules folder.
   Then restart your openmrs server.


4. After the module has been deployed successfully, If any MambaETL-related tables and flat tables already exist in the database, they will be automatically dropped as part of the deployment process.
   After dropping the existing tables, the ETL-related tables will be re-created, ensuring the system starts with fresh, predefined table structures.

 ![analysis tables.png](_markdown%2Fanalysis%20tables.png)


## **Frequently Asked Questions (FAQs)**
.....

## **MambaETL: A technical deep dive**


<span style='color: red'>Step 1:</span>

Before proceeding, please refer to the **A Quick start-up setup guide** section and make sure your MambaETL module has been correctly setup and is running. 


<span style='color: red'>Step 2:</span>

Checkout the project from [git](https://github.com/UCSF-IGHS/openmrs-module-mamba-etl)

Below is an example folder structure of the MambaETL project when you have added all the relevant folders and files required to support MambaETL in your project.  
![folder-structure.png](_markdown%2Ffolder-structure.png)


Run `mvn clean install` to build the project.

Every time you build this project 2 files are auto-generated and placed under the `mamba` folder under the `resources` folder.

The 2 files automatically created under the `mamba` folder are:  

- `create_stored_procedures.sql`

- `jdbc_create_stored_procedures.sql`

The `create_stored_procedures.sql` is an SQL compliant file. It contains all the ETL scripts that have been compiled into one 'big' script ready for deployment.  
This file can be run against your ETL target database as-is, mostly for development and test purposes when you need to quickly and manually run your ETL scripts and test them out.

The `jdbc_create_stored_procedures.sql` is a MySQL compliant file. It contains all the ETL scripts that have been compiled into one 'big' script ready for deployment.

<span style='color: red'>Step 3:</span>  

Ensure your `api` submodule has the structure as shown in the image below. We will go through the relevant files/folders one by one.  
![api-submodule1.png](_markdown%2Fapi-submodule1.png)


<span style='color: red'>Step 4:</span>

`../api/pom.xml`  

Under your MambaETL project created/cloned in `Step 1` above, go to the `api` submodule and in the pom xml add the dependency entry for the [MambaETL core module](https://github.com/UCSF-IGHS/openmrs-module-mamba-core) api dependency.

    <dependency>
        <groupId>org.openmrs.module</groupId>
        <artifactId>mamba-core-api</artifactId>
    </dependency>

<span style='color: red'>Step 5:</span>

Locate the omod file from MambaETL project in the target folder

   ![location-of-omod.png](_markdown%2Flocation-of-omod.png)
    
Then copy the file and add/paste it to the modules subfolder in openmrs server folder 

   ![add mamba omod.png](_markdown%2Fadd%20mamba%20omod.png)

This Module ensures the MambaETL `Stored Procedures` and `Functions` are deployed on your target ETL database.  
The changeset deletes and re-creates all given Stored procedures and functions everytime it is run ensuring any new changes/modifications to the ETL are deployed.


<span style='color: red'>Step 6:</span>  

Copy the example `_etl` folder from the [MambaETL reference/template](https://github.com/UCSF-IGHS/openmrs-module-mamba-etl) module under `omod/src/main/resources` to your ETL project `resources` folder under the `omod` submodule of your project.  
![etl-folder.png](_markdown%2Fetl-folder.png)

Under the `_etl` folder are other sub-folders and files, you may need to edit them according to your needs. 
![json-config.png](_markdown%2Fjson-config.png)

Under the config folder, place your json configuration files for the flattened tables. These configs are not mandatory.
If not provided MambaETL with automatically generate these config files, one for each Encounter type. They are saved in a table (mamba_flat_table_config) in analysis database.
![table config json.png](_markdown%2Ftable%20config%20json.png)
![automatic json configs.png](_markdown%2Fautomatic%20json%20configs.png)

See image below for an example json configuration file contents:
![json-config-file.png](_markdown%2Fjson-config-file.png)

In the database you will have the following tables, And the highlighted one is the transposed table from the above Json that was provided for HTS.
![flat-table.png](_markdown%2Fflat-table.png)

For Extra analysis on the data, one can opt to create dimension and fact tables for easy analysis per encounter/service type for the client data as shown below in the Dimension folder.
![derived-facts-dims.png](_markdown%2Fderived-facts-dims.png)

<span style='color: red'>Step 7:</span>

`../pom.xml`

Extra configurations can be made in the parent `pom.xml` of your project.  
Make sure you have configured the necessary plugins, ETL source and target database names, etc.  
We advise that you look at or copy the MambaETL ref/template module root/parent [pom.xml](https://github.com/UCSF-IGHS/openmrs-module-mamba-etl/blob/main/pom.xml) file for details as there are a number of configurations in this file.  

Notably, don't forget to specify the names of your OpenMRS source database and the ETL target database in this pom.xml file.  

      <!-- The source database (OpenMRS database) -->
      <argument>-d openmrs</argument>
   
      <!-- The target or analysis Database where the ETL data is stored -->
      <argument>-a analysis_db</argument>

      <!-- Set the concepts locale name here -->
      <argument>-l en</argument>

## **The MambaETL service/API layer**
MambaETL has a service or API layer that enables users to pull out ETL or reporting data via this interface.
It is an HTTP Rest webservice interface and can be accessed via the base URL:
`<EMR_URL>/openmrs/ws/rest/v1/mamba/report`

for example:
`http://ohri-demo.globalhealthapp.net/openmrs/ws/rest/v1/mamba/report?report_id=total_deliveries`

To configure this to work, you need to add your reporting queries or entries to the <span style='color: red'>reports.json</span> file found under
the `etl module / omod (submodule) resources / etl / config (folder) / reports.json`.

An example entry can look like this:

        `{
          "report_definitions": [
            {
              "report_name": "MCH Mother HIV Status",
              "report_id": "mother_hiv_status",
              "report_sql": {
                "sql_query": "SELECT pm.hiv_test_result AS hiv_test_result FROM mamba_flat_encounter_pmtct_anc pm INNER JOIN mamba_dim_person p ON pm.client_id = p.person_id WHERE p.uuid = person_uuid AND pm.ptracker_id = ptracker_id",
                "query_params": [
                  {
                    "name": "ptracker_id",
                    "type": "VARCHAR(255)"
                  },
                  {
                    "name": "person_uuid",
                    "type": "VARCHAR(255)"
                  }
                ]
              }
            },
            {
              "report_name": "MCH Total Deliveries",
              "report_id": "total_deliveries",
              "report_sql": {
                "sql_query": "SELECT COUNT(*) AS total_deliveries FROM mamba_dim_encounter e inner join mamba_dim_encounter_type et on e.encounter_type = et.encounter_type_id WHERE et.uuid = '6dc5308d-27c9-4d49-b16f-2c5e3c759757' AND DATE(e.encounter_datetime) > CONCAT(YEAR(CURDATE()), '-01-01 00:00:00')",
                "query_params": []
              }
            },
            {
              "report_name": "MCH HIV-Exposed Infants",
              "report_id": "total_hiv_exposed_infants",
              "report_sql": {
                "sql_query": "SELECT COUNT(DISTINCT ei.infant_client_id) AS total_hiv_exposed_infants FROM mamba_fact_pmtct_exposedinfants ei INNER JOIN mamba_dim_person p ON ei.infant_client_id = p.person_id WHERE ei.encounter_datetime BETWEEN DATE_FORMAT(NOW(), '%Y-01-01') AND NOW() AND birthdate BETWEEN DATE_FORMAT(NOW(), '%Y-01-01') AND NOW()",
                "query_params": []
              }
            }
           ]
        }`

**Note that** the `report_id` value in the report.json configuration file is the same value passed to the URL parameter (report_id) in order to fetch the report corresponding to this id.

Enjoy `MambaETL` at work!
