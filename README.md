# Scandiweb-Senior-Task


# Hello ! 
### SSH RSA-key
- Added scandiweb Key into ***authorized_keys*** in .ssh
### ALB

#### First of all created a self-signed ssl and added it to AWS 
```bash
 aws acm import-certificate  --certificate file://my-aws-public.crt --private-key file://my-aws-private.key --region us-east-1 --profile default
 ```
 - Created the Application Load Balancer with 2 target groups 
1st target group is VARNISH which contains the varnish server 
2nd target group is the Magento-2 EC2 with a rule if the path starts with /media/ or /static/ forward to it 

#### VARNISH

- The varnish EC2 is configured as following 
 ```bash
 sudo vim /etc/varnish/default.vcl
```
```JSON
backend default {

    .host = "3.91.218.20";

    .port = "443";

}
```
then configue varnish to listen on custom port by making configuration file

#### Magento 2 EC2
- Installed nginx, mysql, php-fpm
- Installed Magento 2 using composer 
- Configured Nginx to magento
![Screenshot from 2023-01-17 15-14-46](https://user-images.githubusercontent.com/103090890/212922286-2c524e85-2b37-445e-b557-c8341e1c045e.png)

## URL ALB-1443718690.us-east-1.elb.amazonaws.com
## IP Addresses 

- Varnish : 3.93.234.24


- Magento-2 : 3.91.218.20



## Terraform file example 
```HASHICORP
provider "aws" {
  region = "us-west-2"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "main" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.0.0/24"
}

resource "aws_security_group" "main" {
  name = "main"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port = 80
    to_port = 80
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port = 443
    to_port = 443
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_key_pair" "main" {
  key_name = "main"
  public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDYlPi5I5c5+5S5j9tJ9n1nL"
}

resource "aws_instance" "varnish" {
  ami = "ami-0c84855ba95c71c99"
  instance_type = "t2.micro"
  key_name = aws_key_pair.main.key_name
  vpc_security_group_ids = [aws_security_group.main.id]
  subnet_id = aws_subnet.main.id
}

resource "aws_instance" "magento" {
  ami = "ami-0c84855ba95c71c99"
  instance_type = "t2.micro"
  key_name = aws_key_pair.main.key_name
  vpc_security_group_ids = [aws_security_group.main.id]
  subnet_id = aws_subnet.main.id
}

resource "aws_alb" "main" {
  name = "ALB"
  internal = false
  security_groups = [aws_security_group.main.id]
  subnets = [aws_subnet.main.id]
}
resource "aws_alb_target_group" "varnish" {
name = "varnish"
port = 80
protocol = "HTTP"
vpc_id = aws_vpc.main.id
target_type = "instance"
}

resource "aws_alb_target_group_attachment" "varnish" {
target_group_arn = aws_alb_target_group.varnish.arn
target_id = aws_instance.varnish.id
port = 80
}

resource "aws_alb_listener" "http" {
load_balancer_arn = aws_alb.main.arn
port = "80"
protocol = "HTTP"
default_action {
type = "forward"
target_group_arn = aws_alb_target_group.varnish.arn
}
}

resource "aws_alb_listener" "https" {
load_balancer_arn = aws_alb.main.arn
port = "443"
protocol = "HTTPS"
certificate_arn = "arn:aws:acm:us-west-2:1234567890:certificate/your-certificate-arn"
default_action {
type = "forward"
target_group_arn = aws_alb_target_group.varnish.arn
}
}

resource "aws_alb_target_group" "magento" {
name = "magento"
port = 80
protocol = "HTTP"
vpc_id = aws_vpc.main.id
target_type = "instance"
}

resource "aws_alb_target_group_attachment" "magento" {
target_group_arn = aws_alb_target_group.magento.arn
target_id = aws_instance.magento.id
port = 80
}

resource "aws_route53_record" "alb" {
zone_id = "your_zone_id"
name = "example.com"
type = "A"
alias {
name = aws_alb.main.dns_name
zone_id = aws_alb.main.zone_id
evaluate_target_health = true
}
}
```
  
  

