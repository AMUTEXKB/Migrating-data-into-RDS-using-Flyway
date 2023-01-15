# This hands-on project will guide you through the process of migrating SQL data into an Amazon RDS database using FLYWAY.
This project will teach you how to set up the resources required to migrate data to RDS using FLYWAY. This includes configuring your VPC, Natgateway, securitygroup, RDS,Keypair,Bastion host, and FLYWAY. To use FLYWAY to migrate data into an RDS instance in the private subnet, first download and configure flyway on our computer, then launch a Bastion host in the public subnet. Once the Bastion host is launched in the public subnet, we will create an SSH tunnel to the RDS instance via the Bastion host. We will use flyway to migrate the data into the RDS database in the private subnet once the SSH tunnel is established.

STEP 1
## Building a 3-tier VPC with public and private subnets in two Availability Zones (AZs) using AWS CloudFormation can be done using the following steps:Building a 3-tier VPC with public and private subnets in 2 AZs

Create a CloudFormation template in JSON or YAML format that defines the resources for your VPC, including the VPC itself, subnets, security groups, Elastic IP addresses, load balancers, RDS instances, and EBS volumes.

### 1 Use the AWS::EC2::VPC resource type to define the VPC,
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
### 2 Use the AWS::EC2::InternetGateway resource type to define the InternetGateway,and use the AWS::EC2::VPCGatewayAttachment resource type to associate them with the VPC.
syntax:
```
  igw:
    Type: 'AWS::EC2::InternetGateway'
    Properties: {}

  igwa:
    Type: 'AWS::EC2::VPCGatewayAttachment'
    Properties:
      VpcId: !Ref vpc
      InternetGatewayId: !Ref igw

```
### 3 use the AWS::EC2::Subnet resource type to define the public, private and database subnets in each AZ.
syntax:
```
 PublicSubnetAZ1:
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

  DatabaseSubnetAZ1:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId:
        Ref: vpc
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: us-east-1a
      Tags:
        - Key: stack
          Value: production

  PrivateSubnetAZ1:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId:
        Ref: vpc
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: us-east-1a
      Tags:
        - Key: stack
          Value: production

  PublicSubnetAZ2:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId:
        Ref: vpc
      CidrBlock: 10.0.3.0/24
      MapPublicIpOnLaunch: true
      AvailabilityZone: us-east-1b
      Tags:
        - Key: stack
          Value: production

  PrivateSubnetAZ2:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId:
        Ref: vpc
      CidrBlock: 10.0.4.0/24
      AvailabilityZone: us-east-1b
      Tags:
        - Key: stack
          Value: production

  DatabaseSubnetAZ2:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId:
        Ref: vpc
      CidrBlock: 10.0.5.0/24
      AvailabilityZone: us-east-1b
      Tags:
        - Key: stack
          Value: production
		  
```          
### 4 Use the AWS::EC2::RouteTable resource type to define the Public RouteTable ,and use the AWS::EC2::Route resource type to connect the RouteTable to the InternetGateway.
syntax:
```
  Publicroutetable:
    Type: 'AWS::EC2::RouteTable'
    Properties:
      VpcId:
        Ref: vpc
      Tags:
        - Key: stack
          Value: production

  Publicroute:
    Type: 'AWS::EC2::Route'
    DependsOn: igwa
    Properties:
      RouteTableId:
        Ref: Publicroutetable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId:
        Ref: igw

```
### 5 To associate the two public subnet we created earlier to the RouteTable we  use the AWS::EC2::SubnetRouteTableAssociation resource type to define the Public RouteTable .
syntax:
```
  publicSubnetRouteTableAssociationAZ1:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref PublicSubnetAZ1
      RouteTableId:
        Ref: Publicroutetable
  publicSubnetRouteTableAssociation2:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref PublicSubnetAZ2
      RouteTableId:
        Ref: Publicroutetable
```
### 6 Use AWS::EC2::NatGateway resource type to create your NATGateway
syntax:
```
  NATGateway:
    Type: 'AWS::EC2::NatGateway'
    Properties:
      AllocationId: !GetAtt NATGatewayEIP.AllocationId
      SubnetId: !Ref PublicSubnetAZ1
      Tags:
        - Key: stack
          Value: production
  NATGatewayEIP:
    Type: 'AWS::EC2::EIP'
    Properties:
      Domain: vpc		  
```
### 7 The next thing is to create a private RouteTable, Use AWS::EC2::RouteTable resource type to create your Private RouteTable, and use AWS::EC2::Route to route the traffic to the NATGateway.
syntax:
```
  privateroutetable:
    Type: 'AWS::EC2::RouteTable'
    Properties:
      VpcId:
        Ref: vpc
      Tags:
        - Key: stack
          Value: production
  RouteNATGateway:
    DependsOn: NATGateway
    Type: 'AWS::EC2::Route'
    Properties:
      RouteTableId: !Ref privateroutetable
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NATGateway		  
```
### 8 To associate the two Private subnet we created earlier to the RouteTable we  use the AWS::EC2::SubnetRouteTableAssociation resource type to define the Private RouteTable .
syntax:
```
  privateSubnetRouteTableAssociationAZ1:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref PrivateSubnetAZ1
      RouteTableId:
        Ref: privateroutetable
```
### 9 Repeat step 6,7,8 to create the services in the AZ2(Availability zone)
click [more](https://github.com/AMUTEXKB/Migrating-data-into-RDS-using-Flyway/blob/main/flyway.yml) to see the full code 
### 10 Use the AWS::EC2::SecurityGroup resource type to define the security groups for the public and private subnets,and Use the AWS::EC2::SecurityGroupIngress resource type to specify the inbound traffic rules.
To use flyway to migrate data into the RDS,you need two security group, the ssh security group that would be attached to our bastion host, the database security group that would be attached to our RDS and we would only allow traffic to this port that is coming from  the bastion host.
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
### 11 Before we create our database the first thing we have to specify is the subnet group, the subnet groups allow us to specify the subnet we want to create our database in.Use AWS::RDS::DBSubnetGroup to create your database subnet group
Syntax:
```
	myDBSubnetGroup:
		Properties:
		  DBSubnetGroupDescription: description
		  SubnetIds:
			- !Ref DatabaseSubnet1
			- !Ref DatabaseSubnet2
		  Tags:
			- Key: prod
			  Value: String
		Type: 'AWS::RDS::DBSubnetGroup'
```
### 12 Use the AWS::RDS::DBInstance resource type to create the RDS services.
syntax:
```
  MyDB1:
    Type: 'AWS::RDS::DBInstance'
    Properties:
      DBInstanceIdentifier: !Ref DBInstanceID
      DBName: !Ref DBName
      DBInstanceClass: !Ref DBInstanceClass
      AllocatedStorage: !Ref DBAllocatedStorage
      Engine: MySQL
      EngineVersion: 8.0.28
      MasterUsername: !Ref DBUsername
      MasterUserPassword: !Ref DBPassword
      VPCSecurityGroups:
        - !Ref databasesg
      DBSubnetGroupName: !Ref myDBSubnetGroup
      AvailabilityZone: us-east-1a

```
### 13 Use AWS::EC2::KeyPair to create a key pair that we would use to SSH into our instance
syntax:
```
  NewKeyPair:
    Type: 'AWS::EC2::KeyPair'
    Properties:
      KeyName: MyKeyPair
```	  
### 14 Use AWS::EC2::Instance to launch the bastion host that we would use to SSH into the RDS in the private subnet.
syntax:
```
  bastonhost:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: t2.micro
      ImageId: ami-0b5eea76982371e91
      KeyName:
        Ref: NewKeyPair
      SecurityGroupIds:
        - Ref: bastionhostsg
      SubnetId: !Ref PublicSubnetAZ1
```	  
