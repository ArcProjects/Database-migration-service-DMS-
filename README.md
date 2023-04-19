#  Migrating a database with the Database MIgration Service

In this advanced demo you will be migrating a simple web application (wordpress) from an on-premises environment into AWS.  
The on-premises environment is a virtual web server (simulated using EC2) and a self-managed mariaDB database server (also simulated via EC2)  
You will be migrating this into AWS and running the architecture on an EC2 webserver and RDS managed SQL database.  

This demo consists of 6 stages :-



- STAGE 2 : Establish Private Connectivity Between the environments (VPC Peer)
- STAGE 3 : Create & Configure the AWS Side infrastructure (App and DB)
- STAGE 4 : Migrate Database & Cutover
- STAGE 5 : Cleanup the account



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

# STAGE 1 - FINISH   

Once the stack is in the `CREATE_COMPLETE` status you will have a simulated `on-premises` environment and an AWS environment.
Move to the EC2 console https://console.aws.amazon.com/ec2/v2/home?region=us-east-1  
Click `Running Instances`  
Select the `CatWEB` instance  
Copy down its `Public IPv4 DNS` into your clipboard and open it in a new tab.  
You should see the `Animals4life Hall of Fame` load... this is running from the simulated onpremises environment using the CatDB mariaDB instance.  






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


# STAGE 2 - FINISH   

At this point you have created the peering connection between the VPCs and the gateway objects within each VPC.  
you have also configured routing from ONPremises -> AWS and vice-versa.  
In stage 3 you will use this architecture to begin a migration.  




