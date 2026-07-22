# Devops Project Report: Automated CI/CD Pipeline for a 2-Tier Flask Application on AWS

**Author:** Dhruv Bhatia

---

## **Table of Contents**
1. [Project Overview](#1-project-overview)
2. [Architecture Diagram](#2-architecture-diagram)
3. [Step 1: AWS EC2 Instance Preparation](#3-step-1-aws-ec2-instance-preparation)
4. [Step 2: Install Dependencies on EC2](#4-step-2-install-dependencies-on-ec2)
5. [Step 3: Jenkins Installation and Setup](#5-step-3-jenkins-installation-and-setup)
6. [Step 4: GitHub Repository Configuration](#6-step-4-github-repository-configuration)
    * [Dockerfile](#Docker-file)
    * [Docker-compose.yaml](#Docker-compose.yaml)
    * [Jenkinsfile](#Jenkinsfile)
7. [Step 5: Jenkins Pipeline Creation and Execution](#7-Step-5-Jenkins-Pipeline-Creation-and-Execution)
8. [Conclusion](#8-Conclusion)
9. [Infrastructure Diagram](#9-Infrastructure-Diagram)
10. [Work Flow Diagram](#10-Work-Flow-Diagram)

### **1. Project Overview**
This document outlines the step-by-step process for deploying a 2-tier web application (Flask + MySQL) on an AWS EC2 instance. The deployment is containerized using Docker and Docker Compose. A full CI/CD pipeline is established using Jenkins to automate the build and deployment process whenever new code is pushed to a GitHub repository.

### **2. Architecture Diagram**

![Architecture Diagram](architecture-diagram.png)

---

### **3. Step 1: AWS EC2 Instance Preparation**

1.  **Launch EC2 Instance:**
    * Navigate to the AWS EC2 console.
    * Launch a new instance using the **Ubuntu 22.04 LTS** AMI.
    * Select the **t2.micro** instance type for free-tier eligibility.
    * Create and assign a new key pair for SSH access.

    ![EC2 Instance Launch](ec2-instance-launch.png)

2. **Configure Security Group:**
     * **Create a security group with the following inbound rules:**
        * **Type:** SSH, **Protocol:** TCP, **Port:** 22, **Source:** Your IP
        * **Type:** HTTP, **Protocol:** TCP, **Port:** 80, **Source:** Anywhere (0.0.0.0/0)
        * **Type:** Custom TCP, **Protocol:** TCP, **Port:** 5000 (for Flask), **Source:** Anywhere (0.0.0.0/0)
        * **Type:** Custom TCP, **Protocol:** TCP, **Port:** 8080 (for Jenkins), **Source:** Anywhere (0.0.0.0/0)
    
    ![Security Groups](Security-Groups.png)

3.  **Connect to EC2 Instance:**
    * Use SSH to connect to the instance's public IP address.
    ```bash
    ssh -i /path/to/key.pem ubuntu@<ec2-public-ip>
    ```

    ---

### **4. Step 2: Install Dependencies on EC2**

1.  **Update System Packages:**
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```

2.  **Install Git, Docker, and Docker Compose:**
    ```bash
    sudo apt install git docker.io docker-compose-v2 -y
    ```

3.  **Start and Enable Docker:**
    ```bash
    sudo systemctl start docker
    sudo systemctl enable docker
    ```

4.  **Add User to Docker Group (to run docker without sudo):**
    ```bash
    sudo usermod -aG docker $USER
    newgrp docker
    ```

    ---

### **5. Step 3: Jenkins Installation and Setup**

1. **Install Java:**
   ```bash
   sudo apt update
   sudo apt install fontconfig openjdk-21-jdk -y
   ```

2. **Add Jenkins Repository and Install:**
   ```bash
   sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
   echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
   ```

3. **Start and Enable Jenkins:**
   ```bash
   sudo apt install jenkins -y
   sudo systemctl enable --now jenkins
   ```
4. **Initial Jenkins Setup:**
   * Retrieve the initial admin password:
        ```bash
        sudo cat /var/lib/jenkins/secrets/initialAdminPassword
        ```
    * Access the Jenkins dashboard at `http://<ec2-public-ip>:8080`.
    * Paste the password, install suggested plugins, and create an admin user.

5. **Grant Jenkins Docker Permissions:**
   ```bash
    sudo usermod -aG docker jenkins
    sudo systemctl restart jenkins
    ```

    ![Admin User](admin-user.png)    

### **6. Step 4: GitHub Repository Configuration**

Ensure your GitHub repository contains the following three files.

### **Dockerfile**
This file defines the environment for the Flask application container.
```dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN apt-get update && apt-get install -y gcc default-libmysqlclient-dev pkg-config && \
rm -rf /var/lib/apt/lists/*

COPY requirement.txt .

RUN pip install --no-cache-dir -r requirement.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

### **Docker-compose.yaml**
This file defines and orchestrates the multi-container application (Flask and MySQL).
```yaml
version: "3.8"

services:
  mysql:
    container_name: mysql
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: "root"
      MYSQL_DATABASE: "devops"
    ports:
    - "3306:3306"

    volumes:
    - mysql_data:/var/lib/mysql 
    
    networks:
    - two-tier-net 

    restart: always
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-proot"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 90s

  flask-app:
    container_name: flask-app
    build:
      context: .

    ports:
    - "5005:5000"

    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: root
      MYSQL_DB: devops
    
    networks:
    - two-tier-net

    depends_on:
      mysql:
        condition: service_healthy

    #healthcheck:
    #  test: ["CMD-SHELL", "curl -f http://localhost:5000/health || exit 1"]
     # interval: 10s
      #timeout: 5s
      #retries: 5
      #start_period: 60s

volumes:
  mysql_data:

networks:
  two-tier-net:
  ```

### **Jenkinsfile**
This file contains the pipeline-as-code definition for Jenkins.
```groovy
pipeline {
    agent any 
    stages {
        stage('Clone repo') {
            steps {
                git branch: 'main', url: 'https://github.com/dhruvbhatia-dev/my-own-project.git'               
            }
        }
        stage('Build image') {
            steps {
                sh 'docker build -t flask-app .'
            }
        }
        stage('Deploy with docker compose') {
            steps {
                sh 'docker compose down'

                sh 'docker compose up -d  --build'
            }
        }
    }
}
```

---

### **7.Step 5: Jenkins Pipeline Creation and Execution**

1.  **Create a New Pipeline Job in Jenkins:**
    * From the Jenkins dashboard, select **New Item**.
    * Name the project, choose **Pipeline**, and click **OK**.

2.  **Configure the Pipeline:**
    * In the project configuration, scroll to the **Pipeline** section.
    * Set **Definition** to **Pipeline script from SCM**.
    * Choose **Git** as the SCM.
    * Enter your GitHub repository URL.
    * Verify the **Script Path** is `Jenkinsfile`.

![Jenkins Pipeline](jenkins-pipeline.png)

3. **Configure Github Webhook(Automated Triggers):**
   * Install the Github Integration plugin in Jenkins.
   * Go to your Github repository -> Settings-> Webhooks-> Click Add Webhook.
   * Payload URL: http://<your-ec2-public-ip>:8080/github-webhook/
   * Content type: application/json
   *Select Just the push event and save.
   * In your Jenkins job configuration, check GitHub hook trigger for GITScm polling.

4.  **Run the Pipeline:**
    * Click **Build Now** to trigger the pipeline manually for the first time.
    * Monitor the execution through the **Stage View** or **Console Output**.

![Jenkins Pipeline](pipeline-run.png)

5. **Verify Deployment:**
    * After a successful build, your Flask application will be accessible at `http://<your-ec2-public-ip>:5000`.
    * Confirm the containers are running on the EC2 instance with `docker ps`.

### **8. Conclusion**
The CI/CD pipeline is now fully operational and automated. Any git push to the main branch instantly triggers Jenkins via GitHub Webhooks, building the new Docker image and deploying the updated 2-tier application seamlessly from development to production.

### **9. Infrastructure Diagram**

![Infrastructure Diagram](Infrastructure-Diagram.png)









