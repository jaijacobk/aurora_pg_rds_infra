# How to Build the Multi Regional Aurora Postgres Global Database With an Automatic Failover and Fallback

Aurora Global Database supports managed planned failovers, you can manually invoke a process that promotes one of the existing secondary regions to be the new primary region. A managed planned failover, however, requires a healthy global database cluster.   

An unplanned event occurs when the primary region becomes unhealthy. Unfortunately, there is no AWS-orchestrated automated solution available to promote the secondary region and bring the database up and running. This project illustrates how to acheive this by building the RDS cluster in a certain way along with a series of Lambdas and a step function

![Screenshot](images/image_1.png)
  
### Step:1 (Build the stack-infra in both regions)

1.  aws cloudformation create-stack --stack-name aurora-pg-rds-infra --template-body file://stack-infra.yml --profile saml --region us-east-1 --capabilities CAPABILITY_AUTO_EXPAND
2.  aws cloudformation create-stack --stack-name aurora-pg-rds-infra --template-body file://stack-infra.yml --profile saml --region us-west-2 --capabilities CAPABILITY_AUTO_EXPAND


#### This will create the following
1. A custom KMS Key for the Database encryption and Secret manager encryption. The ARN of the the KMS Key will be exported to a parameter store entry called /demo/rds/kmskey/arn
2. Secret Manager entries called 'theadmin' and  'theuser'. These secretes will be exported the parameter store (demo/rds/theadmin/secret and demo/rds/theuser/secret respectively)
3. A security group called 'LambdaSecurityGroup' for the Lambdas to manage 443 traffic. The value of the securify group will be exported a parameter store entry called /demo/rds/sg/lambda-security-group
4. A parameter store entry called '/demo/rds/global/cluster/name' to store the Gloabl Cluster Name. We will be changing this value when we do a fallback after an unplanned failover.


### Step:2 (Build the stack-iam in both regions)

1.  aws cloudformation create-stack --stack-name aurora-pg-rds-iam --template-body file://stack-iam.yml --profile saml --region us-east-1 --capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_NAMED_IAM
2.  aws cloudformation create-stack --stack-name aurora-pg-rds-iam --template-body file://stack-iam.yml --profile saml --region us-west-2 --capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_NAMED_IAM


#### This will create the following
1. An IAM role for all your Lambdas to connect to the database
2. The ARN of the IAM role will be exported a parameter store entry called /demo/rds/iam/lamdaexecutionrole

### Step:3 (Build the stack-db-west in us-west-2)

1.  aws cloudformation create-stack --stack-name aurora-pg-rds-database --template-body file://stack-db-west.yml --profile saml --region us-west-2 --capabilities CAPABILITY_AUTO_EXPAND 

#### This will create the following
1. A global Cluster
2. Two instances of the database in the WEST where one of the instances will be a WRITER


### Step:4 (Build the stack-db-east in us-east-1)
1.  aws cloudformation create-stack --stack-name aurora-pg-rds-database --template-body file://stack-db-east.yml --profile saml --region us-east-1 --capabilities CAPABILITY_AUTO_EXPAND 

#### This will create the following
1. A new cluster which will be added to the Global Cluster created in the West (above step)
2. Two READER instances of the databae in the east  

![Screenshot](images/image_2.png)  


### Step:5 (Planned Failover/Fallback and Unplanned Failover Lambdas)
1. aws cloudformation create-stack --stack-name aurora-pg-rds-lambdas --template-body file://stack-lambdas.yml --profile saml --region us-east-1 --capabilities CAPABILITY_AUTO_EXPAND 
2. aws cloudformation create-stack --stack-name aurora-pg-rds-lambdas --template-body file://stack-lambdas.yml --profile saml --region us-west-2 --capabilities CAPABILITY_AUTO_EXPAND 

#### This will create the following
1. A lambda called 'demo-lambda-dev-rds-infra-planned-failover-2-east' to manully promote the East as the primary (Deployed to EAST only)
2. A lambda called 'demo-lambda-dev-rds-infra-planned-failover-2-west' to manully promote the West as the primary (Deployed to WEST only)
3. A lambda called 'demo-lambda-dev-rds-infra-detach-and-promote-west' to handle an Unplanned Failover event (Deployed to WEST only)


### Step:6 (Do a Planned Failover to the East and make the East Primary)
1. Fail the global cluster to East to flip the writer from West to East by invoking the lamdas as follows
    aws lambda invoke --function-name arn:aws:lambda:us-east-1:{accountid}:function:demo-lambda-dev-rds-infra-planned-failover-2-east --region us-east-1  --log-type Tail ~/lambda.log
2. Once the failover is complete, your application can use the East end point to connect to the database

![Screenshot](images/image_3.png)  

Now the database is all ready and your applications can start read/write to the databaase using the appropriate end points as shown below

![Screenshot](images/image_4.png)  


### Step:7 (Unplanned Failover )

I will leave upto your imagination to mark the primary region database instance as unhealthy. Here is something came up with 
    1. Have an even rule trigger a lamdba, which connects to the writer and do and update opration
    2. If you get 5 consecutive errors, you can assume that the db instance is irresponsive and it is time to failover to the West

Once you mark the primary as unhealthy, you can start the failover process by invoking the lambda "demo-lambda-dev-rds-infra-detach-and-promote-west" as below  
    aws lambda invoke --function-name arn:aws:lambda:us-west-2:{accountid}:function:demo-lambda-dev-rds-infra-detach-and-promote-west --profile saml --region us-west-2 --log-type Tail ~/lambda.log  
The end result will be the following (a standalone database in the WEST with a READ and WRITE end point)  

![Screenshot](images/image_5.png)  


### Step6 (Fallback to East)
1. Delete the East Stack (do it from the cloudformation)
2. Change the name of the global cluster from the parameter store (/demo/rds/global/cluster/name)
3. Update the West Cloudformation stack

aws cloudformation update-stack --stack-name aurora-pg-rds-db --template-body file://stack-db-west.yml --profile saml --region us-west-2 --capabilities CAPABILITY_AUTO_EXPAND 

4. Recreate the East CloudFormation stack

aws cloudformation create-stack --stack-name aurora-pg-rds-db --template-body file://stack-db-east.yml --profile saml --region us-east-1 --capabilities CAPABILITY_AUTO_EXPAND 

5. Repeat Step:6
