

# Inventory Management System with Spring Boot and PostgreSQL

This project demonstrates how to deploy a Spring Boot application on a VPS using PostgreSQL as the database, NGINX as a reverse proxy, and Let's Encrypt SSL certificates for HTTPS. This guide walks through the steps to set up, deploy, and secure the application.

---

## Table of Contents

1. [Initial VPS Setup](#1-initial-vps-setup)
    - [Accessing the VPS](#accessing-the-vps)
    - [Basic Security Enhancements](#basic-security-enhancements)
2. [Setting Up PostgreSQL Database](#2-setting-up-postgresql-database)
    - [Creating a Database and User](#creating-a-database-and-user)
3. [Deploying the Spring Boot Application](#3-deploying-the-spring-boot-application)
    - [Building and Deploying the JAR](#building-and-deploying-the-jar)
    - [Running the Application as a Service](#running-the-application-as-a-service)
4. [Configuring NGINX as a Reverse Proxy](#4-configuring-nginx-as-a-reverse-proxy)
    - [Installing and Configuring NGINX](#installing-and-configuring-nginx)
    - [Removing Port 8080 from the URL](#removing-port-8080-from-the-url)
5. [Setting Up Domain and DNS](#5-setting-up-domain-and-dns)
    - [Configuring Domain on OVHCloud](#configuring-domain-on-ovhcloud)
    - [Verifying DNS Propagation](#verifying-dns-propagation)
6. [Configuring SSL with Let’s Encrypt](#6-configuring-ssl-with-lets-encrypt)
    - [Generating and Applying SSL Certificates](#generating-and-applying-ssl-certificates)
    - [Renewing Certificates Automatically](#renewing-certificates-automatically)
7. [Testing and Verifying the Application](#7-testing-and-verifying-the-application)
    - [Using cURL to Test Endpoints](#using-curl-to-test-endpoints)
    - [Validating Data Persistence](#validating-data-persistence)
  


---

## 1. Setting Up a VPS Environment

### Initial VPS Configuration

1. **Login to your VPS using SSH**  
   Once you receive the VPS credentials from OVHCloud, connect to your VPS:

   ```bash
   ssh ubuntu@<your_vps_ip>
   ```

2. **Update and upgrade system packages**:

   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

### Disable Root Login

1. **Disable direct root access for security**:
   
   Open the SSH configuration file for editing:

   ```bash
   sudo nano /etc/ssh/sshd_config
   ```

   Find the line `PermitRootLogin` and set it to `no`:

   ```
   PermitRootLogin no
   ```

   Restart the SSH service:

   ```bash
   sudo systemctl restart ssh
   ```

### SSH Key-based Authentication

1. **Generate SSH keys on your local machine**:

   ```bash
   ssh-keygen -t rsa -b 4096 -C "your_email@example.com" -f ~/.ssh/id_rsa_vps2
   ```

2. **Copy the public key to the VPS**:

   ```bash
   ssh-copy-id -i ~/.ssh/id_rsa_vps2.pub ubuntu@<your_vps_ip>
   ```

3. **Test SSH login without a password**:

   ```bash
   ssh -i ~/.ssh/id_rsa_vps2 ubuntu@<your_vps_ip>
   ```

4. **Disable password-based login**:

   Edit `/etc/ssh/sshd_config`:

   ```
   PasswordAuthentication no
   ```

   Restart SSH:

   ```bash
   sudo systemctl restart ssh
   ```

---

## 2. Installing Java and Spring Boot

### Install OpenJDK 17

1. **Install OpenJDK**:

   ```bash
   sudo apt install openjdk-17-jdk -y
   ```

2. **Verify the installation**:

   ```bash
   java -version
   ```

   You should see output similar to:

   ```
   openjdk version "17.0.12" 2024-07-16
   ```

### Clone and Build the Application

1. **Clone your Spring Boot application from GitHub to your local machine**.

2. **Build the application with Maven**:

   ```bash
   mvn clean install
   ```

3. **Transfer the JAR file to the VPS**:

   ```bash
   scp -i ~/.ssh/id_rsa_vps2 target/inventory-system-0.0.1-SNAPSHOT.jar ubuntu@<your_vps_ip>:/home/ubuntu/
   ```

--- 
Aqui está a segunda parte do documento, continuando de onde paramos:

---

## 3. Setting Up PostgreSQL on the VPS

### Install PostgreSQL

1. **Install PostgreSQL**:

   ```bash
   sudo apt install postgresql postgresql-contrib -y
   ```

2. **Switch to the PostgreSQL user**:

   ```bash
   sudo -i -u postgres
   ```

3. **Create a database and user**:

   ```bash
   psql
   CREATE DATABASE inventory_db;
   CREATE USER inventory_user WITH PASSWORD 'admin';
   GRANT ALL PRIVILEGES ON DATABASE inventory_db TO inventory_user;
   \q
   ```

### Configure Spring Boot for PostgreSQL

1. **Update the `application.properties` file** to connect to PostgreSQL:

   ```properties
   spring.datasource.url=jdbc:postgresql://localhost:5432/inventory_db
   spring.datasource.username=inventory_user
   spring.datasource.password=admin

   spring.jpa.hibernate.ddl-auto=update
   spring.jpa.show-sql=true
   spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
   ```

---

## 4. Running the Spring Boot Application

### Run the Application Manually

1. **Start the Spring Boot application**:

   ```bash
   java -jar /home/ubuntu/inventory-system-0.0.1-SNAPSHOT.jar
   ```

   You can access the API using:

   ```bash
   curl http://<your_vps_ip>:8080/api/products
   ```

### Set Up as a Systemd Service

1. **Create a systemd service** for the Spring Boot application:

   ```bash
   sudo nano /etc/systemd/system/inventory.service
   ```

2. **Add the following content to the service file**:

   ```
   [Unit]
   Description=Inventory System Spring Boot Application
   After=syslog.target
   After=network.target

   [Service]
   User=ubuntu
   ExecStart=/usr/bin/java -jar /home/ubuntu/inventory-system-0.0.1-SNAPSHOT.jar
   SuccessExitStatus=143
   StandardOutput=syslog
   StandardError=syslog
   SyslogIdentifier=inventory

   [Install]
   WantedBy=multi-user.target
   ```

3. **Reload systemd and enable the service**:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable inventory.service
   sudo systemctl start inventory.service
   ```

4. **Check the service status**:

   ```bash
   sudo systemctl status inventory.service
   ```

---

## 5. Configuring NGINX as a Reverse Proxy

### Install NGINX

1. **Install NGINX**:

   ```bash
   sudo apt install nginx -y
   ```

2. **Configure NGINX for the application**:

   Create a new configuration file:

   ```bash
   sudo nano /etc/nginx/sites-available/inventory
   ```

   Add the following content:

   ```
   server {
       listen 80;
       server_name danieloliveira.cloud www.danieloliveira.cloud;

       location / {
           proxy_pass http://localhost:8080;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```

3. **Enable the configuration**:

   ```bash
   sudo ln -s /etc/nginx/sites-available/inventory /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl restart nginx
   ```

---

Aqui está a parte final do documento:

---

## 6. Configuring SSL with Let’s Encrypt

### Install Certbot

1. **Install Certbot for NGINX**:

   ```bash
   sudo apt install certbot python3-certbot-nginx -y
   ```

2. **Obtain SSL Certificates**:

   Run Certbot to generate and install SSL certificates for your domain:

   ```bash
   sudo certbot --nginx -d danieloliveira.cloud -d www.danieloliveira.cloud
   ```

   - When prompted, enter your email address for renewal notifications.
   - Certbot will automatically configure NGINX to use HTTPS.

3. **Verify SSL Certificate Renewal**:

   Certbot automatically sets up renewal, but you can manually test it:

   ```bash
   sudo certbot renew --dry-run
   ```


---

## 7. Testing and Verifying the Application

### Test the API Endpoints

You can test the API using `curl` commands. Below are examples for each endpoint:

- **GET all products**: Retrieve a list of all products.

   ```bash
   curl -X GET https://danieloliveira.cloud/api/products
   ```

- **POST a new product**: Create a new product by sending a JSON payload.

   ```bash
   curl -X POST https://danieloliveira.cloud/api/products \
   -H "Content-Type: application/json" \
   -d '{"name": "Smartphone", "description": "A high-end smartphone", "price": 900, "quantity": 15}'
   ```

- **GET a product by ID**: Retrieve a specific product by its ID.

   ```bash
   curl -X GET https://danieloliveira.cloud/api/products/1
   ```

- **PUT (Update) a product by ID**: Update the details of an existing product by its ID.

   ```bash
   curl -X PUT https://danieloliveira.cloud/api/products/1 \
   -H "Content-Type: application/json" \
   -d '{"name": "Smartphone", "description": "An updated description", "price": 950, "quantity": 20}'
   ```

- **DELETE a product by ID**: Delete a specific product by its ID.

   ```bash
   curl -X DELETE https://danieloliveira.cloud/api/products/1
   ```

### Validating Data Persistence

Ensure that after creating, updating, or deleting a product, the database reflects the changes. You can use PostgreSQL commands or tools like Postman to confirm that your products are being correctly managed.

--- 



