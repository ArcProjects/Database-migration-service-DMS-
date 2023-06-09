#  Migrating a database with the Database Migration Service

In this demo you will be migrating a simple web application (wordpress) from an on-premises environment into AWS.  
The on-premises environment is a virtual web server (simulated using EC2) and a self-managed mariaDB database server (also simulated via EC2)  
You will be migrating this into AWS and running the architecture on an EC2 webserver and RDS managed SQL database.  

This demo consists of 5 stages :-


- [STAGE 1 : Provision the environment and review task](https://github.com/ArcProjects/Database-migration-service-DMS-#stage-1--provision-the-environment-and-review-task)
- [STAGE 2 : Establish Private Connectivity Between the environments (VPC Peer)](https://github.com/ArcProjects/Database-migration-service-DMS-#stage-2--establish-private-connectivity-between-the-environments-vpc-peer)
- [STAGE 3 : Create & Configure the AWS Side infrastructure (App and DB)](https://github.com/ArcProjects/Database-migration-service-DMS-#stage-3--create--configure-the-aws-side-infrastructure-app-and-db)
- [STAGE 4 : Migrate Database & Cutover](https://github.com/ArcProjects/Database-migration-service-DMS-#stage-4--migrate-database--cutover)
- [STAGE 5 : Cleanup the account](https://github.com/ArcProjects/Database-migration-service-DMS-#stage-5--cleanup-the-account)

---
# STAGE 1 : Provision the environment and review task
---

# STAGE 1A - Infrastructure Creation



-![dms-stage1](https://user-images.githubusercontent.com/90862957/232962907-d7bd9da4-eb04-4a17-99e9-473016df225a.png)


Click https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateURL=https://learn-cantrill-labs.s3.amazonaws.com/aws-dms-database-migration/DMS.yaml&stackName=DMS to apply the base lab infrastructure  

You should take note of the `parameter` values 

- DBName
- DBPassword
- DBRootPassword
- DBUser

You will need all of these in later stages.  
All defaults should be pre-populated, you just need to scroll to the bottom, check the capabilities box and click `Create Stack`  
---
# STAGE 1 - FINISH   

Once the stack is in the `CREATE_COMPLETE` status you will have a simulated `on-premises` environment and an AWS environment.
Move to the EC2 console https://console.aws.amazon.com/ec2/v2/home?region=us-east-1  
Click `Running Instances`  
Select the `CatWEB` instance  
Copy down its `Public IPv4 DNS` into your clipboard and open it in a new tab.  
You should see the `Animals4life Hall of Fame` load... this is running from the simulated onpremises environment using the CatDB mariaDB instance.  

----
# STAGE 2 : Establish Private Connectivity Between the environments (VPC Peer)
-----

# STAGE 2A - Create a VPC peer between On-Premises and AWS

![dms-stage2](https://user-images.githubusercontent.com/90862957/232962776-467ba062-5238-4054-a683-fdbeebf68499.png)

Move to the VPC Console https://console.aws.amazon.com/vpc/home?region=us-east-1#  
Click on `Peering Connections` under `Virtual Private Cloud`  
Click `Create Peering Connection`  
for `Peering connection name tag` choose `A4L-ON-PREMISES-TO-AWS`  
for `VPC (Requester)` choose `onpremVPC`  
for `VPC (Accepter)` choose `awsVPC`  
Scroll down and click `Create Peering Connection`  
...then click `Actions` and then `Accept Request`  
Click `Accept Request`  
 
---

# STAGE 2B - Create Routes on the On-premises side
Move to the route tabes console https://console.aws.amazon.com/vpc/home?region=us-east-1#RouteTables:sort=routeTableId  
Locate the `onpremPublicRT` route table and select it using the checkbox.  
Click on the `Routes` Tab.  
You're going to add a route pointing at the AWS side networking, using the VPC Peer.  
Click `Edit Routes`  
Click `Add Route`  
For Destination enter `10.16.0.0/16`  
Click the `Target` dropdown & click `Peering Connection` and select the `A4L-ON-PREMISES-TO-AWS` then click `Save Changes`  
The Onpremises network can now route to the AWS Network, but as data transfer requires bi-directional traffic flow, you need to do the same at the other side.

---

# STAGE 2C - Create Routes on the AWS side
Move to the route tabes console https://console.aws.amazon.com/vpc/home?region=us-east-1#RouteTables:sort=routeTableId  
Locate the `awsPublicRT` route table and select it using the checkbox.  
Click on the `Routes` Tab.  
You're going to add a route pointing at the AWS side networking, using the VPC Peer.  
Click `Edit Routes`  
Click `Add Route`  
For Destination enter `192.168.10.0/24`  
Click the `Target` dropdown & click `Peering Connection` and select the `A4L-ON-PREMISES-TO-AWS` then click `Save Changes`  

Move to the route tabes console https://console.aws.amazon.com/vpc/home?region=us-east-1#RouteTables:sort=routeTableId  
Locate the `awsPrivateRT` route table and select it using the checkbox.  
Click on the `Routes` Tab.  
You're going to add a route pointing at the AWS side networking, using the VPC Peer.  
Click `Edit Routes`  
Click `Add Route`  
For Destination enter `192.168.10.0/24`  
Click the `Target` dropdown & click `Peering Connection` and select the `A4L-ON-PREMISES-TO-AWS` then click `Save Changes`  

---
# STAGE 2 - FINISH   

At this point you have created the peering connection between the VPCs and the gateway objects within each VPC.  
you have also configured routing from ONPremises -> AWS and vice-versa.  
In stage 3 you will use this architecture to begin a migration.  

---
# STAGE 3 : Create & Configure the AWS Side infrastructure (App and DB)
----


# STAGE 3A - CREATE THE RDS INSTANCE


![dms-stage3](https://user-images.githubusercontent.com/90862957/232963928-f233e250-7d42-4cd2-8fb3-b70259bcabed.png)

Move to the RDS Console https://console.aws.amazon.com/rds/home?region=us-east-1  
Click `Subnet Groups`  
Click `Create DB Subnet Group`  
For `Name` call it `A4LDBSNGROUP`
enter the same for description
in the `VPC` dropdown, choose `awsVPC`  
Under availability zones, choose `us-east-1a` and `us-east-1b`  
for subnets check the box next to `10.16.32.0/20` which is privateA and `10.16.96.0/20` which is privateB  
Scroll down and click `Create`  
Click on `Databases`  
Click `create Database`  
Choose `Standard Create`  
Choose `MariaDB`
Choose `Free Tier` for `Templates`  

You will be using the same database names and credentials to keep things simple for this demo lesson, but note that in production this could be different.

for `DB instance identifier` enter `a4lwordpress`  
for `Master username` choose `a4lwordpress`  
for `Masterpassword` enter the DBPassword parameter for cloudformation which you noted down in stage of this demo
enter that same password in the `Confirm password` box  
Scroll down to `Connectivity`  
for `Virtual private cloud (VPC)` choose `awsVPC`  
make sure `Subnet Groups` is set toe `a4ldbsngroup`  
for `public access` choose `No`  
for `VPC security groups` select `Choose Existing` and choose  `***-awsSecurityGroupDB-***` (*** aren't important)  
remove the `Default` security group by clicking the `X`    
Scroll down and expand `Additional configuration`  
Under `Initial database name` enter `a4lwordpress`  
Scroll down and click `Create Database`  

This will take some time.. and you cant continue to `Stage4` until the database is in a ready state.
----
# STAGE 3B - CREATE THE EC2 INSTANCE

Move to the EC2 Console https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Home:  
Click `Instances`  
Click `Launch Instances`  
Enter `awsCatWeb` for the Name of the instance.  
Clck `Amazon Linux` and ensure `Amazon Linux 2 AMI (HVM) ...... SSD Volume Type` is selected.  
Ensuring the architecture is set to `64-bit (x86)` below.    
Choose the `Free Tier eligable` instance (should be t2.micro or t3.micro)  
Scroll down and choose `Proceed without key pair (not recommended)` in the dropdown  
Next to `Network Settings` click `Edit`  
For `VPC` pick `awsVPC`  
For `Subnet` pick `aws-PublicA`  
Select `Select an existing security group`  
Choose `***-awsSecurityGroupWeb-***` (*** aren't important)  
Scroll down past Storage and expand `Advanced Details` (don't confuse this with `Advanced Network Configuration` in the current area)  
for `IAM Instance Profile` pick `***-awsInstanceProfile-****` (*** aren't important)  
Click `Launch Instance`  
Click `View All Instances`  

Wait for the `awsCatWeb` instance to be in a `Running` state with `2/2 checks` before continuing.
------

# STAGE 3C - INSTALL WORDPRESS Requirements

Select the `awsCatWeb` instance, right click, `Connect`  
Select `Session Manager` and click `Connect`  
When connected type `sudo bash` to run a privileged bash shell
then update the instance with a `yum -y update` and wait for it to complete.  
Then install the apache web server with `yum -y install httpd mariadb`  (the mariadb part is for the mysql tools)
Then install php with `amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2 `  
then make sure apache is running and set to run at startup with 

```
systemctl enable httpd
systemctl start httpd
```

You now have a running apache web server with the ability to connect to the wordpress database (currently running onpremises)
----

# STAGE 3D - MIGRATE WORDPRESS Content over

You're going to edit the SSH config on this machine to allow password authentication on a temporary basis.  
You will use this to copy the wordpress data across to the awsCatWeb machine from the on-premises CatWeb Machine  

run a `nano /etc/ssh/sshd_config`  
locate `PasswordAuthentication no` and change to `PasswordAuthentication yes` , then `ctrl+o` to save and `ctrl+x` to exit.  
then set a password on the ec2-user user  
run a `passwd ec2-user` and enter the `DBPassword` you noted down at the start of the demo.  
**this is only temporary.. we're using the same password throughout the demo to make things easier and less prone to mistakes**

restart SSHD to make those changes with `service sshd restart`  or `systemctl restart sshd`


Return back to the EC2 console https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:  
with the `awsCatWeb` instance selected, note down its `Private IPV4 Address` you will need this in a moment.  

Select the `CatWeb` instance, right click, `Connect`  
Select `Session Manager` and click `Connect`  
When connected type `sudo bash` to run a privileged bash shell  

move to the webroot folder by typing `cd /var/www/`  

run a `scp -rp html ec2-user@privateIPofawsCatWeb:/home/ec2-user` and answer `yes` to the authenticity warning.  
this will copy the wordpress local files from `CatWeb` (on-premises) to `awsCatWeb` (aws)

**now move back to the `awsCatWeb` server, if you dont have it open still, reconnect as per below**

Select the `awsCatWeb` instance, right click, `Connect`  
Select `Session Manager` and click `Connect`  
When connected type `sudo bash` to run a privileged bash shell

move to the `ec2-user` home folder by doing a `cd /home/ec2-user`  
then do an `ls -la` and you should see the html folder you just copied.  
`cd html`  
next copy all of these files into the webroot of `awsCatWeb` by doing a `cp * -R /var/www/html/`

----

# STAGE 3E - Fix Up Permissions & verify `awsCatWeb` works

run the following commands to enforce the correct permissions on the files you've just copied across

```
usermod -a -G apache ec2-user   
chown -R ec2-user:apache /var/www
chmod 2775 /var/www
find /var/www -type d -exec chmod 2775 {} \;
find /var/www -type f -exec chmod 0664 {} \;
sudo systemctl restart httpd
```

Move to the EC2 running instances console https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:  
Select the `awsCatWeb` instance  
copy down its `public IPv4 DNS` into your clipboard and open it in a new tab  
if working, this Web Instance (aws) is now loading using the on-premises database.

---

# STAGE 3 - FINISH   

At this point you have a functional AWS based wordpress application instance.  
You have migrated content from the on-premises virtual machine (simulated) using SCP.  
And you have tested that its connects to the on-premises DB.
In the next stage you will migrate the on-premises DB to AWS using DMS.
**before you continue, make sure the a4lwordpress RDS DB is in an Available state** https://console.aws.amazon.com/rds/home?region=us-east-1#databases:  

---
# STAGE 4 : Migrate Database & Cutover
---

# STAGE 4A - CREATE THE DMS SUBNET GROUP


![dms](https://user-images.githubusercontent.com/90862957/232964406-1c7f1af4-3ec8-4142-8f71-863af47d0c7f.png)


https://console.aws.amazon.com/dms/v2/home?region=us-east-1#subnetGroup  
Click `Create Subnet Group`  
For Name and Description use `A4LDMSSNGROUP`
for `VPC` choose `awsVPC`  
for `Add Subnets` choose `aws-privateA` and `aws-privateB`  
Click `Create subnet group` 
 

# STAGE 4B - CREATE THE DMS REPLICATION INSTANCE

Move to the DMS Console https://console.aws.amazon.com/dms/v2/home?region=us-east-1#replicationInstances  
Click `create Replication Instance`  
for name enter `A4LONPREMTOAWS`
use the same for `Description`  
for `Instance Class` choose `dms.t3.micro`  
for `VPC` choose `awsVPC`  
For `MultiAZ` make sure it's set to `dev or test workload (single AZ)`  
and for `Pubicly Accessible` uncheck the box  
Expand `Advanced security and network configuration`  
Ensure `a4ldmssngroup` is selected in `Replication subnet group`  
For `VPC security group(s)` choose `***-awsSecurityGroupDB-***`  
Click `Create`  

# STAGE 4C - CREATE THE DMS SOURCE ENDPOINT
Move to https://console.aws.amazon.com/dms/v2/home?region=us-east-1#endpointList  
Click `Create Endpoint`  
For `Endpoint type` choose `Source Endpoint` and make sure that `Select RDS DB Instance` is UNCHECKED  
Under `Endpoint configuration` set `Endpoint identifier` to be `CatDBOnpremises`  
Under `Source Engine` set `mariadb`  
Under `Access to endpoint database` choose `Provide access information manually`  
Under `Server name` use the privateIPv4 address of `CatDB` (get it from EC2 console)  
For port `3306`  
For username `a4lwordpress`  
for password user the DBPassword you noted down in stage 1  
click `create endpoint`  

# STAGE 4D - CREATE THE DMS DESTINATION ENDPOINT (RDS)  
Move to https://console.aws.amazon.com/dms/v2/home?region=us-east-1#endpointList  
Click `Create Endpoint`  
For `Endpoint type` choose `Target Endpoint`  
Check `Select RDS DB Instance`  
Select `a4lwordpress` in the dropdown  
It will prepopulate the boxes  
Under `Access to endpoint database` choose `Provide access information manually`  
For `Password` enter the DBPassword you noted down in stage1  
Scroll down and click `Create Endpoint`  

# STAGE 4E - TEST THE ENDPOINTS

**make sure the replication instance is ready**
Verify by going to `Replication Instances` and make sure the status is `Available`  
Go back to `Endspoints`  

Select the `a4lwordpress` endpoint, click `Actions` and then `Test Connections`  
Click `Run Test` and make sure after a few minutes the status moves to `successful`  
Go back to `Endpoints`  
Select the `catdbonpremises` endpoint, click `Actions` and then `Test Connections`  
Click `Run Test` and make sure after a few minutes the status moves to `successful`  

If both of these are successful you can continue to the next step.  

# STAGE 4F - Migrate
Move to migration tasks  https://console.aws.amazon.com/dms/v2/home?region=us-east-1#tasks  
Click `Create task`  
for `Task identifier` enter `A4LONPREMTOAWSWORDPRESS`
for `Replication instance` pick the replication instance you just created  
for `Source database endpoint` pick `catdbonpremises`  
for `Target database endpoint` pick `a4lwordpress`  
for `Migration type` pick `migrate existing data` **you could pick and replicate changes here if this were a high volume production DB**  
for `Table mappings` pick `Wizard`  
Click `Add new selection rule`  
in `Schema` box select `Enter a Schema`  
in `Schema Name` type `a4lwordpress`  
Scroll down and click `Create Task`  

This starts the replication task and does a full load from `catdbonpremises` to the RDS Instance.  
It will create the task  
then start the task  
then it will be in the `Running` State until it moves into `Load complete`  


At this point the data has been migrated into the RDS instance  

# STAGE 4G - Cutover the application instance

Move to the RDS Console https://console.aws.amazon.com/rds/home?region=us-east-1#  
Click `Databases`  
Click `a4lwordpress`  
under `Endpoint & Port` copy the `endpoint` dns name into your clipboard  

Move to the EC2 console https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:  
Select the `awsCatWeb` instance, right click, `Connect`  
Select `Session Manager` and click `Connect`  
When connected type `sudo bash` to run a privileged bash shell
run `cd /var/www/html`
run `nano wp-config.php`  
locate the define DB_HOST line, and reolace the IP address with the RDS Host you just copied down into you clipboard  
run `ctrl+o` to save and `ctrl+x` to exit.  

Run the script below, to update the wordpress database with the new instance DNS name

```
#!/bin/bash
source <(php -r 'require("/var/www/html/wp-config.php"); echo("DB_NAME=".DB_NAME."; DB_USER=".DB_USER."; DB_PASSWORD=".DB_PASSWORD."; DB_HOST=".DB_HOST); ')
SQL_COMMAND="mysql -u $DB_USER -h $DB_HOST -p$DB_PASSWORD $DB_NAME -e"
OLD_URL=$(mysql -u $DB_USER -h $DB_HOST -p$DB_PASSWORD $DB_NAME -e 'select option_value from wp_options where option_id = 1;' | grep http)
HOST=$(curl http://169.254.169.254/latest/meta-data/public-hostname)
$SQL_COMMAND "UPDATE wp_options SET option_value = replace(option_value, '$OLD_URL', 'http://$HOST') WHERE option_name = 'home' OR option_name = 'siteurl';"
$SQL_COMMAND "UPDATE wp_posts SET guid = replace(guid, '$OLD_URL','http://$HOST');"
$SQL_COMMAND "UPDATE wp_posts SET post_content = replace(post_content, '$OLD_URL', 'http://$HOST');"
$SQL_COMMAND "UPDATE wp_postmeta SET meta_value = replace(meta_value,'$OLD_URL','http://$HOST');"
```

Move to the EC2 console https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:  
Select `CatDB`, right click `stop instance`   
Move to the EC2 console https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:  
Select `CatWeb`, right click `stop instance`   

Move to the EC2 console https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:  
Select `awsCatWeb` and get its `Public IPv4 DNS` 
Open this in a new tab
It should still show the application ... this is now pointed at the RDS instance after a full migration

# STAGE 4 Finish

![dms-endstage](https://user-images.githubusercontent.com/90862957/232964553-26beda6a-cd42-4da8-9ec8-d529456fe2c6.png)


At this point you have created a VPC peer between the simulated On-premises environment and AWS
You have fully migrated the wordpress application files from on-premises (simulated) into AWS  
You have provisioned an RDS DB Instance  
And you have used DMS to perform a simple migration of the database from on-premises (simulated) to AWS  



-----
# STAGE 5 : Cleanup the account
-----

# STAGE 5A - EC2




Move to the EC2 Console https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:  
Select `awsCatWeb`, right click, instance state, terminate instance, terminate  

# STAGE 5B - RDS
Move to the RDS Console https://console.aws.amazon.com/rds/home?region=us-east-1#databases:  
Click `Databases`  
Select `a4lwordpress`  
Click `Actions` then `Delete`  
Uncheck `Create final snapshot?`  
Check the `I acknowledge...`, type `delete me` into the box and cick `Delete`  

**wait for this to delete**
Move to https://console.aws.amazon.com/rds/home?region=us-east-1#db-subnet-groups-list:  
Select the `a4ldbsngroup` click `Delete` then `Delete` again

# STAGE 5C- DMS
go to DMS console https://console.aws.amazon.com/dms/v2/home?region=us-east-1#tasks  
Select the `a4lonpremtoawswordpress` task, click `Actions`, `Delete` and then `Delete` again.

Got to https://console.aws.amazon.com/dms/v2/home?region=us-east-1#replicationInstances  
Delect the `a4lonpremtoaws` replication instance,  click `Actions`, `Delete` and then `Delete` again.

Move to https://console.aws.amazon.com/dms/v2/home?region=us-east-1#endpointList  
Select one of the endpoints, click `Actions`, `Delete` and then `Delete` again.
Select the other endpoint, click `Actions`, `Delete` and then `Delete` again.

**wait for the replication instance to finish deleting**
Move to https://console.aws.amazon.com/dms/v2/home?region=us-east-1#subnetGroup  
Select `a4ldmssngroup` click `Actions`, `Delete` and then `Delete` again.

# STAGE 5D - VPC PEER

Move to the VPC Console https://console.aws.amazon.com/vpc/home?region=us-east-1  
Click `Route Tables`  
Select `awsPrivateRT`  
Click `Routes`  
Click `Edit Routes`  
Click `X` next to the VPC Peer Route  
Click `Save Routes` and click `Close`  
Select `awsPublicRT`  
Click `Routes`  
Click `Edit Routes`  
Click `X` next to the VPC Peer Route  
Click `Save Routes` and click `Close`  
Select `onpremPublicRT`  
Click `Routes`  
Click `Edit Routes`  
Click `X` next to the VPC Peer Route  
Click `Save Routes` and click `Close`  

Move to VPC Peering Connections https://console.aws.amazon.com/vpc/home?region=us-east-1#PeeringConnections:sort=vpcPeeringConnectionId  
Select the `VPC Peer` you created earlier `A4L-ON-PREMISES-TO-AWS`, click `Actions`, `Delete VPC Peering Connection`, Click `Yes, Delete`  

# STAGE 5E - CLOUDFORMATION STACK

Move to the cloudformation console https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks?filteringText=&filteringStatus=active&viewNested=true&hideStacks=false   
Select the `DMS` stack  
Click `Delete` then `Delete Stack`  


