# This hands-on project will guide you through the process of migrating SQL data into an Amazon RDS database using FLYWAY.
 In this project you would learn how to setup the necessary resources to migrate data to RDS using FLYWAY this includes setting up your VPC,Natgateway,securitygroup,RDS,Keypair,Bastion host and FLYWAY.
 To use FLYWAY to migrate data into RDS instance in the private subnet,first we would have to download and configure flyway on our computer ,after that we would launch a Bastionhost in the public subnet once we have launched the Bastion host in the public subnet we will create an SSH tunnel to the rds instance through the Bastion host .
 Once the SSH tunnelis setup, we will use flyway to migrate the data into the RDS database in the private subnet
 

STEP 1
## Building a 3-tier VPC with public and private subnets in two Availability Zones (AZs) using AWS CloudFormation can be done using the following steps:Building a 3-tier VPC with public and private subnets in 2 AZs

Create a CloudFormation template in JSON or YAML format that defines the resources for your VPC, including the VPC itself, subnets, security groups, Elastic IP addresses, load balancers, RDS instances, and EBS volumes.

### Use the AWS::EC2::VPC resource type to define the VPC,
syntax:
```
  vpc:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: !Ref vpccidrblock
      EnableDnsSupport: 'true'
      EnableDnsHostnames: 'true'
      Tags:
        - Key: stack
          Value: production
```

### and use the AWS::EC2::Subnet resource type to define the public, private and database subnets in each AZ.
syntax:
```
 PublicSubnet1:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId:
        Ref: vpc
      CidrBlock: 10.0.0.0/24
      MapPublicIpOnLaunch: true
      AvailabilityZone: us-east-1a
      Tags:
        - Key: stack
          Value: production

  DatabaseSubnet1:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId:
        Ref: vpc
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: us-east-1a
      Tags:
        - Key: stack
          Value: production

  PrivateSubnet1:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId:
        Ref: vpc
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: us-east-1a
      Tags:
        - Key: stack
          Value: production
```          

### Use the AWS::EC2::SecurityGroup resource type to define the security groups for the public and private subnets,and use the AWS::EC2::SecurityGroupIngress resource type to specify the inbound traffic rules.
syntax:
```
  databasesg:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupDescription: Allow ssh to client database
      VpcId: !Ref vpc
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          SourceSecurityGroupId: !Ref bastionhostsg

  bastionhostsg:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupDescription: Allow http to client host
      VpcId: !Ref vpc
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 102.89.23.192/32

```
