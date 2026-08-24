# ☁️ Proyecto Final - Infraestructura Cloud en GCP & Despliegue Full Stack

Este repositorio contiene la documentación detallada y los scripts necesarios para el aprovisionamiento de una infraestructura de red VPC desacoplada en **Google Cloud Platform (GCP)**, junto con el despliegue y solución de problemas de una aplicación web Java Full Stack (Spring Boot + JSF) conectada a MariaDB.

---

## 📐 Arquitectura de Red e Infraestructura Cloud

Diagrama arquitectónico de la red:

![[Diagrama_Proyecto_Final.png]]

A diferencia de otros proveedores cloud, en GCP no creamos un CIDR global para la red VPC completa, sino que definimos subredes regionales asociadas a rangos de IP específicos. Las reglas de Firewall se aplican a nivel de VPC y se filtran hacia las instancias mediante **etiquetas de red (network tags)**.

### Subredes Diseñadas

* **Subred Pública (`public-net`):** Rango `10.0.1.0/24` (Región `us-central1`). Alberga el servidor web accesible desde Internet.
* **Subred Privada (`private-net`):** Rango `10.0.2.0/24` (Región `us-central1`). Aísla el servidor de base de datos sin IP pública.

---

## 🛠️ Tecnologías e Infraestructura

* **Cloud Provider:** Google Cloud Platform (VPC, Compute Engine, Cloud NAT, IAP).
* **Servidor Web / Proxy:** Nginx (Reverse Proxy) en Debian 13.
* **Base de Datos:** MariaDB 11.x en Debian 12.
* **Backend Application:** Java 21 (OpenJDK), Spring Boot 3.1.2, Spring Data JPA.
* **Frontend UI:** Jakarta Server Faces (JSF) / PrimeFaces.
* **Orquestación de Servicios:** Systemd (`zona_fit.service`).

---

## 🚀 Pasos de Configuración e Infraestructura

### 1. Creación de la VPC y Subredes

Desde la consola de GCP (**VPC network > VPC networks**), se crea la red `final-project-vpc` en modo **Custom** y se añaden las subredes `public-net` (`10.0.1.0/24`) y `private-net` (`10.0.2.0/24`).

### 2. Configuración de Cloud NAT para la Subred Privada

Para permitir que la subred privada descargue paquetes y actualizaciones sin exponerse a Internet, creamos un Cloud Router y Cloud NAT mediante la Google Cloud CLI:

```sh
# 1. Crear el Cloud Router (necesario para el NAT)
gcloud compute routers create nat-router \
    --network=final-project-vpc \
    --region=us-central1

# 2. Crear el Cloud NAT asociado a la subred privada
gcloud compute routers nats create custom-nat \
    --router=nat-router \
    --region=us-central1 \
    --auto-allocate-nat-external-ips \
    --nat-custom-subnet-ip-ranges=private-net
```

### 3. Reglas de Firewall (Security Groups)

```sh
# Permitir HTTP (Puerto 80) a la VM Web desde Internet
gcloud compute firewall-rules create allow-public-http \
    --network=final-project-vpc \
    --direction=INGRESS \
    --action=ALLOW \
    --rules=tcp:80 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=web-server \
    --description="Permite trafico HTTP saliente/entrante publico al servidor Web"

# Permitir SSH (Puerto 22) a la VM Web desde Internet
gcloud compute firewall-rules create allow-public-ssh \
    --network=final-project-vpc \
    --direction=INGRESS \
    --action=ALLOW \
    --rules=tcp:22 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=web-server \
    --description="Permite acceso SSH al servidor Web"

# Permitir Tráfico Interno desde Subred Pública a Privada (MariaDB Puerto 3306)
gcloud compute firewall-rules create allow-internal-db \
    --network=final-project-vpc \
    --direction=INGRESS \
    --action=ALLOW \
    --rules=tcp:3306 \
    --source-ranges=10.0.1.0/24 \
    --target-tags=db-server \
    --description="Permite trafico interno desde la subred web hacia la base de datos"

# Permitir SSH seguro mediante IAP a la VM Privada
gcloud compute firewall-rules create allow-iap-ssh-db \
    --network=final-project-vpc \
    --direction=INGRESS \
    --action=ALLOW \
    --rules=tcp:22 \
    --source-ranges=35.235.240.0/20 \
    --target-tags=db-server \
    --description="Permite acceso SSH por IAP a la VM privada"
```

**Nota de habilitación de APIs**: *Asegúrate de habilitar los servicios necesarios ejecutando:*

```sh
gcloud services enable compute.googleapis.com iap.googleapis.com servicenetworking.googleapis.com
```

### 4. Aprovisionamiento de Instancias (VMs)

