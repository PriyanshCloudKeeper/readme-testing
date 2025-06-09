# Janus: SCIM 2.0 to Keycloak Bridge

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Java Version](https://img.shields.io/badge/Java-17-blue.svg)](https://www.java.com)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2.0-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![Keycloak](https://img.shields.io/badge/Keycloak-26.1.4-blue.svg)](https://www.keycloak.org/)

**Janus** is a robust, production-ready service that acts as a bridge between the **SCIM 2.0 protocol** and the **Keycloak Admin API**. It exposes a standard, compliant SCIM 2.0 interface, allowing any SCIM-compatible Identity Provider (IdP) or client to manage users and groups in a Keycloak instance seamlessly.

The name **Janus** is inspired by the Roman god of gates, transitions, and doorways, perfectly symbolizing this application's role as a gateway between two distinct systems.

---

## 🌟 Features

-   **SCIM 2.0 Compliance**: Implements core SCIM 2.0 resources and operations.
    -   **Users**: Create, Read (Get), Update (PUT/PATCH), and Delete.
    -   **Groups**: Create, Read, Update (PUT/PATCH), and Delete, including member management.
    -   **Discovery**: `ServiceProviderConfig`, `ResourceTypes`, and `Schemas` endpoints are fully implemented.
-   **Secure by Design**: Endpoints are secured with OAuth 2.0 JWT Bearer Tokens, validated against a Keycloak realm.
-   **Robust & Scalable**: Built with Spring Boot for high performance and easy containerization.
-   **Enterprise-Ready**: Maps standard SCIM attributes and the Enterprise User extension to Keycloak user attributes.
-   **Extensible**: Cleanly structured with mappers and services, making it easy to add custom attribute mappings.

## 🏛️ Architecture

The application follows a classic three-tier architecture within a Dockerized environment, orchestrated by Docker Compose and fronted by an NGINX reverse proxy.

1.  **NGINX Reverse Proxy**: Handles incoming traffic, terminates SSL/TLS, and routes requests to the appropriate service based on the domain name. It ensures both Keycloak and the Janus bridge are accessible via standard HTTPS.
2.  **Janus (SCIM Bridge)**: The core Spring Boot application that translates SCIM requests to Keycloak API calls.
3.  **Keycloak**: The Identity and Access Management system, responsible for authentication and storing user/group data.
4.  **MariaDB**: The persistent database backend for Keycloak.


*A high-level diagram of the deployment architecture.*

## 🚀 Getting Started

These instructions will guide you through setting up the complete environment on a fresh server (e.g., an AWS EC2 instance).

### Prerequisites

-   A Linux server (Ubuntu 22.04 LTS is recommended).
-   Two registered domain names (e.g., `keycloak.yourdomain.com` and `scim.yourdomain.com`).
-   DNS `A` records for both domains pointing to your server's public IP address.
-   **Docker** and **Docker Compose** installed.
    ```bash
    # Install Docker
    sudo apt-get update
    sudo apt-get install -y docker.io
    sudo systemctl start docker
    sudo systemctl enable docker
    sudo usermod -aG docker $USER
    newgrp docker # Activate group change without needing to log out/in
    
    # Install Docker Compose
    sudo apt-get install -y docker-compose-v2
    ```
-   **NGINX** and **Certbot** installed.
    ```bash
    sudo apt-get install -y nginx certbot python3-certbot-nginx
    ```

### 1. Clone the Repository

Clone the project to your server. The project includes the Janus source code and the `docker-compose.yml` file.


### 2. Configure Environment Variables

Create a .env file in the root directory of the cloned project. This file will store all your sensitive and environment-specific configurations.

```bash
nano .env
```

Paste the following content into the file, adjusting the values to match your setup.

```bash
# Database Credentials
MARIADB_ROOT_PASSWORD=your_strong_mariadb_root_password
MARIADB_DATABASE=keycloak
MARIADB_USER=keycloak
MARIADB_PASSWORD=your_strong_keycloak_db_password

# Keycloak Admin Credentials & Host
KEYCLOAK_ADMIN_USER=admin
KEYCLOAK_ADMIN_PASSWORD=your_strong_keycloak_admin_password
KC_HOSTNAME_URL=https://keycloak.yourdomain.com
KEYCLOAK_HOST_HTTP_PORT=8081

# Janus (SCIM Bridge) Host
SCIM_BRIDGE_HOST_PORT=8082

# Janus Configuration (Keycloak Admin Client & Security)
# The realm that issues tokens to secure the bridge itself
KEYCLOAK_BRIDGE_SECURED_BY_REALM=master
# The realm the admin client will connect to (often the same as above)
KEYCLOAK_ADMIN_CLIENT_REALM=master
# The realm where users/groups will be provisioned
KEYCLOAK_PROVISIONING_TARGET_REALM=master
# Client ID and secret for the bridge (created in Keycloak in a later step)
KEYCLOAK_ADMIN_CLIENT_ID=scim-bridge-client
KEYCLOAK_ADMIN_CLIENT_SECRET=generate_a_strong_secret_here
# The public-facing URL of the SCIM bridge
SCIM_BRIDGE_EXTERNAL_URL=https://scim.yourdomain.com
```

### 3. Set Up NGINX Reverse Proxy & SSL

We will create NGINX server blocks to route traffic to our Docker containers and use Certbot to secure them with free, trusted SSL certificates from Let's Encrypt.

-   Create NGINX Configuration Files

    Create a configuration file for Keycloak:
    ```bash
    sudo nano /etc/nginx/sites-available/keycloak.yourdomain.com
    ```

    Paste this configuration (replace keycloak.yourdomain.com with your actual domain):

    ```bash
    server {
        listen 80;
        server_name keycloak.yourdomain.com;
        root /var/www/html; # Required for Certbot webroot challenge
        location ~ /.well-known/acme-challenge/ {
            allow all;
        }
    }
    ```  
    
    Create a configuration file for the SCIM bridge:
    ```bash
    sudo nano /etc/nginx/sites-available/scim.yourdomain.com
    ```

    Paste this configuration (replace scim.yourdomain.com with your actual domain):
    ```bash
    server {
        listen 80;
        server_name scim.yourdomain.com;
        root /var/www/html; # Required for Certbot webroot challenge
        location ~ /.well-known/acme-challenge/ {
            allow all;
        }
    }
    ```
