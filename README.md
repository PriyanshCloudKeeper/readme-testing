# Janus: SCIM 2.0 to Keycloak Bridge

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Java Version](https://img.shields.io/badge/Java-17-blue.svg)](https://www.java.com)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2.0-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![Keycloak](https://img.shields.io/badge/Keycloak-26.1.4-blue.svg)](https://www.keycloak.org/)

**Janus** is a robust, production-ready service that acts as a bridge between the **SCIM 2.0 protocol** and the **Keycloak Admin API**. It exposes a standard, compliant SCIM 2.0 interface, allowing any SCIM-compatible Identity Provider (IdP) or client to manage users and groups in a Keycloak instance seamlessly.

The name **Janus** is inspired by the Roman god of gates, transitions, and doorways, perfectly symbolizing this application's role as a gateway between two distinct systems.

## 🌟 Features

-   **SCIM 2.0 Compliance**: Implements core SCIM 2.0 resources and operations.
    -   **Users**: Create, Read (Get), Update (PUT/PATCH), and Delete.
    -   **Groups**: Create, Read, Update (PUT/PATCH), and Delete, including member management.
    -   **Discovery**: `ServiceProviderConfig`, `ResourceTypes`, and `Schemas` endpoints are fully implemented.
-   **Secure by Design**: Endpoints are secured with OAuth 2.0 JWT Bearer Tokens, validated against a Keycloak realm.
-   **Robust & Scalable**: Built with Spring Boot for high performance and easy containerization.
-   **Enterprise-Ready**: Maps standard SCIM attributes and the Enterprise User extension to Keycloak user attributes.
-   **Extensible**: Cleanly structured with mappers and services, making it easy to add custom attribute mappings.

## 🚀 Getting Started

Please refer to the full documentation [here](https://yourdomain.com/docs) for setup instructions, prerequisites, and usage examples.

## 🧪 Using the SCIM API

To interact with the SCIM API, first obtain a token:

```bash
export access_token=$(curl -s -d "client_id=admin-cli" -d "username=\${KEYCLOAK_ADMIN_USER}" -d "password=\${KEYCLOAK_ADMIN_PASSWORD}" -d "grant_type=password" "https://keycloak.yourdomain.com/realms/master/protocol/openid-connect/token" | jq -r .access_token)
```

Then you can make requests such as:

```bash
curl -X GET "https://scim.yourdomain.com/scim/v2/Users" -H "Authorization: Bearer $access_token"
```

## 🛠️ License

This project is licensed under the MIT License.
