# Interview Questions for DevOps Profile
## Q1.Create EC2 instance on AWS with Terraform
We'll use Terraform to provision an EC2 instance. Here's how to create a basic EC2 instance on AWS.

Terraform Configuration `(main.tf)`:

```
provider "aws" {
  region = "us-west-2"
}

resource "aws_instance" "my_ec2_instance" {
  ami             = "ami-0c55b159cbfafe1f0"  # Update to the latest Amazon Linux AMI ID in your region
  instance_type   = "t2.micro"
  key_name        = "your-key-pair"  # Replace with your own EC2 key pair
  security_groups = ["my-security-group"]

  tags = {
    Name = "MyEC2Instance"
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y aws-cli
              EOF
}

resource "aws_security_group" "my_security_group" {
  name        = "my-security-group"
  description = "Allow SSH, HTTP, and Docker ports"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## Q2. Install Docker and Nginx with Ansible
Once the EC2 instance is up and running, you can use Ansible to install Docker and Nginx.

Ansible Playbook `(install_docker_nginx.yml)`:
```
---
- name: Install Docker and Nginx
  hosts: ec2
  become: true
  tasks:
    - name: Install Docker
      yum:
        name: docker
        state: present
    
    - name: Start Docker service
      service:
        name: docker
        state: started
        enabled: true

    - name: Install Nginx
      yum:
        name: nginx
        state: present

    - name: Start Nginx service
      service:
        name: nginx
        state: started
        enabled: true
```

You would define the host group in the `hosts.ini` file:

hosts.ini:
```
[ec2]
<your-ec2-public-ip>
```
## Q3. Create GitHub Actions pipeline to build and push Docker image to ECR, SSH into the server and pull latest image
Here's an example of a GitHub Actions pipeline `(.github/workflows/deploy.yml)`:

GitHub Actions Pipeline `(deploy.yml)`:

```  
name: Deploy to EC2 with Docker

on:
  push:
    branches:
      - main

jobs:
  build_and_deploy:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v2
    
    - name: Log in to Amazon ECR
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build Docker image
      run: |
        docker build -t my-app .

    - name: Tag Docker image
      run: |
        docker tag my-app:latest <aws_account_id>.dkr.ecr.<region>.amazonaws.com/my-app:latest

    - name: Push Docker image to ECR
      run: |
        docker push <aws_account_id>.dkr.ecr.<region>.amazonaws.com/my-app:latest
    
    - name: SSH into EC2 and pull the latest image
      uses: appleboy/ssh-action@v0.1.0
      with:
        host: ${{ secrets.EC2_HOST }}
        username: ec2-user
        key: ${{ secrets.EC2_PRIVATE_KEY }}
        script: |
          docker pull <aws_account_id>.dkr.ecr.<region>.amazonaws.com/my-app:latest
          docker stop my-app-container || true
          docker rm my-app-container || true
          docker run -d --name my-app-container -p 80:80 <aws_account_id>.dkr.ecr.<region>.amazonaws.com/my-app:latest
```

## Q4. Run the Docker containers on EC2
In the GitHub Actions pipeline, after pulling the Docker image, the container will be started using the following command (as shown above in step 3):

`docker run -d --name my-app-container -p 80:80 <aws_account_id>.dkr.ecr.<region>.amazonaws.com/my-app:latest`

## Q5. Configure appropriate Docker container ports
When running the Docker container, we expose port 80 for HTTP. Ensure that your EC2 security group allows inbound traffic on port 80.

```
ingress {
  from_port   = 80
  to_port     = 80
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
```

## Q6. Set up Nginx as a reverse proxy
You'll need to configure Nginx to forward incoming requests to the Docker containers. Edit the Nginx configuration file to set it up as a reverse proxy.

Nginx Configuration `(/etc/nginx/nginx.conf)`:

```
server {
    listen 80;
    
    server_name your-domain.com;
    
    location / {
        proxy_pass http://localhost:8080;  # Change the port to match your Docker container
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
**Make sure to restart Nginx after updating the configuration:**

`sudo systemctl restart nginx`

## Q7. Enable CloudWatch logs on AWS for monitoring the EC2 instance
You can enable CloudWatch monitoring for your EC2 instance by installing the CloudWatch Agent.

**1.Step-by-Step:**

Install the CloudWatch Agent on EC2:

`sudo yum install -y amazon-cloudwatch-agent`

**2.Create the CloudWatch configuration file `(/opt/aws/amazon-cloudwatch-agent/bin/config.json)`:**
```
{
  "logs": {
    "metrics_collected": {
      "cpu": {
        "measurement": ["cpu_usage_idle"],
        "metrics_collection_interval": 60
      }
    }
  }
}
```
**3.Start the CloudWatch Agent:**
```
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a start \
    -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json
```

This will allow you to monitor metrics from your EC2 instance in the AWS CloudWatch console.
