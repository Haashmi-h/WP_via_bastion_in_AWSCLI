# Wordpress installation using bastion server setup - AWS CLI
***
<br />
 
Hi Everyone,
<br />

This is a demonstration of deploying Wordpress website using bastion host setup via AWS-CLI method and some AWS features.<br />
A bastion host is a server that provides secure access to Linux instances located in the private and public subnets of a virtual private cloud (VPC).<br />
It is sometimes called a jump box or jump server. 
In live production environment, this will be done using terraform, bash etc..  <br />
But we are implementing this setup via AWSCLI for learning purpose.


 <br />
This task basically requires following:

- ***IAM user having EC2FullAccess, VPCFullAccess***
- ***Machine with AWSCLI installed.***
- ***A VPC, Internet Gateway and NAT gateway***
- ***Elastic IP address***
- ***SSH key pair and 3 security groups***
- ***3 EC2 instances***
 <br />
  <br />
<br />
I have logged in to AWSCLI using the following command with an IAM user using it's Access Key ID and Secret Access Key as login credentials.

```sh 
[root@ip-172-31-35-96 ~]# aws configure
AWS Access Key ID [None]: XXXXXXXXXXXX
AWS Secret Access Key [None]: XXXXXXXXXXXX
Default region name [None]: us-east-2
Default output format [None]: json
```
 
<br />

#### 1. VPC creation:

A new VPC with IP range 172.16.0.0/16 is created using following command and stored the command output to a file "VPCid.txt", since this VPC id is needed in next steps.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-vpc --cidr-block 172.16.0.0/16 --query Vpc.VpcId --output text > VPCid.txt
[root@ip-172-31-35-96 ~]# cat VPCid.txt
vpc-0cb6d74973d96e7f9
[root@ip-172-31-35-96 ~]#
```

Assigned a tag for this VPC as "webserver-vpc"
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-tags  --resources vpc-0cb6d74973d96e7f9 --tags Key=Name,Value=webserver-vpc
[root@ip-172-31-35-96 ~]#
```
<br />

#### 2. Defining  the necessary SUBNETs:

Here, 3 subnets are created from the IP range assigned on VPC, since the wordpress is setup via bastion server.
2 public subnets are for  the 'instance loading the website contents' and an instance which acts 'bastion server through which website is hosted'
1 private subnet is for an instance acts as database server.

I have created the subnets with VPC id, IPv4 CIDR and availability zone.
**a)**
A subnet with IP range 172.16.0.0/18 in the availability zone "us-east-2a" for one of the public subnets.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-subnet --vpc-id vpc-0cb6d74973d96e7f9 --cidr-block 172.16.0.0/18 --availability-zone us-east-2a
```

Assigned a tag as follows to identify the subnet.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-tags --resources subnet-0cc505b6df3c064bf --tags Key=Name,Value=webserver-public1
```

The following entry is for obtaining the Public IP once this subnet is used to create an EC2 instance.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 modify-subnet-attribute --subnet-id subnet-0cc505b6df3c064bf --map-public-ip-on-launch
```
**b)**
Next subnet with IP range 172.16.64.0/18 in the availability zone "us-east-2b" for another public subnet.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-subnet --vpc-id vpc-0cb6d74973d96e7f9 --cidr-block 172.16.64.0/18 --availability-zone us-east-2b
```
"map-public-ip-on-launch" feature is enabled to get the Public IP once this subnet is used to create an EC2 instance.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 modify-subnet-attribute --subnet-id subnet-0f6d37b857c63cde6 --map-public-ip-on-launch
```
Assigned a tag as follows to identify the subnet.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-tags --resources subnet-0f6d37b857c63cde6 --tags Key=Name,Value=webserver-public2
[root@ip-172-31-35-96 ~]#
```