```sh
# Servidor Web (Subred Pública con IP Externa)
gcloud compute instances create web-server \
    --zone=us-central1-a \
    --machine-type=e2-micro \
    --network-interface=subnet=public-net,network-tier=PREMIUM \
    --tags=web-server \
    --image-family=debian-13 \
    --image-project=debian-cloud \
    --boot-disk-size=10GB \
    --description="Servidor Web en Subred Publica"

# Servidor de Base de Datos (Subred Privada sin IP Externa)
gcloud compute instances create db-server \
    --zone=us-central1-a \
    --machine-type=e2-micro \
    --network-interface=subnet=private-net,no-address \
    --tags=db-server \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --boot-disk-size=10GB \
    --description="Servidor de Base de Datos en Subred Privada"
```

## 🌐 Configuración de Servicios

### Servidor Web (Nginx - Prueba Inicial)

En `web-server`:

```sh
sudo apt update && sudo apt upgrade -y && sudo apt install nginx -y
```

### Servidor de Base de Datos (MariaDB)

En `db-server`:

```sh
sudo apt update && sudo apt upgrade -y && sudo apt install mariadb-server -y
sudo mariadb-secure-installation
```

Modificar `/etc/mysql/mariadb.conf.d/50-server.conf` configurando `bind-address = 0.0.0.0`. Iniciar servicio:

```sh
sudo systemctl start mariadb && sudo systemctl enable mariadb
```

Crear base de datos y usuario restringido a la subred pública:

```sql
CREATE DATABASE mi_base_de_datos;
CREATE USER 'usuario_web'@'10.0.1.%' IDENTIFIED BY 'tu_contraseña_segura';
GRANT ALL PRIVILEGES ON mi_base_de_datos.* TO 'usuario_web'@'10.0.1.%';
FLUSH PRIVILEGES;

USE mi_base_de_datos;
CREATE TABLE Empleados (id INT PRIMARY KEY, nombre VARCHAR(50));
INSERT INTO Empleados (id, nombre) VALUES (1, 'Administrador');
```

## ☕ Despliegue de la Aplicación Java (CRUD Zona Fit)

> 🔗 **Repositorio del código fuente:** *El código de esta aplicación Java Spring Boot / JSF se
> encuentra disponible en [CodeCorrupted/simple-crud](https://github.com/CodeCorrupted/simple-crud).*

### 1. Instalación de Java y Archivos de Aplicación

En `web-server`:

```sh
sudo apt update && sudo apt install openjdk-21-jre-headless -y
sudo mkdir -p /var/www/zona_fit
sudo mv ~/app.jar /var/www/zona_fit/zona_fit.jar
sudo chown -R www-data:www-data /var/www/zona_fit
```

### 2. Configuración de Variables de Entorno y Systemd

Crear el archivo seguro `/etc/zona_fit.env`:

```
DB_HOST=db-server
DB_PORT=3306
DB_NAME=zona_fit_db
DB_USER=usuario_web
DB_PASS=tu_contraseña_segura
```

Ajustar permisos:

```sh
sudo chmod 600 /etc/zona_fit.env
sudo chown www-data:www-data /etc/zona_fit.env
```

Crear el servicio de Systemd `/etc/systemd/system/zona_fit.service`:

```ini
[Unit]
Description=Aplicacion Java Zona Fit
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

### 3. Reverse Proxy en Nginx y DNS Local

Modificar `/etc/nginx/sites-available/default` para redirigir el tráfico al puerto `8080`:

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

Reiniciar Nginx:

```sh
sudo systemctl restart nginx.service.
```

Añadir la resolución local en `/etc/hosts` de `web-server`:

```
10.0.2.x db-server
```

### 4. Inicialización de la Base de Datos Final

En `db-server`:

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

Iniciar el servicio Java:

```sh
sudo systemctl daemon-reload
sudo systemctl enable zona_fit --now
```

## 🛠️ Simulación de Caída de Servicio y Diagnóstico (Troubleshooting)

### Escenario de Incidencia

Tras una actualización del sistema y modificaciones en las configuraciones de producción, 
la aplicación web dejó de estar disponible retornando errores de conexión.

### Análisis de Logs

Ejecutando la inspección del servicio con `journalctl`:

```sh
sudo journalctl -u zona_fit.service -n 50 -f
```

Se identificó el siguiente error en la pila de ejecución:

```
Aug 23 20:29:02 web-server java[7265]: HikariPool-1 - Exception during pool initialization.
Aug 23 20:29:02 web-server java[7265]: java.sql.SQLNonTransientConnectionException: Socket fail to connect to host:address=(host=db-server)(port=3036)(type=primary). Connect timed out
```

### Causa Raíz

El log muestra un fallo de *Connect timed out* hacia el puerto `3036`. Al revisar `/etc/zona_fit.env`, 
se detectó un error tipográfico en la asignación del puerto de MariaDB:

```
# Error detectado:
DB_PORT=3036
```

### Resolución

Se corrigió la variable de entorno ajustando el puerto estándar de MySQL/MariaDB a `3306`:

```
DB_PORT=3306
```

Posteriormente, se reinició el servicio (`sudo systemctl restart zona_fit`), restableciendo el funcionamiento normal y 
permitiendo el acceso a la interfaz web en `http://IP-publica-web-server/index.xhtml`.
