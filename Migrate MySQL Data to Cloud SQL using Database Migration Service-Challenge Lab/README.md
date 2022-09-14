# Migrate MySQL Data to Cloud SQL using Database Migration Service: Challenge Lab

lab: https://www.cloudskillsboost.google/focuses/20393?parent=catalog

### Task 1. Configure a Database Migration Service connection profile for a stand-alone MySQL database

#### Database Migration API -> Enable

#### Navigation menu -> Database Migration -> Database Migration

#### CREATE PROFILE

#### Database engine -> MySQL



| Property | Value |
| -------- | -------- |
| Username | admin |
| Password | changeme |
| *hostname | changeme |
| others | default |


 
#### (*hastname) Navigation menu -> Compute Engine -> VM instances -> instance -> External IP
	
### Task 2. Perform a one-time migration of a stand-alone MySQL database to Cloud SQL

#### Navigation menu -> Database Migration -> Migration jobs

#### CREATE MIGRATION JOB

#### Get Start
| Property | Value |
| -------- | -------- |
| Cloud SQL Destination Instance ID | mysql-fin-372 |

#### Souce database engine -> MySQL

#### Migration job type -> One-time

#### Other Step

| Property | Value |
| -------- | -------- |
| Cloud SQL Destination Instance ID | mysql-fin-372 |
| Root password | supersecret! |
| Database version | Cloud SQL for MySQL 5.7 |
| Machine type | Standard |
| CPU	1 vCPU | 3.75GB |
| Storage type | SSD |
| Storage capacity | 10GB |
	
#### VNC peer -> default

#### Create -> Start

#### Navigation menu -> SQL -> select instance -> Open Cloud Shell(into mysql)

*Cloud Shell*

`use customers_data;
select count(*) from customers;`

### Task 3. Create a continuous Database Migration Service migration job to migrate a stand-alone MySQL database to Cloud SQL

#### Get Start
| Property | Value |
| -------- | -------- |
| Cloud SQL Destination Instance ID | mysql-fin-372 |

#### Souce database engine -> MySQL

#### Migration job type -> Continuous

#### Other Step

| Property | Value |
| -------- | -------- |
| Cloud SQL Destination Instance ID | mysql-fin-xxx-con |
| Root password | supersecret! |
| Database version | Cloud SQL for MySQL 5.7 |
| Machine type | Standard |
| CPU	1 vCPU | 3.75GB |
| Storage type | SSD |
| Storage capacity | 10GB |
	
#### VNC peer -> default

#### Create -> Start

#### Navigation menu -> SQL -> select instance -> Open Cloud Shell(into mysql)

### Task 4. Test that the continuous Database Migration Service job replicates updated source data

#### Navigation menu -> Compute Engine -> VM instances -> instance -> SSH

*SSH*

`mysql -u admin -p`
password `changeme`

`use customers_data;
update customers set gender = 'FEMALE' where addressKey = 934;`

### Task 5. Promote the destination Cloud SQL for MySQL database to a stand-alone database

#### Navigation menu -> Database Migration -> Migration jobs -> select continuous job 

#### Promote
	



