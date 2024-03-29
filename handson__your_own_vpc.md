# Creating you own VPC for OpenShift v4

I walk you through how to create your VPC manually to install OCPv4 on AWS dashboard.

## Preparations to create your VPC

![vpc_design](https://github.com/bysnupy/handson/blob/master/images/vpc_design.png)

"install-config.yaml" should include 3 private and 3 subnet IDs as follows. And you should select machineCIDR in your VPC CIDR.
```
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineCIDR: 10.0.0.0/16
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: ap-northeast-1
    userTags:
      user: test
    subnets:
    - subnet-08b0dcab7875eaa71
    - subnet-0371bc505996d580f
    - subnet-09e967834e7e18a87
    - subnet-0f4220dc1dfb8b467
    - subnet-0c86ef8e6ed2e68a5
    - subnet-04fc969510e8b7d1f
```

Type| Subnet Name| CIDR
-|-|-
Private|a-northeast-1a-private-subnet|10.0.0.0/20
Private|a-northeast-1c-private-subnet|10.0.16.0/20
Private|a-northeast-1d-private-subnet|10.0.32.0/20
Public|a-northeast-1a-public-subnet|10.0.128.0/20
Public|a-northeast-1c-public-subnet|10.0.144.0/20
Public|a-northeast-1d-public-subnet|10.0.160.0/20

## Create VPC

I will create VPC CIDR with "10.0.0.0/16" which is matched with "machineCIDR" in install-config.yaml

![vpc1](https://github.com/bysnupy/handson/blob/master/images/vpc1.png)

Enable enableDnsHostnames as follows. enableDnsSupport is enabled by default.

![vpc2](https://github.com/bysnupy/handson/blob/master/images/vpc2.png)

## Create private and public subnets

Creating 3 private subnets as repeating the following operation.

![vpc3](https://github.com/bysnupy/handson/blob/master/images/vpc3.png)

Creating 3 public subnets as repeating the following operation.

![vpc6](https://github.com/bysnupy/handson/blob/master/images/vpc6.png)

## Create each routing table for each private subnet

In this case, the public subnet routing table is used with the VPC default routing tables by default.
You can custom your public subnet routing table according to your use cases. Such as creating each routing table for each public subnet.
Anyway we need to create another routing table for each private tables as reating the following operations.

![vpc12](https://github.com/bysnupy/handson/blob/master/images/vpc12.png)

Change current routing table association to new each one as follows.
We need to configure each NAT Gateway to each routing table on each private subnet.
This is more stable than sharing one NAT Gateway with all private subnet. 

![vpc13](https://github.com/bysnupy/handson/blob/master/images/vpc13.png)

## Create Internet Gateway and attach it to your VPC

![vpc9](https://github.com/bysnupy/handson/blob/master/images/vpc9.png)

Attach the IGW to your VPC.

![vpc10](https://github.com/bysnupy/handson/blob/master/images/vpc10.png)

## Create NAT Gateway

![vpc14](https://github.com/bysnupy/handson/blob/master/images/vpc14.png)

## Add routing rules for each routing table on each public subnet.

Add routing rule for IGW to public subnet routing tables.

![vpc11](https://github.com/bysnupy/handson/blob/master/images/vpc11.png)

Add routing rule for NAT Gateway to each routing table on each private subnet.

![vpc15](https://github.com/bysnupy/handson/blob/master/images/vpc15.png)

Done.