**c)**
Next subnet with IP range 172.16.128.0/18 in the availability zone "us-east-2c" for the private subnet.
```sh 
root@ip-172-31-35-96 ~]# aws ec2 create-subnet --vpc-id vpc-0cb6d74973d96e7f9 --cidr-block 172.16.128.0/18 --availability-zone us-east-2 c
```
No "map-public-ip-on-launch" is needed since this is a private subnet.
Assigned tag for this subnet too.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-tags --resources subnet-058d23aadcefd67b7 --tags Key=Name,Value=webserver-private
```

Here are the complete subnet details defined from the VPC "vpc-0a6f9f2d328928671" with IP range "172.16.0.0/16".
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 describe-subnets    --filters "Name=vpc-id,Values=vpc-0cb6d74973d96e7f9"
{
    "Subnets": [
        {
            "MapPublicIpOnLaunch": true,
            "AvailabilityZoneId": "use2-az1",
            "Tags": [
                {
                    "Value": "webserver-public1",
                    "Key": "Name"
                }
            ],
            "AvailableIpAddressCount": 16379,
            "DefaultForAz": false,
            "SubnetArn": "arn:aws:ec2:us-east-2:953912081854:subnet/subnet-0cc505b6df3c064bf",
            "Ipv6CidrBlockAssociationSet": [],
            "VpcId": "vpc-0cb6d74973d96e7f9",
            "MapCustomerOwnedIpOnLaunch": false,
            "AvailabilityZone": "us-east-2a",
            "SubnetId": "subnet-0cc505b6df3c064bf",
            "OwnerId": "953912081854",
            "CidrBlock": "172.16.0.0/18",
            "State": "available",
            "AssignIpv6AddressOnCreation": false
        },
        {
            "MapPublicIpOnLaunch": false,
            "AvailabilityZoneId": "use2-az3",
            "Tags": [
                {
                    "Value": "webserver-private",
                    "Key": "Name"
                }
            ],
            "AvailableIpAddressCount": 16379,
            "DefaultForAz": false,
            "SubnetArn": "arn:aws:ec2:us-east-2:953912081854:subnet/subnet-058d23aadcefd67b7",
            "Ipv6CidrBlockAssociationSet": [],
            "VpcId": "vpc-0cb6d74973d96e7f9",
            "MapCustomerOwnedIpOnLaunch": false,
            "AvailabilityZone": "us-east-2c",
            "SubnetId": "subnet-058d23aadcefd67b7",
            "OwnerId": "953912081854",
            "CidrBlock": "172.16.128.0/18",
            "State": "available",
            "AssignIpv6AddressOnCreation": false
        },
        {
            "MapPublicIpOnLaunch": true,
            "AvailabilityZoneId": "use2-az2",
            "Tags": [
                {
                    "Value": "webserver-public2",
                    "Key": "Name"
                }
            ],
            "AvailableIpAddressCount": 16379,
            "DefaultForAz": false,
            "SubnetArn": "arn:aws:ec2:us-east-2:953912081854:subnet/subnet-0f6d37b857c63cde6",
            "Ipv6CidrBlockAssociationSet": [],
            "VpcId": "vpc-0cb6d74973d96e7f9",
            "MapCustomerOwnedIpOnLaunch": false,
            "AvailabilityZone": "us-east-2b",
            "SubnetId": "subnet-0f6d37b857c63cde6",
            "OwnerId": "953912081854",
            "CidrBlock": "172.16.64.0/18",
            "State": "available",
            "AssignIpv6AddressOnCreation": false
        }
    ]
}
[root@ip-172-31-35-96 ~]#
```

<br />

#### 3. INTERNET GATEWAY:

It is created to avail internet-routable traffics to the VPC .
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text
igw-0ad57855dc6c56834
[root@ip-172-31-35-96 ~]#
```
Assigned tag "webserver-igw".
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-tags \
>  --resources igw-0ad57855dc6c56834 --tags Key=Name,Value=webserver-igw
[root@ip-172-31-35-96 ~]#
```
Attached this Internet gateway  to VPC using following command with VPC id and  internet-gateway id.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 attach-internet-gateway --vpc-id vpc-0cb6d74973d96e7f9 --internet-gateway-id igw-0ad57855dc6c56834
[root@ip-172-31-35-96 ~]#
```

<br />

#### 4. NAT GATEWAY:

A NAT gateway is needed here so that instance created in the private subnet can connect to services outside the VPC; we need this to setup private database server here.
Here, Elastic IP address is needed to deploy NAT gateways.

For that, the following command is used to allocate the elastic IP address.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 allocate-address
{
    "Domain": "vpc",
    "PublicIpv4Pool": "amazon",
    "PublicIp": "3.134.121.252",
    "AllocationId": "eipalloc-0d8cdfc03dd527f60",
    "NetworkBorderGroup": "us-east-2"
}
[root@ip-172-31-35-96 ~]#
```
An elastic IP is created on the region "us-east-2".
Allocation ID from the above results along with private subnet ID is needed to create a NAT gateway.

