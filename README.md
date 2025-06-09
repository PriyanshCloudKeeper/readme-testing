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
-   **Secure by Design**: Endpoints are secured and support authentication via pre-configured static Bearer Tokens (ideal for IdPs with limited OAuth2 client capabilities). The internal Keycloak Admin Client communication remains secured by standard OAuth 2.0 JWTs
-   **Robust & Scalable**: Built with Spring Boot for high performance and easy containerization.
-   **Enterprise-Ready**: Maps standard SCIM attributes and the Enterprise User extension to Keycloak user attributes.
-   **Extensible**: Cleanly structured with mappers and services, making it easy to add custom attribute mappings.

## 🏛️ Architecture

The application follows a classic three-tier architecture within a Dockerized environment, orchestrated by Docker Compose and fronted by an NGINX reverse proxy.

1.  **NGINX Reverse Proxy**: Handles incoming traffic, terminates SSL/TLS, and routes requests to the appropriate service based on the domain name. It ensures both Keycloak and the Janus bridge are accessible via standard HTTPS.
2.  **Janus (SCIM Bridge)**: The core Spring Boot application that translates SCIM requests to Keycloak API calls.
3.  **Keycloak**: The Identity and Access Management system, responsible for authentication and storing user/group data.
4.  **MariaDB**: The persistent database backend for Keycloak.


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

### a. Create NGINX Configuration Files

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

### b. Enable the Sites

Enable the NGINX site configurations and restart NGINX:

```bash
sudo ln -s /etc/nginx/sites-available/keycloak.yourdomain.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/scim.yourdomain.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

### c. Obtain SSL Certificates

Use **Certbot** to obtain SSL certificates and automatically configure HTTPS:

```bash
sudo certbot --nginx -d keycloak.yourdomain.com -d scim.yourdomain.com
```

- Follow the prompts.
- Choose the option to redirect HTTP to HTTPS.
- Certbot will automatically modify the NGINX configuration to enable SSL.

---

### d. Update NGINX to Proxy to Docker Containers

Now modify the NGINX configurations to proxy incoming HTTPS traffic to the corresponding internal services.

#### Update the Keycloak NGINX Configuration

Edit the file:

```bash
sudo nano /etc/nginx/sites-available/keycloak.yourdomain.com
```

Locate the `server { listen 443 ssl; ... }` block and replace the `location /` block with:

```nginx
location / {
    proxy_set_header        Host $host;
    proxy_set_header        X-Real-IP $remote_addr;
    proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header        X-Forwarded-Proto $scheme;
    proxy_pass              http://localhost:8081; # Port from .env file
}
```

#### Update the SCIM Bridge NGINX Configuration

Edit the file:

```bash
sudo nano /etc/nginx/sites-available/scim.yourdomain.com
```

Replace the `location /` block with:

```nginx
location / {
    proxy_set_header        Host $host;
    proxy_set_header        X-Real-IP $remote_addr;
    proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header        X-Forwarded-Proto $scheme;
    proxy_pass              http://localhost:8082; # Port from .env file
}
```

Test and restart NGINX:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

### 4. Build and Run the Application Stack

Now that the environment is configured, you can start the services using Docker Compose.

```bash
# In the root of the project directory
docker compose up --build -d
```

This command will:

- Build the scim-bridge Docker image from the source code.
- Pull the keycloak and mariadb images.
- Start all three containers in detached mode (`-d`).

Check the logs to ensure everything started correctly:

```bash
docker compose logs -f
# Press Ctrl+C to exit logs
```

---

### 5. Configure Keycloak

After the containers are running, you need to create a client in Keycloak for the Janus bridge to use.

1. Navigate to your Keycloak instance: `https://keycloak.yourdomain.com`.
2. Log in with the admin credentials you set in the `.env` file.
3. Go to **Clients** in the **master** realm (or your target realm).
4. Click **Create client**.
5. Set the **Client ID** to match `KEYCLOAK_ADMIN_CLIENT_ID` from your `.env` file (e.g., `scim-bridge-client`).
6. Click **Next**.
7. Ensure **Client authentication** is **On** and select **Service accounts roles**.
8. Click **Save**.

A new set of tabs will appear. Go to the **Service account roles** tab.

- Click **Assign role**.
- Filter for **realm-management** and assign the following roles:
  - `manage-groups`
  - `manage-users`
  - `query-groups`
  - `query-users`
  - `view-users`

Go to the **Credentials** tab.

- You will see a **Client secret**.
- This secret must match the `KEYCLOAK_ADMIN_CLIENT_SECRET` in your `.env` file.
- If it doesn’t, regenerate it and update your `.env` file.
- Then restart the SCIM bridge container:

```bash
docker compose restart scim-bridge
```

---

### 🧪 Using the SCIM API

Your SCIM bridge is now ready to accept requests at `https://scim.yourdomain.com`.

#### Get an Access Token

Use the following command to get an access token from Keycloak:

```bash
export access_token=$(curl -s -d "client_id=admin-cli" -d "username=${KEYCLOAK_ADMIN_USER}" -d "password=${KEYCLOAK_ADMIN_PASSWORD}" -d "grant_type=password" "https://keycloak.yourdomain.com/realms/master/protocol/openid-connect/token" | jq -r .access_token)
```

#### Make API Calls

**Get all users:**

```bash
curl -X GET "https://scim.yourdomain.com/scim/v2/Users" \
-H "Authorization: Bearer $access_token"
```

**Create a new user:**

```bash
curl -X POST "https://scim.yourdomain.com/scim/v2/Users" \
-H "Authorization: Bearer $access_token" \
-H "Content-Type: application/scim+json" \
-d '{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
  "userName": "newuser-from-scim",
  "name": {
    "givenName": "Test",
    "familyName": "User"
  },
  "emails": [{
    "value": "test.user@example.com",
    "primary": true
  }],
  "active": true
}'
```
