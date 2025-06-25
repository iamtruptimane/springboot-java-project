
# HomesteaderIndia – 3-Tier Deployment on AWS (Spring Boot + JSP + PostgreSQL)

## Project Introduction

HomesteaderIndia is a web-based platform that simplifies property listing and discovery for users looking to buy or rent homes. The application is developed using Spring Boot (for backend logic), JSP (for the user interface), and PostgreSQL (as the database). 

This deployment follows a 3-tier architecture on AWS where:
- The **frontend** (JSP + Tomcat) handles user interaction,
- The **backend** (Spring Boot) manages business logic,
- The **database** (PostgreSQL) stores application data.

The architecture ensures separation of concerns, improved security, and easier scalability for real-time production environments.




## 1. Architecture Overview
![](./img/WhatsApp%20Image%202025-06-25%20at%209.32.11%20PM.jpeg)

| **Tier** | **Purpose**         | **Subnet Type**  | **Component**      | **EC2 Role**              |
|---------|---------------------|------------------|--------------------|---------------------------|
| Tier 1  | Frontend (UI)       | Public Subnet    | Tomcat + JSP App   | Handles browser requests  |
| Tier 2  | Application (Logic) | Private Subnet   | Spring Boot        | Handles business logic    |
| Tier 3  | Database (Storage)  | Private Subnet   | PostgreSQL DB      | Stores persistent data    |

---

##  2. VPC & Subnet Setup

### 2.1 Create a VPC

- **Name:** `SpringBoot-VPC`
- **CIDR block:** `10.0.0.0/16`
- **DNS Hostnames:** Enabled 

![](./img/Screenshot%20(788).png)

### 2.2 Create Subnets

| Subnet Name    | CIDR Block    | AZ           | Type    |
|----------------|---------------|--------------|---------|
| Public-Subnet  | 10.0.1.0/24   | ap-south-1a  | Public  |
| Private-App    | 10.0.2.0/24   | ap-south-1a  | Private |
| Private-DB     | 10.0.3.0/24   | ap-south-1a  | Private |

> Enable Auto-Assign Public IP for **Public Subnet**.

![](./img/Screenshot%20(789).png)

### 2.3 Create Internet Gateway

- **Name:** `SpringBoot-IGW`
- **Attach** to `SpringBoot-VPC`

### 2.4 Create Route Tables
![](./img/Screenshot%20(790).png)

#### a. Public Route Table

- **Name:** `Public-RT`
- **Associate with:** `Public-Subnet`
- **Add Route:**  
  `0.0.0.0/0` ➜ Internet Gateway

![](./img/Screenshot%20(791).png)

#### b. Private Route Table

- **Name:** `Private-RT`
- **Associate with:** `Private-App`, `Private-DB`
- No external route initially.

![](./img/Screenshot%20(792).png)

##  3. NAT Gateway Setup

### Why NAT?

Private subnets need NAT to access the internet for updates/installations.

### Steps

1. **Allocate Elastic IP**  
2. **Create NAT Gateway**
   - Subnet: `Public-Subnet`
   - Elastic IP: Use the one created above
   - Name: `SpringBoot-NATGW`
3. **Update Private Route Table**
   - Add Route:
     - `0.0.0.0/0` ➜ NAT Gateway

---

## 4. Security Groups

| SG Name     | Attached To  | Inbound Rules                              | Outbound     |
|-------------|--------------|---------------------------------------------|--------------|
| Tomcat-SG   | Frontend EC2 | 22 (My IP), 8080 (Anywhere)                 | All traffic  |
| App-SG      | Backend EC2  | 22 (My IP), 8081 (From Tomcat-SG)           | All traffic  |
| DB-SG       | DB EC2       | 5432 (From App-SG), 22 (From Bastion if needed) | All traffic  |


![](./img/Screenshot%20(794).png)

## 5. Launch EC2 Instances

| Role      | AMI           | Subnet         | Type     | Ports      | Key Pair     |
|-----------|---------------|----------------|----------|------------|--------------|
| Frontend  | Ubuntu 22.04  | Public-Subnet  | t2.micro | 22, 8080   | trupti_key   |
| Backend   | Ubuntu 22.04  | Private-App    | t2.micro | 8081       | trupti_key   |
| Database  | Ubuntu 22.04  | Private-DB     | t2.micro | 5432       | trupti_key   |

![](./img/Screenshot%20(793).png)

## 6. Copy Private Key to Frontend Server

### Step 1: Copy Key

```bash
scp -i trupti-key.pem trupti-key.pem ubuntu@<frontend-public-ip>:/home/ubuntu/
````

### Step 2: Set Permissions

```bash
ssh -i trupti-key.pem ubuntu@<frontend-public-ip>
chmod 400 trupti-key.pem
```

---

## 7. DB Server Configuration (Private-DB)

### Install PostgreSQL

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
```

### Create DB & User

```bash
sudo -u postgres psql
CREATE DATABASE homestadercravita;
CREATE USER postgres WITH PASSWORD 'Grp4545@@';
ALTER ROLE postgres WITH SUPERUSER;
\q
```

### Configure PostgreSQL

* **File:** `/etc/postgresql/16/main/postgresql.conf`

```bash
listen_addresses = '*'
```

* **File:** `/etc/postgresql/16/main/pg_hba.conf`

```
host    all             all             10.0.0.0/16           md5
```

### Restart Service

```bash
sudo systemctl restart postgresql
```

---

## 8. Backend Setup (Private-App)

### SSH to Backend

```bash
ssh -i trupti-key.pem ubuntu@<frontend-public-ip>
ssh -i trupti-key.pem ubuntu@<backend-private-ip>
```

### Install Dependencies

```bash
sudo apt update
sudo apt install git maven openjdk-17-jdk -y
```

### Clone & Build Project

```bash
git clone https://github.com/iamtruptimane/springboot-java-project.git
cd springboot-java-project
MAVEN_OPTS="-Xmx2048m" mvn clean package
```

### Run App

```bash
nohup java -jar target/CravitaProject_HomeSteader-0.0.1-SNAPSHOT.jar > /tmp/app.log 2>&1 &
```

### Verify

```bash
curl http://localhost:8081/
```

---

## 9. Frontend Setup with Nginx

### Install Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

### Replace Default Config

```bash
sudo mv /etc/nginx/sites-available/default /etc/nginx/sites-available/default.bak
sudo nano /etc/nginx/sites-available/default
```

### Paste Below Config

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://<backend-private-ip>:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Restart Nginx

```bash
sudo systemctl restart nginx
```

---

## Access the App

Visit:

**http\://frontend-public-ip**

![](./img/Screenshot%20(795).png)
![](./img/Screenshot%20(796).png)
![](./img/Screenshot%20(797).png)

## Project Summary

**HomesteaderIndia** is a monolithic Java application built using Spring Boot, JSP (for frontend), and PostgreSQL (for data storage). This project demonstrates how to deploy the application using a secure and scalable **3-tier architecture on AWS** with proper VPC design, subnetting, NAT, and security group configurations. Each component is hosted on a dedicated EC2 instance across isolated subnets to follow industry-grade deployment practices.