Created NAT
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-nat-gateway    --subnet-id subnet-0cc505b6df3c064bf --allocation-id eipalloc-0d8cdfc03dd527f60
{
    "NatGateway": {
        "NatGatewayAddresses": [
            {
                "AllocationId": "eipalloc-0d8cdfc03dd527f60"
            }
        ],
        "VpcId": "vpc-0cb6d74973d96e7f9",
        "State": "pending",
        "NatGatewayId": "nat-09fa3e6ec0bd80fc8",
        "SubnetId": "subnet-0cc505b6df3c064bf",
        "CreateTime": "2022-12-09T09:06:58.000Z"
    },
    "ClientToken": "b8d7e36f-daa5-4ed3-86a4-4980d798fe1a"
}
[root@ip-172-31-35-96 ~]#
```
Assigned a tag.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-tags  --resources nat-09fa3e6ec0bd80fc8 --tags Key=Name,Value=webserver-nat
[root@ip-172-31-35-96 ~]#
```

<br />

#### 5. ROUTETABLEs.

The route tables are needed to determine where network traffic from our subnet or gateway is directed.

**a)**
There is a route table that automatically comes with every VPC.
It controls the routing for all subnets that are not explicitly associated with any other route table.

We can get the details of the default routetable entries details which is created automatically along with the created VPC using the following entry.

```sh 
[root@ip-172-31-35-96 ~]#aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-0a6f9f2d328928671"
```
Routetable id  is needed for proceeding further with routetables. Check the following entry from the above command-result.
> *"RouteTableId": "rtb-05fcd42cffa693caa",*

Associated public subnets with this routetable.
For that the subnet IDs can be obtained from the following command.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-0cb6d74973d96e7f9" --query "Subnets[*].{ID:SubnetId, CIDR:CidrBlock}"
[
    {
        "CIDR": "172.16.0.0/18",
        "ID": "subnet-0cc505b6df3c064bf"
    },
    {
        "CIDR": "172.16.128.0/18",
        "ID": "subnet-058d23aadcefd67b7"
    },
    {
        "CIDR": "172.16.64.0/18",
        "ID": "subnet-0f6d37b857c63cde6"
    }
]
```

Associated default routetable for the public subnet "webserver-public1".
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 associate-route-table  --subnet-id subnet-0cc505b6df3c064bf --route-table-id rtb-064183d461a240264
{
    "AssociationState": {
        "State": "associated"
    },
    "AssociationId": "rtbassoc-0f21694f13a8d2c48"
}
```
Similarly , associated the "webserver-public2"
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 associate-route-table  --subnet-id subnet-0f6d37b857c63cde6 --route-table-id rtb-064183d461a240264
```

Added tags to the default route table as "webserver-rtb-default".
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-tags --resources rtb-064183d461a240264 --tags Key=Name,Value=webserver-rtb-default
[root@ip-172-31-35-96 ~]#
```
Added a route entry to internet gateway on the default routetable.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-route --route-table-id rtb-064183d461a240264 --destination-cidr-block 0.0.0.0/0 --gateway-id igw -0ad57855dc6c56834
{
    "Return": true
}
[root@ip-172-31-35-96 ~]#
```

Complete details on this default routetable which is associated with the VPC.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 describe-route-tables --route-table-id rtb-064183d461a240264
{
    "RouteTables": [
        {
            "Associations": [
                {
                    "AssociationState": {
                        "State": "associated"
                    },
                    "RouteTableAssociationId": "rtbassoc-0c30d7dedb9c8230c",
                    "Main": true,
                    "RouteTableId": "rtb-064183d461a240264"
                }
            ],
            "RouteTableId": "rtb-064183d461a240264",
            "VpcId": "vpc-0cb6d74973d96e7f9",
            "PropagatingVgws": [],
            "Tags": [
                {
                    "Value": "webserver-rtb-default",
                    "Key": "Name"
                }
            ],
            "Routes": [
                {
                    "GatewayId": "local",
                    "DestinationCidrBlock": "172.16.0.0/16",
                    "State": "active",
                    "Origin": "CreateRouteTable"
                },
                {
                    "GatewayId": "igw-0ad57855dc6c56834",
                    "DestinationCidrBlock": "0.0.0.0/0",
                    "State": "active",
                    "Origin": "CreateRoute"
                }
            ],
            "OwnerId": "953912081854"
        }
    ]
}
[root@ip-172-31-35-96 ~]#
```
Refer to the "Routes" section to see the default route entry and recently added route entry.


