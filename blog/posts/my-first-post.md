
---
slugTitle: my-first-post
title: "My First Post"
date: "2025-10-06"
topics: ["AWS", "DevOps", "Infrastructure"]
image: "[https://guilherme-beckman.github.io/portfolio-assets/liads/images/liads.png](https://guilherme-beckman.github.io/portfolio-assets/liads/images/liads.png)"

---

![[Pasted image 20251006152233.png]]

---

## 🧭 Architecture Overview

The current infrastructure consists of a **reverse proxy (Nginx)**, an **API Gateway (Kong)**, a **Service Discovery (Consul)**, and the **databases (Postgres and MySQL)**.
The goal is to centralize access control, routing, security, and service discovery within an **isolated Docker network** (`kong-net`).

The communication flow can be visualized as follows:

```
User Request → Nginx → Kong Gateway → Consul → Services
```

---

## ⚙️ Request Flow

1. **Entry via Nginx**

   * The user makes an HTTP or HTTPS request to the configured domain.
   * Nginx acts as the entry point (reverse proxy) and redirects HTTP → HTTPS.
   * After that, it forwards the request to the **Kong Gateway** within the same Docker network.
2. **Kong Gateway (API Gateway)**

   * Kong maintains a list of configured **routes** (`route1`, `route2`, `route3`, etc.), each pointing to a logical service.
   * When a request arrives, Kong identifies the corresponding route and queries **Consul** to discover the actual service IP and port.
   * After receiving the information from Consul, it forwards the request to the target container (e.g., the `pantanal-app` service or another microservice).
3. **Consul (Service Discovery)**

   * Consul acts as the internal **DNS resolver** of the architecture.
   * It is responsible for registering and storing the IPs and ports of available services.
   * Kong uses the address `172.18.0.10:8600` (Consul’s DNS port) to resolve services by name (`service.consul`).
4. **Response to the User**

   * The service returns the response to Kong.
   * Kong forwards it back to Nginx.
   * Nginx returns the final result to the client.

---

## 🐳 Structure of `docker-compose.yml`

The `docker-compose.yml` file defines **only the base server infrastructure**, without including business microservices. It contains the following components:

| Service             | Function                              | IP            | Port(s)    | Notes                                        |
| ------------------- | ------------------------------------- | ------------- | ---------- | -------------------------------------------- |
| **nginx**           | HTTPS/HTTP entry point, reverse proxy | `172.18.0.12` | 80, 443    | Redirects and forwards to Kong               |
| **kong-gateway**    | Main API Gateway                      | Automatic     | 8000–8445  | Routing, authentication, and traffic control |
| **kong-database**   | PostgreSQL database used by Kong      | `172.18.0.11` | 5432       | Stores configurations and routes             |
| **kong-migrations** | Runs Kong database migrations         | —             | —          | Executed before `kong-gateway` starts        |
| **consul**          | Service discovery (internal DNS)      | `172.18.0.10` | 8500, 8600 | Name resolution and service status           |
| **mysql**           | Database for the `Pantanal 3D` system | `172.18.0.14` | 3306       | Used only by the Pantanal application        |
| **pantanal-app**    | Laravel application (Pantanal 3D)     | `172.18.0.15` | 8080       | Connected to MySQL; will be extracted later  |

---

## 🔄 Next Steps

1. **Create a “Service Registrar”**

   * A lightweight service (can be built in Go, Node.js, or Java Spring WebFlux) that automatically registers new services in **Consul**.
   * It can monitor running containers (via Docker SDK or Consul API) and dynamically register/unregister them.
2. **Remove `pantanal-app` from this compose**

   * The current `docker-compose.yml` should contain **only the base infrastructure** (Nginx, Kong, Consul, databases).
   * The `pantanal-app` should be moved to a separate file (`docker-compose.services.yml`) dedicated to applications.
3. **Centralize DNS in Consul**

   * Ensure all services register with names like `service-name.service.consul`, allowing Kong to automatically resolve them via internal DNS (`KONG_DNS_RESOLVER`).
4. **Automate Kong route configuration**

   * Create a script that dynamically registers routes in Kong when new services are detected in Consul.

---

Quer que eu também traduza o nome das seções (“Visão Geral da Arquitetura”, “Fluxo da Requisição” etc.) para inglês formal de documentação técnica (ex: “System Overview”, “Request Lifecycle”)? Isso deixaria o post com um tom mais profissional em inglês.
