# aws-vpc-autoscaling-web-architecture-withCFT
"An automated, high-availability multi-tier web application infrastructure on AWS deployed using CloudFormation (IaC), featuring custom VPC, Application Load Balancer, and EC2 Auto Scaling."

AWSTemplateFormatVersion: '2010-09-09'
Description: High Availability Multi-Tier Web Infrastructure using VPC, ALB, and Auto Scaling

Parameters:
  VpcCIDR:
    Type: String
    Default: 10.0.0.0/16
    Description: CIDR block for the VPC
  
  PublicSubnet1CIDR:
    Type: String
    Default: 10.0.1.0/24
    Description: CIDR block for Public Subnet 1
  
  PublicSubnet2CIDR:
    Type: String
    Default: 10.0.2.0/24
    Description: CIDR block for Public Subnet 2

  PrivateSubnet1CIDR:
    Type: String
    Default: 10.0.3.0/24
    Description: CIDR block for Private Subnet 1

  PrivateSubnet2CIDR:
    Type: String
    Default: 10.0.4.0/24
    Description: CIDR block for Private Subnet 2

Resources:
  # 1. Custom VPC
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: AWS-Demo-H-VPC

  # 2. Internet Gateway
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: AWS-Demo-IGW

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref MyVPC
      InternetGatewayId: !Ref InternetGateway

  # 3. Subnets
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: !Ref PublicSubnet1CIDR
      AvailabilityZone: !Select [ 0, !GetAZs '' ]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: Public-Subnet-1

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: !Ref PublicSubnet2CIDR
      AvailabilityZone: !Select [ 1, !GetAZs '' ]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: Public-Subnet-2

  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: !Ref PrivateSubnet1CIDR
      AvailabilityZone: !Select [ 0, !GetAZs '' ]
      Tags:
        - Key: Name
          Value: Private-Subnet-1

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: !Ref PrivateSubnet2CIDR
      AvailabilityZone: !Select [ 1, !GetAZs '' ]
      Tags:
        - Key: Name
          Value: Private-Subnet-2

  # 4. Route Table & Internet Route for Public Subnets
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC
      Tags:
        - Key: Name
          Value: Public-Route-Table

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable

  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRouteTable

  # 5. Security Groups
  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP traffic to ALB
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow traffic from ALB and SSH
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref ALBSecurityGroup
        - IpProtocol: tcp
          FromPort: 8000
          ToPort: 8000
          SourceSecurityGroupId: !Ref ALBSecurityGroup

  # 6. Application Load Balancer
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: App-Load-Balancer
      Subnets:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
      SecurityGroups:
        - !Ref ALBSecurityGroup
      Scheme: internet-facing

  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: Web-Target-Group
      Port: 80
      Protocol: HTTP
      VpcId: !Ref MyVPC
      TargetType: instance

  ALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Port: 80
      Protocol: HTTP

  # 7. Launch Template & Auto Scaling Group
  WebServerLaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: WebServerLaunchTemplate
      LaunchTemplateData:
        ImageId: ami-0c7217cdde317cfec # Note: Region-specific Amazon Linux 2 AMI
        InstanceType: t3.micro
        SecurityGroupIds:
          - !Ref WebServerSecurityGroup
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash
            yum update -y
            yum install -y httpd
            systemctl start httpd
            systemctl enable httpd
            echo "<h1>Hello from AWS Auto Scaling Instance!</h1>" > /var/www/html/index.html

  WebServerASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      VpcZoneIdentifier:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
      LaunchTemplate:
        LaunchTemplateId: !Ref WebServerLaunchTemplate
        Version: !GetAtt WebServerLaunchTemplate.LatestVersionNumber
      MinSize: '1'
      MaxSize: '3'
      DesiredCapacity: '2'
      TargetGroupARNs:
        - !Ref TargetGroup

Outputs:
  ALBURL:
    Description: URL of the Application Load Balancer
    Value: !GetAtt ApplicationLoadBalancer.DNSName