**b)**
Created a new route table for private subnet and NATgateway.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-route-table --vpc-id vpc-0cb6d74973d96e7f9 --query RouteTable.RouteTableId --output text
rtb-0fbdb48c7a306befc
[root@ip-172-31-35-96 ~]#
```
Added tags to identify this.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-tags \
>  --resources rtb-0fbdb48c7a306befc --tags Key=Name,Value=webserver-rtb-private
[root@ip-172-31-35-96 ~]#
```
Associated the private subnet "webserver-private" to this newly created private routetable.
```sh 
[root@ip-172-31-35-96 ~]#  aws ec2 associate-route-table  --subnet-id subnet-058d23aadcefd67b7 --route-table-id rtb-0fbdb48c7a306befc
{
    "AssociationState": {
        "State": "associated"
    },
    "AssociationId": "rtbassoc-07601c170379847f0"
}
[root@ip-172-31-35-96 ~]#
```

Added routes to NATgateway "webserver-nat".
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-route --route-table-id rtb-0fbdb48c7a306befc --destination-cidr-block 0.0.0.0/0 --nat-gateway-id  nat-09fa3e6ec0bd80fc8
{
    "Return": true
}
[root@ip-172-31-35-96 ~]#
```
Complete routes information on this "webserver-rtb-private".
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 describe-route-tables --route-table-id rtb-0fbdb48c7a306befc
{
    "RouteTables": [
        {
            "Associations": [],
            "RouteTableId": "rtb-0fbdb48c7a306befc",
            "VpcId": "vpc-0cb6d74973d96e7f9",
            "PropagatingVgws": [],
            "Tags": [
                {
                    "Value": "webserver-rtb-private",
                    "Key": "Name"
                }
            ],
            "Routes": [
                {
                    "GatewayId": "local",
                    "DestinationCidrBlock": "172.16.0.0/16",
                    "State": "active",
                    "Origin": "CreateRouteTable"
                },
                {
                    "Origin": "CreateRoute",
                    "DestinationCidrBlock": "0.0.0.0/0",
                    "NatGatewayId": "nat-09fa3e6ec0bd80fc8",
                    "State": "active"
                }
            ],
            "OwnerId": "953912081854"
        }
    ]
}
[root@ip-172-31-35-96 ~]#
```
<br />

#### 6. SECURITY GROUPS

Since this setup requires 3 instances, 3 separate security groups are needed using "webserver-vpc".
**a)** sg-bastion
This is for the instance which acts as bastion server.
Created and assigned necessary tags.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-security-group --group-name sgbastion --description "Security group for SSH access-Bastion" --vp c-id vpc-0cb6d74973d96e7f9
{
    "GroupId": "sg-01d3942da08063e95"
}
[root@ip-172-31-35-96 ~]#
------------------
[root@ip-172-31-35-96 ~]# aws ec2 create-tags \
>  --resources sg-01d3942da08063e95 --tags Key=Name,Value=sg-bastion
[root@ip-172-31-35-96 ~]#
```
Added inbound rule to accept SSH access from my server's public IP address, from which I have initiated the whole setup.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 authorize-security-group-ingress --group-id sg-01d3942da08063e95 --protocol tcp --port 22 --cidr 18.222.196.47/24
[root@ip-172-31-35-96 ~]#
```

