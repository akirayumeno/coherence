---
title: "Package Python scripts to EC2 with Docker"
date: 2025-09-03T17:01:01+09:00
draft: false          
type: "post"         
categories:
  - Memo
  - Daily
tags:
  - Docker
  - Python
  - AWS
author: "coherence"  
aliases:
  - /archives/1
---

## Environment

- Amazon linux 2023  
- Vscode  
- Python 3.11  

## Containerization and Deployment
Actually I need make a tool for testing the vpn network stability recently, so I package the python scripts to EC2 with docker. So we can use CloudWatch Events/Scheduler or AWS Batch to run scripts regularly and so on.

## Build Dockerfile
```bash
# Use Alpine version can minimize the image size
# !!!Note: Although this image contains Python, it is a slim version and lacks many standard Linux utilities, so you should install the tools that you need.
FROM python:3.11-slim-bookworm

# Set the working directory: Create a folder inside the container to store your scripts
WORKDIR /app

# Copy script: Copy the local *.py to the /app directory in the container
# Note: Your script only uses the Python standard library, so a requirements.txt file is not needed.
COPY *.py /app/*.py

# Define the entry point: set the default command to execute when the container starts
# ENTRYPOINT ensures that your Python script is always executed when the container starts
ENTRYPOINT ["python", "/app/*.py"]
```

## Install Docker in EC2
```bash
# Update system packages
sudo yum update -y 

# Install Docker
sudo yum install docker -y

# Start Docker service
sudo service docker start

# Add the current user to the docker group so that you don't need to use sudo every time you run docker commands.
sudo usermod -a -G docker ec2-user 
# Note: If you use the SSH session to connect EC2, after running this command, you need to exit the SSH session and reconnect for it to take effect.

# Check the docker infomation
docker info
```

## Transfer files to EC2
```bash
# Assume your SSH key file is my-key.pem
# Replace with your EC2 IP address and username
scp -i my-key.pem Dockerfile ec2-user@<EC2_IP address>:/home/ec2-user/vpn-test/
scp -i my-key.pem findlog.py ec2-user@<EC2_IP address>:/home/ec2-user/vpn-test/
```

## Build Docker image
```bash
# 1. Switch to the project directory
cd /home/ec2-user/vpn-test/

# 2. Execute Docker build command# -t tag name: vpn-tester:local# . indicates that the Dockerfile is in the current directory
docker build -t vpn-tester:local .
```

## Run Docker image
```bash
docker run vpn-tester:local

```

## Additional words
In modern platform engineering practices, the latest tag is usually reserved for the 'golden' version that has been built and tested through a CI/CD pipeline and pushed to a centralized image repository (such as ECR or Docker Hub).
- ① **vpn-tester:latest (CI/CD / ECR pushed version)**: Indicates that this image has undergone automated testing and is a standard, trusted version deployed across multiple environments (ECS, Fargate, multiple EC2 instances).
- ② **vpn-tester:local (EC2 local build version)**: Indicates that this image was built directly on an EC2 instance. It is not centrally managed in an ECR repository and exists only in the local Docker cache of that EC2 instance. Using the local tag helps prevent confusion with the 'official' latest image.

