# Cloud_Task3

## What the Task is ?
### create a fully secured Web Portal for our company. For this, I have created a setup for Wordpress Software with a dedicated Databse server that will only be accessible by the Wordpress & not by the outside world.
## Project Description 

### 1- Write a Infrastructure as code using terraform, which automatically create a VPC.

### 2- In that VPC we have to create 2 subnets: 
####  a) public subnet [ Accessible for Public World! ]
####  b) private subnet [ Restricted for Public World! ]

### 3- Create a public facing internet gateway for connecting our VPC/Network to the internet world and attach this gateway to our VPC.

### 4- Create a routing table for Internet gateway so that instance can connect to outside world, update and associate it with public subnet.

### 5- Launch an ec2 instance which has Wordpress already setup, having the security group allowing port 80 so that our client can connect to our wordpress site. Also attach the key to instance to facilitate further login.

### 6- Launch an ec2 instance which has MYSQL already setup with security group allowing port 3306 in private subnet so that our wordpress VM can connect with the same. Also attach the key with the same.

## SETPS :
### 1 First of all I create a Virtual Private Cloud (VPC). It Provide a virtual isolated Network. So here we can custmize our Network according our requirement provide by VPC. you can provide any network ip range according to your requirement .
```
resource "aws_vpc" "vishnu_vpc" {
  cidr_block           = "192.168.0.0/16"
  instance_tenancy     = "default"
  enable_dns_hostnames = true
  tags = {
    Name = "vishnu_vpc"
  }
  ```
  ### 2 i created Two subnet one for Public world and other one for Private Network.
  ```
  resource "aws_subnet" "vishnu_public_subnet" {
  vpc_id                  = "${aws_vpc.vishnu_vpc.id}"
  cidr_block              = "192.168.0.0/24"
  availability_zone       = "ap-south-1a"
  map_public_ip_on_launch = "true"
  tags = {
    Name = "vishnu_public_subnet"
  }
}



resource "aws_subnet" "vishnu_private_subnet" {
  vpc_id            = "${aws_vpc.vishnu_vpc.id}"
  cidr_block        = "192.168.1.0/24"
  availability_zone = "ap-south-1a"
  tags = {
    Name = "vishnu_private_subnet"
  }
}
```
  
### 3 I create An internet gateway. It  is a horizontally scaled, redundant, and highly available VPC component that allows communication between instances in your VPC and the internet. It therefore imposes no availability risks or bandwidth constraints on your network traffic.

```
resource "aws_internet_gateway" "vishnu_gw" {
  vpc_id = "${aws_vpc.vishnu_vpc.id}"
  tags = {
    Name = "vishnu_gw"
  }
}
```
### 4 :  I create a Routing Table & associate it with the Public Subnet.  A route table contains a set of rules, called routes, that are used to determine where network traffic is directed. A routing table, or routing information base (RIB), is an electronic file or database-type object that is stored in a router or a networked computer, holding the routes (and in some cases, metrics associated with those routes) to particular network destinations. This information contains the topology of the network close to it.A

```
resource "aws_route_table" "vishnu_rt" {
  vpc_id = "${aws_vpc.vishnu_vpc.id}"

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = "${aws_internet_gateway.vishnu_gw.id}"
  }

  tags = {
    Name = "vishnu_rt"
  }
}


resource "aws_route_table_association" "vishnu_rta" {
  subnet_id      = "${aws_subnet.vishnu_public_subnet.id}"
  route_table_id = "${aws_route_table.vishnu_rt.id}"
}
```
## 5 I create A security group for my wordpress that can we access by public word. A Sicurity group kind of A Virtual Firewall here we specfied the Inbound and outbound rule for our Network.
```
resource "aws_security_group" "vishnu_sg" {

  name   = "vishnu_sg"
  vpc_id = "${aws_vpc.vishnu_vpc.id}"


  ingress {

    description = "allow_http"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]

  }


  ingress {

    description = "allow_ssh"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {

    description = "allow_icmp"
    from_port   = 0
    to_port     = 0
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {

    description = "allow_mysql"
    from_port   = 3306
    to_port     = 3306
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }


  tags = {

    Name = "vishnu_sg"
  }
}
```
## 6 Now, we create another security group which will be used to launch MySQL. This security group will keep the MySQL accessible only through the wordpress and not through outside world 
```
resource "aws_security_group" "vishnu_sg_private" {

  name   = "vishnu_sg_private"
  vpc_id = "${aws_vpc.vishnu_vpc.id}"

  ingress {

    description     = "allow_mysql"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.vishnu_sg.id]

  }


  ingress {

    description     = "allow_icmp"
    from_port       = -1
    to_port         = -1
    protocol        = "icmp"
    security_groups = [aws_security_group.vishnu_sg.id]

  }

  egress {

    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }


  tags = {

    Name = "vishnu_sg_private"
  }
}

resource "aws_instance" "wordpress" {

  ami                         = "ami-ff82f990"
  instance_type               = "t2.micro"
  key_name                    = "test_vpc"
  subnet_id                   = "${aws_subnet.vishnu_public_subnet.id}"
  security_groups             = ["${aws_security_group.vishnu_sg.id}"]
  associate_public_ip_address = true
  availability_zone           = "ap-south-1a"


  tags = {
    Name = "vishnu_wordpress"
  }
}
```
## 7Now , We launch our Wordpress and MySQL instances using all the resources that we have created above.

### WordPress
```
resource "aws_instance" "wordpress" {

  ami                         = "ami-ff82f990"
  instance_type               = "t2.micro"
  key_name                    = "test_vpc"
  subnet_id                   = "${aws_subnet.vishnu_public_subnet.id}"
  security_groups             = ["${aws_security_group.vishnu_sg.id}"]
  associate_public_ip_address = true
  availability_zone           = "ap-south-1a"


  tags = {
    Name = "vishnu_wordpress"
  }
}
```
### MySql
```
resource "aws_instance" "sql" {
  ami               = "ami-08706cb5f68222d09"
  instance_type     = "t2.micro"
  key_name          = "test_vpc"
  subnet_id         = "${aws_subnet.vishnu_private_subnet.id}"
  availability_zone = "ap-south-1a"
  security_groups   = ["${aws_security_group.vishnu_sg_private.id}"]

  tags = {
    Name = "vishnu_sql"
  }
}
```

### 9 Now, I  run my terraform code. For doing so, we first run the command terraform init. This will download the necessary plugins.

### Then, we run the command terraform apply --auto-approve.
![Terraform Apply](TASK/Screenshot from 2020-08-17 13-43-07.png)
