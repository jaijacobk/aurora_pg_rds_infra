# How to Build the Aurora Postgres Global Database (Multi Region)

### Step1 (Build the stack-infra in both regions)

1.  aws cloudformation create-stack --stack-name aurora-pg-rds-infra --template-body file://stack-infra.yml --profile saml --region us-east-1 --capabilities CAPABILITY_AUTO_EXPAND
2.  aws cloudformation create-stack --stack-name aurora-pg-rds-infra --template-body file://stack-infra.yml --profile saml --region us-west-2 --capabilities CAPABILITY_AUTO_EXPAND


### Step2 (Build the stack-iam in both regions)

1.  aws cloudformation create-stack --stack-name aurora-pg-rds-iam --template-body file://stack-iam.yml --profile saml --region us-east-1 --capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_NAMED_IAM
2.  aws cloudformation create-stack --stack-name aurora-pg-rds-iam --template-body file://stack-iam.yml --profile saml --region us-west-2 --capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_NAMED_IAM


### Step3 (Build the stack-db-west in us-west-2)

1.  aws cloudformation create-stack --stack-name aurora-pg-rds-db --template-body file://stack-db-west.yml --profile saml --region us-west-2 --capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_NAMED_IAM

### Step4 (Build the stack-db-east in us-east-1)
1.  aws cloudformation create-stack --stack-name aurora-pg-rds-db --template-body file://stack-db-east.yml --profile saml --region us-east-1 --capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_NAMED_IAM

### Step5 (Fail to East and make the East Primary)
1. Fail the global cluster to East to flip the writer from West to East
2. Once the failover is complete, your applicaiton can use the East end point to connect to the database

### Step5 (Detach and Promote West in the event of a East failure)
1. Use the detach and promote to make the West the primary
2. Once the detach is complete, your applicaiton can use the West end point to connect to the database

### Step6 (Fallback to East)
1. Delete the East Stack
2. Change the name of the global cluster (add a -1 at the end)
3. Recreate the East CloudFormation stack
4. Once this is over, fail to East to make the East primary again
