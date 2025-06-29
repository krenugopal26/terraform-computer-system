# terraform-computer-system


provider "aws" {
  region     = "us-east-1"
  access_key = "AKIAZIU6QAHZHU3BW3VI"
  secret_key = "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"
}

# 1. Key Pair
resource "tls_private_key" "windows_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "windows_key_pair" {
  key_name   = "windows_compute_key"
  public_key = tls_private_key.windows_key.public_key_openssh
}

resource "local_file" "private_key_pem" {
  content         = tls_private_key.windows_key.private_key_pem
  filename        = "${path.module}/windows_compute_key.pem"
  file_permission = "0400"
}

# 2. Default VPC and Subnet
data "aws_vpc" "default" {
  default = true
}

data "aws_subnet" "default`" {
  filter {
    name   = "default-for-az"
    values = ["true"]
  }

  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }

  availability_zone = "us-east-1a"  # Replace with another AZ if needed
}

# 3. Security Group (RDP only)
resource "aws_security_group" "windows_sg" {
  name        = "windows_compute_security_group"
  description = "Allow RDP inbound traffic"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    from_port   = 3389
    to_port     = 3389
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # For demo only; restrict this in real-world
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# 4. Latest Windows AMI
data "aws_ami" "windows2025" {
  most_recent = true

  filter {
    name   = "name"
    values = ["Windows_Server-2025-English-Full-Base-*"]
  }

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }

  owners = ["amazon"]
}

# 5. EC2 Instance
resource "aws_instance" "windows_compute" {
  ami                         = data.aws_ami.windows2025.id
  instance_type               = "t3a.medium"
  key_name                    = aws_key_pair.windows_key_pair.key_name
  vpc_security_group_ids      = [aws_security_group.windows_sg.id]
  subnet_id                   = data.aws_subnet.default_1.id
  associate_public_ip_address = true

  root_block_device {
    volume_size           = 30
    volume_type           = "gp2"
    delete_on_termination = false
  }

  tags = {
    Name = "windows_compute"
  }
}

# 6. SNS Topic & Subscription
resource "aws_sns_topic" "alert_topic" {
  name = "windows_compute_alert_topic"
}

resource "aws_sns_topic_subscription" "email_alert" {
  topic_arn = aws_sns_topic.alert_topic.arn
  protocol  = "email"
  endpoint  = "systemadmin@abc.com"  # Replace with your actual email
}

# 7. CloudWatch Alarm
resource "aws_cloudwatch_metric_alarm" "cpu_alarm" {
  alarm_name          = "windows_compute_cpu_utilization"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "Alert: windows_compute EC2 instance CPU utilization exceeded 80%"
  alarm_actions       = [aws_sns_topic.alert_topic.arn]

  dimensions = {
    InstanceId = aws_instance.windows_compute.id
  }
}

