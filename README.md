Build using AWS CodeBuild, CodeDeploy, CodePipeline


## Prepare before create CodeBuild

### 1. Create a Buildspec 

#### Case 1. Use building source codes from S3 

```
# Download a repository you want to build on CodeBuild using git clone.
# the example repository
$ git clone https://github.com/lightbitbird/akka-playground.git

# Create the s3 bucket for uploading source codes 
$ aws s3 mb s3://akka-playground-codebuild-input

# Create buildspec.yml
$ touch buildspec.yml

# Upload the zipped file
$ zip akka-playground.zip ./
$ aws s3 cp akka-playground.zip s3://akka-playground-codebuild-input
```

#### Case 2. Use building source codes from Github

You need to include a buildspec.yml on the git repository.


### 2. Create the S3 bucket for uploading a packaged zip file after building codes.

```
# the example repository
$ aws s3 mb s3://akka-playground-codebuild-output

```

### 3. Start building from the managed console or AWS CLI
- Artifact: Amazon S3
- Bucket Name: S3 bucket created at 1
- package: zip
- additonal setting: 

    - Cache type: local

    - SourceCache: on

    - CustomeCache: on


### 4. Delete S3 input bucket( *Case 1 )

```
aws s3 rb s3://akka-playground-codebuild-input --force
```

#

## CodeDeploy

### 1. Create CodeDeploy Role

```
# Create CodeDeploy Role
$ vim CodeDeployDemo-Trust.json
# add iam role below 
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "codedeploy.ap-northeast-1.amazonaws.com",
                    "codedeploy.ap-northeast-2.amazonaws.com"
                ]
            },
            "Action": "sts:AssumeRole"
        }
    ]
}

$ aws iam create-role --role-name CodeDeployServiceRole --assume-role-policy-document file://CodeDeployDemo-Trust.json


# Create EC2 Instance Role
$ vim CodeDeployDemo-EC2-Trust.json
# add iam role below 
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}

$ aws iam create-role --role-name CodeDeployDemo-EC2-Instance-Profile --assume-role-policy-document file://CodeDeployDemo-EC2-Trust.json


# Put Role access permissions to EC2 Instance
$ vim CodeDeployDemo-EC2-Permissions.json
# add iam role below 
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "s3:Get*",
                "s3:List*"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::akka-playground-codebuild-output/*",
                "arn:aws:s3:::codepipeline-ap-northeast-1-xxxxxxxxxxxx/*",
                "arn:aws:s3:::aws-codedeploy-ap-northeast-1/*",
                "arn:aws:s3:::aws-codedeploy-ap-northeast-2/*"
	        ]
        }
    ]
}

$ aws iam put-role-policy --role-name CodeDeployDemo-EC2-Instance-Profile --policy-name CodeDeployDemo-EC2-Permissions --policy-document file://CodeDeployDemo-EC2-Permissions.json

# Create the EC2 instance profile
$ aws iam create-instance-profile --instance-profile-name CodeDeployDemo-EC2-Instance-Profile

# Add A role to the EC2 instance profile
$ aws iam add-role-to-instance-profile --instance-profile-name CodeDeployDemo-EC2-Instance-Profile --role-name CodeDeployDemo-EC2-Instance-Profile


```

### 2. Create EC2 instance with CodedeployDemo-EC2-Instance-Role

- AMI: Select Amazon Linux 2
- IAM Role: Select IAM Role created at 1
- Security Group: Open HTTP through 80 port on inbound
- Tag: add tags [Name/CodeDeployDemo, deploy/01]


### 3. Install CodeDeploy agent on the EC2 instance

Go to EC2 instance you created at 2 with ssh and install on the EC2 instance

```
# Create a install script.
vim install.sh

sudo yum update
sudo yum install ruby
sudo yum install wget
cd /home/ec2-user
wget https://aws-codedeploy-ap-northeast-1.s3.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto

chmod 755 install.sh
./install.sh

# Install OpenJdk 11
wget https://d3pxv6yz143wms.cloudfront.net/11.0.3.7.1/java-11-amazon-corretto-devel-11.0.3.7-1.x86_64.rpm
sudo yum localinstall java-11-amazon-corretto-devel-11.0.3.7-1.x86_64.rpm
```


### 4. Create CodeDeploy
You need to include a appspec.yml on the git repository.

- IAM Role: `CodeDeployServiceRole`

- Deploy Group: Select `deploy` created the above step.

