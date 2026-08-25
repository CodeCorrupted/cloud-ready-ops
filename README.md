# ☁️ Final Project - Cloud Infrastructure on GCP & Full Stack Deployment

This repository contains detailed documentation and configuration scripts for provisioning a 
decoupled VPC network infrastructure on **Google Cloud Platform (GCP)**, along with the deployment and 
troubleshooting of a Full Stack Java web application (Spring Boot + JSF) connected to MariaDB.

---
## 📋 Table of Contents

* [Network Architecture & Cloud Infrastructure](#-network-architecture--cloud-infrastructure)
  * [Subnet Design](#subnet-design)
* [Tech Stack & Infrastructure](#-tech-stack--infrastructure)
* [Infrastructure Setup & Configuration](#-infrastructure-setup--configuration)
  * [1. VPC and Subnet Creation](#1-vpc-and-subnet-creation)
  * [2. Cloud NAT Configuration for Private Subnet](#2-cloud-nat-configuration-for-private-subnet)
  * [3. Firewall Rules (Security Groups)](#3-firewall-rules-security-groups)
  * [4. VM Instance Provisioning](#4-vm-instance-provisioning)
* [Service Provisioning](#-service-provisioning)
  * [Web Server (Nginx - Initial Test)](#web-server-nginx---initial-test)
  * [Database Server (MariaDB)](#database-server-mariadb)
* [Java Application Deployment (Zona Fit CRUD)](#-java-application-deployment-zona-fit-crud)
  * [1. Java JRE & Application Files Setup](#1-java-jre--application-files-setup)
  * [2. Environment Variables & Systemd Configuration](#2-environment-variables--systemd-configuration)
  * [3. Nginx Reverse Proxy & Local DNS](#3-nginx-reverse-proxy--local-dns)
  * [4. Database Schema Initialization](#4-database-schema-initialization)
* [Service Outage Simulation & Troubleshooting](#-service-outage-simulation--troubleshooting)
  * [Incident Scenario](#incident-scenario)
  * [Log Analysis](#log-analysis)
  * [Root Cause](#root-cause)
 * [Resolution](#resolution)
 
---

## 📐 Network Architecture & Cloud Infrastructure

Network Architecture Diagram:

![Diagrama_Proyecto_Final.png](https://github.com/CodeCorrupted/cloud-ready-ops/blob/main/Diagrama_Proyecto_Final.png)

Unlike other cloud providers, GCP does not require defining a single global CIDR block for the entire VPC network; instead,
regional subnets are created with specific IP ranges. Firewall rules apply to the VPC level and are filtered to instances using **network tags**.

### Subnet Design

* **Public Subnet (`public-net`):** Range `10.0.1.0/24` (Region `us-central1`). Hosts the public-facing Web Server.
* **Private Subnet (`private-net`):** Range `10.0.2.0/24` (Region `us-central1`). Isolates the Database Server without a public IP.

---

## 🛠️ Tech Stack & Infrastructure

* **Cloud Provider:** Google Cloud Platform (VPC, Compute Engine, Cloud NAT, IAP).
* **Web Server / Proxy:** Nginx (Reverse Proxy) on Debian 13.
* **Database:** MariaDB 11.x on Debian 12.
* **Backend Application:** Java 21 (OpenJDK), Spring Boot 3.1.2, Spring Data JPA.
* **Frontend UI:** Jakarta Server Faces (JSF) / PrimeFaces.
* **Service Management:** Systemd (`zona_fit.service`).

---

## 🚀 Infrastructure Setup & Configuration

### 1. VPC and Subnet Creation

From the GCP Console (**VPC network > VPC networks**), create the network `final-project-vpc` in 
**Custom** mode and add `public-net` (`10.0.1.0/24`) and `private-net` (`10.0.2.0/24`).

### 2. Cloud NAT Configuration for Private Subnet

To allow instances in the private subnet to download packages and security updates without exposing them to the internet, 
configure a Cloud Router and Cloud NAT via Google Cloud CLI:

```sh
# 1. Create Cloud Router (Required for NAT)
gcloud compute routers create nat-router \
    --network=final-project-vpc \
    --region=us-central1

# 2. Create Cloud NAT linked to the private subnet
gcloud compute routers nats create custom-nat \
    --router=nat-router \
    --region=us-central1 \
    --auto-allocate-nat-external-ips \
    --nat-custom-subnet-ip-ranges=private-net
```

### 3. Firewall Rules (Security Groups)

```sh
# Allow Public HTTP (Port 80) to Web VM
gcloud compute firewall-rules create allow-public-http \
    --network=final-project-vpc \
    --direction=INGRESS \
    --action=ALLOW \
    --rules=tcp:80 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=web-server \
    --description="Allow public HTTP traffic to web server"

# Allow Public SSH (Port 22) to Web VM
gcloud compute firewall-rules create allow-public-ssh \
    --network=final-project-vpc \
    --direction=INGRESS \
    --action=ALLOW \
    --rules=tcp:22 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=web-server \
    --description="Allow SSH access to web server"

# Allow Internal Traffic from Public Subnet to Private Subnet (MariaDB Port 3306)
gcloud compute firewall-rules create allow-internal-db \
    --network=final-project-vpc \
    --direction=INGRESS \
    --action=ALLOW \
    --rules=tcp:3306 \
    --source-ranges=10.0.1.0/24 \
    --target-tags=db-server \
    --description="Allow internal traffic from web subnet to database"

# Allow Secure SSH via IAP to Private VM
gcloud compute firewall-rules create allow-iap-ssh-db \
    --network=final-project-vpc \
    --direction=INGRESS \
    --action=ALLOW \
    --rules=tcp:22 \
    --source-ranges=35.235.240.0/20 \
    --target-tags=db-server \
    --description="Allow SSH via IAP to private VM"
```

> **API Enablement Note**: *Ensure necessary GCP APIs are enabled before running CLI commands:*
> ```sh
> gcloud services enable compute.googleapis.com iap.googleapis.com servicenetworking.googleapis.com
> ```

### 4. VM Instance Provisioning

```sh
# Web Server (Public Subnet with External IP)
gcloud compute instances create web-server \
    --zone=us-central1-a \
    --machine-type=e2-micro \
    --network-interface=subnet=public-net,network-tier=PREMIUM \
    --tags=web-server \
    --image-family=debian-13 \
    --image-project=debian-cloud \
    --boot-disk-size=10GB \
    --description="Web Server in Public Subnet"

# Database Server (Private Subnet without External IP)
gcloud compute instances create db-server \
    --zone=us-central1-a \
    --machine-type=e2-micro \
    --network-interface=subnet=private-net,no-address \
    --tags=db-server \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --boot-disk-size=10GB \
    --description="Database Server in Private Subnet"
```

## 🌐 Service Provisioning

### Web Server (Nginx - Initial Test)

On `web-server`:

```sh
sudo apt update && sudo apt upgrade -y && sudo apt install nginx -y
```

### Database Server (MariaDB)

On `db-server`:

```sh
sudo apt update && sudo apt upgrade -y && sudo apt install mariadb-server -y
sudo mariadb-secure-installation
```

Edit `/etc/mysql/mariadb.conf.d/50-server.conf` and set `bind-address = 0.0.0.0`. Enable service:

```sh
sudo systemctl start mariadb && sudo systemctl enable mariadb
```

Create database and application user restricted to the public subnet:

```sql
CREATE DATABASE mi_base_de_datos;
CREATE USER 'usuario_web'@'10.0.1.%' IDENTIFIED BY 'your_secure_password';
GRANT ALL PRIVILEGES ON mi_base_de_datos.* TO 'usuario_web'@'10.0.1.%';
FLUSH PRIVILEGES;

USE mi_base_de_datos;
CREATE TABLE Empleados (id INT PRIMARY KEY, nombre VARCHAR(50));
INSERT INTO Empleados (id, nombre) VALUES (1, 'Administrador');
```

## ☕ Java Application Deployment (Zona Fit CRUD)

>🔗 **Source Code Repository:** *The Java Spring Boot / JSF application source code is
>hosted at ![CodeCorrupted/simple-crud](https://github.com/CodeCorrupted/simple-crud).*

### 1. Java JRE & Application Files Setup

On `web-server`:

```sh
sudo apt update && sudo apt install openjdk-21-jre-headless -y
sudo mkdir -p /var/www/zona_fit
sudo mv ~/app.jar /var/www/zona_fit/zona_fit.jar
sudo chown -R www-data:www-data /var/www/zona_fit
```

### 2. Environment Variables & Systemd Configuration

Create secured environment file `/etc/zona_fit.env`:

```
DB_HOST=db-server
DB_PORT=3306
DB_NAME=zona_fit_db
DB_USER=usuario_web
DB_PASS=your_secure_password
```

Adjust file permissions:

```sh
sudo chmod 600 /etc/zona_fit.env
sudo chown www-data:www-data /etc/zona_fit.env
```

Create Systemd unit configuration at `/etc/systemd/system/zona_fit.service`:

```ini
[Unit]
Description=Java Zona Fit Application
After=network.target

[Service]
User=www-data
EnvironmentFile=/etc/zona_fit.env
ExecStart=/usr/bin/java -jar /var/www/zona_fit/zona_fit.jar
SuccessExitStatus=143
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### 3. Nginx Reverse Proxy & Local DNS

Update `/etc/nginx/sites-available/default` to forward traffic to port `8080`:

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    location / {
        proxy_pass [http://127.0.0.1:8080](http://127.0.0.1:8080);
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Reload Nginx:

```sh
sudo systemctl restart nginx.service
```

Add local DNS entry in `/etc/hosts` on `web-server`:

```
10.0.2.x db-server
```

### 4. Database Schema Initialization

On `db-server`:

```sql
CREATE DATABASE zona_fit_db;
GRANT ALL PRIVILEGES ON zona_fit_db.* TO 'usuario_web'@'10.0.1.%';
FLUSH PRIVILEGES;

USE zona_fit_db;
CREATE TABLE cliente (
    id INT NOT NULL AUTO_INCREMENT,
    nombre VARCHAR(50) NULL,
    apellido VARCHAR(50) NULL,
    membresia INT NULL,
    PRIMARY KEY (id),
    UNIQUE INDEX membresia_UNIQUE (membresia ASC)
);
```

Enable and start application service:

```sh
sudo systemctl daemon-reload
sudo systemctl enable zona_fit --now
```

## 🛠️ Service Outage Simulation & Troubleshooting

### Incident Scenario

Following a routine maintenance update, web service configuration changes caused the web 
application to become unavailable, returning connection error responses.

### Log Analysis

Inspecting system service logs using `journalctl`:

```sh
sudo journalctl -u zona_fit.service -n 50 -f
```

Identified exception in the application startup log:

```
Aug 23 20:29:02 web-server java[7265]: HikariPool-1 - Exception during pool initialization.
Aug 23 20:29:02 web-server java[7265]: java.sql.SQLNonTransientConnectionException: Socket fail to connect to host:address=(host=db-server)(port=3036)(type=primary). Connect timed out
```

### Root Cause

Log output revealed a Connect timed out error attempting to target port `3036`.
Reviewing `/etc/zona_fit.env` confirmed a typo in the database port definition:

```
# Misconfigured Port:
DB_PORT=3036
```

### Resolution

Updated the environment file to use the default MariaDB/MySQL port `3306`:

```
DB_PORT=3306
```

Restarted the daemon (`sudo systemctl restart zona_fit`), successfully re-establishing database connectivity and 
restoring full access to the web interface at `http://web-server-public-ip/index.xhtml`.