**b)**  sg-frontend
Created this security group for the instance which has website files, and assigned tag.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-security-group --group-name sgfrontend --description "Security group for 80,443 from all and 22
from Bastion" --vpc-id vpc-0cb6d74973d96e7f9
{
    "GroupId": "sg-0ad3a3a130323d491"
}
[root@ip-172-31-35-96 ~]# aws ec2 create-tags  --resources sg-0ad3a3a130323d491 --tags Key=Name,Value=sg-frontend
[root@ip-172-31-35-96 ~]#
```
Added inbound rules as follows:
Allow SSH from "sg-bastion"
Allow webaccess with 443 and 80 from anywhere IPv4
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 authorize-security-group-ingress --group-id sg-0ad3a3a130323d491 --protocol tcp --port 22 --source-group sg-01d3942da08063e95
[root@ip-172-31-35-96 ~]# aws ec2 authorize-security-group-ingress --group-id sg-0ad3a3a130323d491 --protocol tcp --port 80 --cidr 0.0.0. 0/0
[root@ip-172-31-35-96 ~]# aws ec2 authorize-security-group-ingress --group-id sg-0ad3a3a130323d491 --protocol tcp --port 443 --cidr 0.0.0 .0/0
[root@ip-172-31-35-96 ~]#
```
**c)** sg-backend
Created security group for database server and assigned tag
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-security-group --group-name sgbackend --description "Security group for 3306 from frontend and 2 2 from Bastion" --vpc-id vpc-0cb6d74973d96e7f9
{
    "GroupId": "sg-0ffc21bf882a0131d"
}
[root@ip-172-31-35-96 ~]# aws ec2 create-tags  --resources sg-0ffc21bf882a0131d --tags Key=Name,Value=sg-backend
[root@ip-172-31-35-96 ~]#
```

Added inbound rules as follows:
Allow SSH from "sg-bastion"
Allow remote mysql access from "sg-frontend"
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 authorize-security-group-ingress --group-id sg-0ffc21bf882a0131d --protocol tcp --port 22 --source-grou p sg-01d3942da08063e95
[root@ip-172-31-35-96 ~]# aws ec2 authorize-security-group-ingress --group-id sg-0ffc21bf882a0131d --protocol tcp --port 3306 --source-gr oup sg-0ad3a3a130323d491
[root@ip-172-31-35-96 ~]#
```
<br />

#### 7. Keypair.
Ceated a keypair and assigned readonly permission for the user.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-key-pair --key-name newkey.pem --query "KeyMaterial" --output text >  newkey.pem
[root@ip-172-31-35-96 ~]#

[root@ip-172-31-35-96 ~]# chmod 400 newkey.pem
[root@ip-172-31-35-96 ~]#
```
<br />

#### 8. AMI information.
We need AMI ID for creating EC2 instances via AWS-CLI.
Here, instances will be created using same  AmazonLinux AMI on my machine. So, AMI ID from my device can be obtained by executing the curl command on  meta-data url as given below:
```sh 
[root@ip-172-31-35-96 ~]# curl http://169.254.169.254/latest/meta-data/ami-id
ami-0beaa649c482330f7
[root@ip-172-31-35-96 ~]#
```

<br />

#### 9. EC2 instance


**a)** bastion-server: for connecting host server and dtabase server
The bastion hosts provide secure access to Linux instances located in the private and public subnets of your virtual private cloud.
Launched an "t2.micro" AmazonLinux EC2 instace with the security group "sg-bastin" and the subnet "webserver-public1".
Added tags
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 run-instances --image-id ami-0beaa649c482330f7 --count 1 --instance-type t2.micro --key-name newkey.pem
--security-group-ids sg-01d3942da08063e95 --subnet-id subnet-0cc505b6df3c064bf
[root@ip-172-31-35-96 ~]# aws ec2 create-tags --resources i-0d9a9c897fd542ab8 --tags Key=Name,Value=bastions-server
[root@ip-172-31-35-96 ~]#
```

Enabled public DNS hostname with VPC, so that a public DNS hostname entry will be available.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 modify-vpc-attribute --vpc-id vpc-0cb6d74973d96e7f9 --enable-dns-hostnames "{\"Value\":true}"
[root@ip-172-31-35-96 ~]#
```

It can be verified from the following command.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 describe-vpc-attribute --vpc-id vpc-0cb6d74973d96e7f9 --attribute enableDnsHostnames
{
    "VpcId": "vpc-0cb6d74973d96e7f9",
    "EnableDnsHostnames": {
        "Value": true
    }
}
```

**b)** frontend-server: to manage webserver.

