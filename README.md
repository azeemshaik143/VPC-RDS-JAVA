Deploy Java Web Application on AWS Using CloudFormation
1. Introduction
This documentation provides a step-by-step guide to deploying a Java-based web application in AWS using CloudFormation. The architecture includes a Virtual Private Cloud (VPC) with public and private subnets, an EC2 instance for running the Java web application, a private EC2 instance for database management, and an RDS MySQL instance for storing application data.

Architecture Overview:
VPC: Contains public and private subnets.

EC2 Instances:

Public EC2: Web application server, which is publicly accessible.

Private EC2: Database management instance, which is privately accessible.

RDS MySQL: Database hosted within the private subnet, accessed by the EC2 instances.

NAT Gateway: Provides internet access to private instances.

2. CloudFormation Stack for VPC Setup
2.1. Create Stack Using CloudFormation
Save the YAML script as vpc.yaml and upload it to AWS CloudFormation. This template creates the following resources:

A VPC (demo-vpc).

Two subnets (public and private).

An internet gateway for public access.

A NAT Gateway for private subnet internet access.

EC2 security groups and route tables.

AWSTemplateFormatVersion: 2010-09-09
Description: VPC template
Resources:
  myVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: 'true'
      EnableDnsHostnames: 'true'
      Tags:
       - Key: Name
         Value: Demo-VPC

  myInternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
      - Key: Name
        Value: Demo-IGW

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId:
        Ref: myVPC
      InternetGatewayId:
        Ref: myInternetGateway

  myPubSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref myVPC
      CidrBlock: 10.0.0.0/20
      AvailabilityZone: "us-east-1a"
      Tags:
      - Key: Name
        Value: Demo-app-sub

  myPvtSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref myVPC
      CidrBlock: 10.0.16.0/20
      AvailabilityZone: "us-east-1b"
      Tags:
      - Key: Name
        Value: Demo-db-sub

  myPUBRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId:
        Ref: myVPC
      Tags:
      - Key: Name
        Value: Demo-app-rt

  myPvtRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId:
        Ref: myVPC
      Tags:
      - Key: Name
        Value: Demo-db-rt

  myPUBSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId:
        Ref: myPubSubnet
      RouteTableId:
        Ref: myPUBRouteTable

  myPvtSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId:
        Ref: myPvtSubnet
      RouteTableId:
        Ref: myPvtRouteTable

  MyIGWRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref myPUBRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref myInternetGateway

2.2. Confirm Resource Creation
After creating the CloudFormation stack, verify the following:

VPC (Demo-VPC)

Subnets: Demo-app-sub (public), Demo-db-sub (private)

Internet Gateway: Demo-IGW (attached to VPC)

Route Tables: Demo-app-rt (public), Demo-db-rt (private)

3. NAT Gateway Setup
Allocate an Elastic IP for the NAT Gateway.

Create the NAT Gateway in the public subnet (Demo-app-sub).

Attach the NAT Gateway to the private route table (Demo-db-rt) to allow the private subnet to access the internet.

4. EC2 Instances Setup
4.1. Public EC2 Instance (Web Server)
Launch an EC2 instance (e.g., Ubuntu) in the public subnet (Demo-app-sub).

Open ports: 22 (SSH), 3306 (MySQL), 80 (HTTP) in the public EC2 security group.

This instance will host the Java web application.

4.2. Private EC2 Instance (DB server)
Launch a second EC2 instance in the private subnet (Demo-db-sub).

Use a private security group allowing SSH access only from the public EC2 instance.

This instance will be used to manage the database.

4.3. Connect to Public EC2 Instance
To connect to the public EC2 instance:

ssh -i "your-key.pem" ubuntu@<Public_EC2_IP>
4.4. Jump Host Configuration
To connect to the private EC2 instance, use the public EC2 instance as a jump host:

Copy .pem file to the public EC2 instance:

scp -i "your-key.pem" your-key.pem ubuntu@<Public_EC2_IP>:/home/ubuntu/
SSH into the public EC2 and connect to the private EC2:


ssh -i "your-key.pem" ubuntu@<Private_EC2_IP>
5. Deploy Java Web Application Using Maven and Tomcat
5.1. Install Dependencies on Public EC2 Instance
On the public EC2 instance, install Java, Maven, Git, and Tomcat:

sudo apt update -y
sudo apt install openjdk-17-jre -y
sudo apt install maven -y
sudo apt install git -y
sudo apt install unzip

5.2. Clone and Build the Java Application
Clone the Java project from GitHub:
git clone https://github.com/Ai-TechNov/aws-rds-java.git
cd aws-rds-java
Modify the login.jsp and userRegistration.jsp files to update the RDS endpoint.

Update pom.xml for Java 17 and MySQL 8:

<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
</properties>
<dependencies>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.33</version>
    </dependency>
</dependencies>
5.3. Build and Deploy
Build the project:

mvn clean package
Copy the .war file to Tomcat’s webapps directory:

cp target/*.war /opt/tomcat/webapps/
Start Tomcat:
/opt/tomcat/bin/startup.sh
5.4. Access the Web Application
The web application should now be available at:

http://<Public_EC2_IP>:8080/LoginWebApp/
6. MySQL Setup on Private EC2 Instance
6.1. Connect to MySQL RDS
SSH into the private EC2 instance:


ssh -i "your-key.pem" ec2-user@<Private_EC2_IP>
Install MySQL client:

sudo yum install mysql -y
Connect to the RDS MySQL instance using its endpoint:

mysql -h <RDS_Endpoint> -u admin -p
Enter the MySQL root password when prompted.

6.2. Create Database and Schema
Create the jwt database:
CREATE DATABASE jwt;
USE jwt;
Create the USER table:

CREATE TABLE USER (
    id INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
    first_name VARCHAR(45) NOT NULL,
    last_name VARCHAR(45) NOT NULL,
    email VARCHAR(45) NOT NULL,
    username VARCHAR(45) NOT NULL,
    password VARCHAR(45) NOT NULL,
    regdate DATE NOT NULL,
    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

7. Final Testing
Ensure that the Java web application is accessible at and the all the data will be stored in database server:

http://<Public_EC2_IP>:8080/LoginWebApp

<img width="1366" height="768" alt="Image" src="https://github.com/user-attachments/assets/7f798f54-4b4c-4a41-b3a2-5c10b0ba620b" />

<img width="1366" height="768" alt="Image" src="https://github.com/user-attachments/assets/13bececb-283a-4dfe-8a93-a397b1bb3d63" />


