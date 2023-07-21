# cloudformation-template-devopsacademy
AWSTemplateFormatVersion: '2010-09-09'
Description: Stack per la creazione di una VPC, un gruppo di autoscaling EC2 e un Application Load Balancer (ALB)

Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Nome
          Value: "vera"
        - Key: Cognome
          Value: "ranieri"
        - Key: Environment
          Value: DevOpsAcademy

  MyInternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: "MyInternetGateway"

  MyInternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref MyVPC
      InternetGatewayId: !Ref MyInternetGateway
  
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.0.0/24
      AvailabilityZone: us-east-1a
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: "Public Subnet 1"
        - Key: Environment
          Value: DevOpsAcademy

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: us-east-1b
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: "Public Subnet 2"
        - Key: Environment
          Value: DevOpsAcademy

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC

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

  InternetRoute:
    Type: AWS::EC2::Route
    DependsOn: MyInternetGateway
    Properties:
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref MyInternetGateway
      RouteTableId: !Ref PublicRouteTable

  MySecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group per le istanze EC2
      VpcId: !Ref MyVPC

  MyLaunchConfig:
    Type: AWS::AutoScaling::LaunchConfiguration
    Properties:
      ImageId: ami-05548f9cecf47b442
      InstanceType: t2.micro
      SecurityGroups:
        - !Ref MySecurityGroup
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          yum update -y
          yum install -y httpd php php-mysql
          systemctl enable httpd
          systemctl start httpd
          echo "<?php phpinfo(); ?>" > /var/www/html/index.php
          chkconfig httpd on

  MyAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      LaunchConfigurationName: !Ref MyLaunchConfig
      MinSize: "2"
      MaxSize: "5"
      DesiredCapacity: "2"
      VPCZoneIdentifier:
        - !Ref PublicSubnet1  
        - !Ref PublicSubnet2  

  MyTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: MyTargetGroup
      Port: 80
      Protocol: HTTP
      VpcId: !Ref MyVPC
      TargetType: instance
      Tags:
        - Key: Nome
          Value: "veronica"
        - Key: Cognome
          Value: "ranieri"
        - Key: Environment
          Value: DevOpsAcademy

  MyLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: MyLoadBalancer
      Subnets:
        - !Ref PublicSubnet1  
        - !Ref PublicSubnet2
      SecurityGroups:
        - !Ref MySecurityGroup

  MyListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref MyLoadBalancer
      Protocol: HTTP
      Port: 80
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref MyTargetGroup

Outputs:
  MyAutoScalingGroup:
    Value: !Ref MyAutoScalingGroup
    Description: Auto Scaling Group for the application instances
  MyVPC:
    Value: !Ref MyVPC
    Description: The ID of the VPC
  MyPublicSubnet1:
    Value: !Ref PublicSubnet1
    Description: The ID of the public subnet 1
  MyPublicSubnet2:
    Value: !Ref PublicSubnet2
    Description: The ID of the public subnet 2
  MyLoadBalancerEndpoint:
    Value: !GetAtt MyLoadBalancer.DNSName
    Description: The endpoint URL of the load balancer