Launched an "t2.micro" AmazonLinux EC2 instace with the security group "sg-frontend" and the subnet "webserver-public2".
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 run-instances --image-id ami-0beaa649c482330f7 --count 1 --instance-type t2.micro --key-name newkey.pem
--security-group-ids sg-0ad3a3a130323d491 --subnet-id subnet-0f6d37b857c63cde6                                                         
Added necessary tags.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 create-tags --resources i-0070582cf4d616b2a --tags Key=Name,Value=frontend-server        
[root@ip-172-31-35-96 ~]#     
```
                                                                                                       
**c)**backend-server: to manage database.

Launched an "t2.micro" AmazonLinux EC2 instace with the security group "sg-backend" and the subnet "webserver-private".
Added necessary tags.
```sh 
[root@ip-172-31-35-96 ~]# aws ec2 run-instances --image-id ami-0beaa649c482330f7 --count 1 --instance-type t2.micro --key-name newkey.pem 
--security-group-ids sg-0ffc21bf882a0131d --subnet-id subnet-058d23aadcefd67b7
[root@ip-172-31-35-96 ~]# aws ec2 create-tags --resources i-08f85085c52d8b082 --tags Key=Name,Value=backend-server
```

Please note down the instance IDs from the output of commands given above.
All the instances details can be viewed from following commands with instance ID.

 `[root@ip-172-31-35-96 ~]# aws ec2 describe-instances --instance-ids i-0d9a9c897fd542ab8`

`[root@ip-172-31-35-96 ~]# aws ec2 describe-instances --instance-ids i-0070582cf4d616b2a`

`[root@ip-172-31-35-96 ~]# aws ec2 describe-instances --instance-ids i-08f85085c52d8b082`


<br />

#### 10. Deploying WORDPRESS website through bastion-server setup
We will be connecting both webserver and database server through this server.
**a)**
Accessed bastion server "bastions-server" from the localmachine using the key "newkey.pem".
```sh 
[root@ip-172-31-35-96 ~]# ssh -i newkey.pem ec2-user@3.135.249.234
The authenticity of host '3.135.249.234 (3.135.249.234)' can't be established.
ECDSA key fingerprint is SHA256:WaIRIPyiPagkx0uLWvGm+Ohy2jflkuNX7qvD/JY5lHU.
ECDSA key fingerprint is MD5:1d:c9:4a:f8:93:39:90:6e:07:91:aa:e3:a9:f3:27:19.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '3.135.249.234' (ECDSA) to the list of known hosts.

      __|  __|_  )
      _|  (    /  Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
19 package(s) needed for security, out of 31 available
Run "sudo yum update" to apply all updates.
Updated hostname to easily idenify the actual server.
[ec2-user@ip-172-16-15-121 ~]$ hostname
ip-172-16-15-121.us-east-2.compute.internal
[ec2-user@ip-172-16-15-121 ~]$ sudo hostnamectl set-hostname bastion.us-east-2.compute.internal
[ec2-user@ip-172-16-15-121 ~]$
```
**b)**
Logged in to "frontend-server" from  "bastions-server" using "newkey.pem" .
For that, copied the newkey.pem from local machine to "bastions-server" and updated permissions of key file to 400.
```sh 
[root@bastion ~]# ssh -i newkey.pem ec2-user@172.16.111.219
The authenticity of host '172.16.111.219 (172.16.111.219)' can't be established.
ECDSA key fingerprint is SHA256:f0GwlyqB1k76Vz5W9SUBug7kdWLYdl/arzikTCm0CIw.
ECDSA key fingerprint is MD5:e8:2c:65:aa:15:f7:83:78:10:10:dd:d2:07:36:b5:01.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.16.111.219' (ECDSA) to the list of known hosts.

      __|  __|_  )
      _|  (    /  Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
19 package(s) needed for security, out of 31 available
Run "sudo yum update" to apply all updates.
[ec2-user@ip-172-16-111-219 ~]$
Updated the hostname.
[ec2-user@ip-172-16-111-219 ~]$ sudo hostnamectl set-hostname frontend.us-east-2.compute.internal
[ec2-user@ip-172-16-111-219 ~]$
```

I've installed apache via yum, linux via amazon-linux-extras in "frontend-server".
Then extracted the wordpress website contents from  https://wordpress.org/latest.zip and moved contents to the default documentroot *"/var/www/html".*
Assigned proper ownership and activated the wordpress configuration file "wp-config.php".
The commands used in "frontend-server" for hosting the wordpress website.
```sh 
  4  sudo yum install -y httpd
  5  sudo amazon-linux-extras install php7.4
  6  sudo wget https://wordpress.org/latest.zip
  7  systemctl restart httpd.service
  8  sudo systemctl restart httpd.service
  9  sudo systemctl enable httpd.service
  10  sudo unzip latest.zip
  11  cp -a wordpress/* /var/www/html/
  12  sudo cp -a wordpress/* /var/www/html/
  13  sudo chown -R apache.apache /var/www/html/*
  14  ll /var/www/html/
  15  sudo mv /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
```

**c)**
This server acts as remote database server to the wordpress host.
Logged in to "backend-server" from  "bastions-server" using "newkey.pem" and updated hostname.
```sh 
[ec2-user@bastion ~]$ ssh -i newkey.pem ec2-user@172.16.190.154
The authenticity of host '172.16.190.154 (172.16.190.154)' can't be established.
ECDSA key fingerprint is SHA256:8PMN8CPOQiKdIU9ZNTiA3pVIp+WzVog8j4eeHfAcBAU.
ECDSA key fingerprint is MD5:1c:b3:4c:a5:67:96:86:73:44:08:1a:ea:fe:2f:c9:ce.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.16.190.154' (ECDSA) to the list of known hosts.

      __|  __|_  )
      _|  (    /  Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
[ec2-user@ip-172-16-190-154 ~]$ hostname
ip-172-16-190-154.us-east-2.compute.internal
[ec2-user@ip-172-16-190-154 ~]$ sudo hostnamectl set-hostname backend.us-east-2.compute.internal
[ec2-user@ip-172-16-190-154 ~]$
```
Installed mariadb-server on the server.
Commands used for configuring the mariadb-server.
```sh 
      yum install -y mariadb-server
    5  systemctl restart mariadb.service
    6  systemctl status mariadb.service
    7  systemctl enable mariadb.service
    8  mysql_secure_installation
```

Created a database user "dbuser", database "wp_db" and assigned necessary privileges to access database user from the private IP of instance "frontend-server" via mysqlprompt.                                                                  
```sh                                              
MariaDB [(none)]> create database wp_db;                                             
Query OK, 1 row affected (0.00 sec)                                                 
                                                                                     
MariaDB [(none)]> create user 'dbuser'@'172.16.111.219' identified by 'wp123';       
Query OK, 0 rows affected (0.00 sec)                                                 
                                                                                     
MariaDB [(none)]> grant all privileges on wp_db.* to 'dbuser'@'172.16.111.219';     
Query OK, 0 rows affected (0.00 sec)                                                 
                                                                                     
MariaDB [(none)]> flush privileges;                                                 
Query OK, 0 rows affected (0.00 sec)                                                                                                     
MariaDB [(none)]>                                         
```
                         
Logged in back to the "frontend-server" to update these database name, database username, database password entries and the private IP of "backend-server" as "HOST" entry in "wp-config.php" file. Then restarted webserver to reflect the changes.

Thus, successfully completed the wordpress installation.
It can be done by accessing either public DNS name or public IP address. The public DNS name and public IP address of the "frontend-server" can be found using any of the following commands.
```sh 
From localmachine:
[root@ip-172-31-35-96 ~]# aws ec2 describe-instances --instance-ids i-0070582cf4d616b2a


From "frontend-server" using Instance metadata:
[root@frontend ~]$curl -s http://169.254.169.254/latest/meta-data/public-hostname
[root@frontend ~]$curl http://169.254.169.254/latest/meta-data/public-ipv4
```

The website accessed with Public DNS name:
<img width="899" alt="wp website" src="https://user-images.githubusercontent.com/117455666/208154876-6a5d0787-0b0d-4925-b830-8785ef5ef6f8.png">

<br />
<br />

***Additional info:***

*We can add the public IP address of  "frontend-server" as an A record for any newly created subdomain on live domain or any new domain via Route53. 
Then update "WP_Home" and "siteurl" entries to the domain entry in the wordpress configuration file. 
For example, here I've pointed wordpress.haashdev.tech to the server's public IP, so that the URL part website will be accessed as follows:*
![image](https://user-images.githubusercontent.com/117455666/208155072-13bdbe8d-27be-41d5-920b-551de8aaf6da.png)

<br />
<br />
<br />

***Thank You***

