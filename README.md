# ⚙️ E-commerce Configuration Repository

This repository stores **externalized configurations** for all microservices in the E-commerce Microservices Ecosystem. It is consumed by the [Config Server](https://github.com/DanLearnings/ecommerce-config-server) using Spring Cloud Config, enabling centralized configuration management with version control.

---

## 🎯 Purpose

In a microservices architecture, managing configurations across multiple services becomes complex. This repository solves that problem by:

- ✅ **Centralizing** all application configurations in one place
- ✅ **Versioning** configurations using Git for traceability and rollback
- ✅ **Environment-specific** configurations (dev, prod, staging)
- ✅ **Secure** storage of sensitive data (externalized via environment variables)
- ✅ **Dynamic** configuration updates without rebuilding services

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  ecommerce-config-repo (Git)                 │
│                                                               │
│  ├── api-gateway.yml                                         │
│  ├── inventory-service.yml                                   │
│  ├── notification-service.yml                                │
│  └── application.yml (common configs)                        │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Git Clone/Pull
                     ▼
        ┌────────────────────────────┐
        │  ecommerce-config-server   │
        │      (Port 8888)           │
        └────────┬───────────────────┘
                 │
                 │ HTTP REST API
                 │ (e.g., /inventory-service/default)
                 │
    ┌────────────┼────────────┬────────────────┐
    │            │            │                │
    ▼            ▼            ▼                ▼
┌─────────┐ ┌──────────┐ ┌────────────┐ ┌──────────────┐
│   API   │ │Inventory │ │Notification│ │    Future    │
│ Gateway │ │ Service  │ │  Service   │ │   Services   │
└─────────┘ └──────────┘ └────────────┘ └──────────────┘
```

---

## 📂 Repository Structure

```
ecommerce-config-repo/
├── api-gateway.yml                  # API Gateway configuration
├── inventory-service.yml            # Inventory Service configuration
├── notification-service.yml         # Notification Service configuration
├── application.yml                  # Common configurations (optional)
├── application-dev.yml              # Dev environment (future)
├── application-prod.yml             # Production environment (future)
└── README.md                        # This file
```

### Naming Convention

Spring Cloud Config automatically maps files to services based on naming:

- `{service-name}.yml` → Default profile
- `{service-name}-{profile}.yml` → Specific profile (e.g., `inventory-service-prod.yml`)
- `application.yml` → Shared across all services

---

## 📋 Configuration Files

### 1. api-gateway.yml

Configurations for the API Gateway service.

**Key Settings:**
- Server port (8080)
- Eureka registration
- Gateway discovery locator (WebFlux)
- Route definitions (if manual routing is needed)

**Example:**
```yaml
server:
  port: 8080

spring:
  application:
    name: api-gateway
  
  cloud:
    gateway:
      server:
        webflux:
          discovery:
            locator:
              enabled: true
              lower-case-service-id: true

eureka:
  client:
    service-url:
      defaultZone: http://service-registry:8761/eureka/
```

---

### 2. inventory-service.yml

Configurations for the Inventory Service.

**Key Settings:**
- Server port (8081)
- H2 Database configuration
- Eureka registration
- JPA/Hibernate settings

**Example:**
```yaml
server:
  port: 8081

spring:
  application:
    name: inventory-service
  
  datasource:
    url: jdbc:h2:mem:inventorydb
    driver-class-name: org.h2.Driver
    username: sa
    password:
  
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true

eureka:
  client:
    service-url:
      defaultZone: http://service-registry:8761/eureka/
```

---

### 3. notification-service.yml

Configurations for the Notification Service.

**Key Settings:**
- Server port (8083)
- RabbitMQ connection
- Gmail SMTP configuration
- H2 Database configuration

**Example:**
```yaml
server:
  port: 8083

spring:
  application:
    name: notification-service
  
  rabbitmq:
    host: rabbitmq
    port: 5672
    username: admin
    password: admin
  
  mail:
    host: smtp.gmail.com
    port: 587
    username: ${MAIL_USERNAME}
    password: ${MAIL_PASSWORD}
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true

eureka:
  client:
    service-url:
      defaultZone: http://service-registry:8761/eureka/
```

**⚠️ Note:** Email credentials are externalized using environment variables (`${MAIL_USERNAME}`, `${MAIL_PASSWORD}`) for security.

---

## 🔐 Security Best Practices

### 1. Environment Variables for Secrets

**Never commit sensitive data** (passwords, API keys, tokens) directly to this repository.

**Instead, use placeholders:**
```yaml
spring:
  mail:
    username: ${MAIL_USERNAME}
    password: ${MAIL_PASSWORD}
```

**Set environment variables in:**
- Docker: `-e MAIL_USERNAME=your-email@gmail.com`
- Kubernetes: ConfigMaps and Secrets
- Cloud Platforms: Environment variable management

---

### 2. Private Repository (Recommended for Production)

For production environments, make this repository **private** and configure SSH/token authentication:

**Config Server with SSH:**
```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: git@github.com:DanLearnings/ecommerce-config-repo.git
          clone-on-start: true
          default-label: main
```

---

### 3. Encryption (Advanced)

Spring Cloud Config supports **encrypted properties** using symmetric or asymmetric encryption:

```yaml
spring:
  datasource:
    password: '{cipher}FKSAJDFGYOS8F7GLHAKERGFHLSAJ'
```

See [Spring Cloud Config Encryption Documentation](https://cloud.spring.io/spring-cloud-config/multi/multi__spring_cloud_config_server.html#_encryption_and_decryption) for details.

---

## 🚀 How Services Consume Configurations

### 1. Service Configuration (application.yml)

Each microservice includes a minimal `application.yml`:

```yaml
spring:
  application:
    name: inventory-service
  
  config:
    import: "configserver:http://config-server:8888"
  
  cloud:
    config:
      fail-fast: true  # Fail if Config Server is unavailable
```

---

### 2. Startup Flow

```
1. Service starts
   └─→ Reads local application.yml
   └─→ Discovers Config Server URL

2. Service connects to Config Server
   └─→ Requests: GET /inventory-service/default
   └─→ Config Server clones/pulls this Git repo
   └─→ Returns merged configuration

3. Service applies configuration
   └─→ Merges with local application.yml
   └─→ Starts with final configuration
```

---

## 🧪 Testing Configurations

### 1. Via Config Server REST API

```bash
# Get inventory-service configuration
curl http://localhost:8888/inventory-service/default

# Get notification-service configuration
curl http://localhost:8888/notification-service/default

# Get api-gateway configuration
curl http://localhost:8888/api-gateway/default
```

**Expected Response:**
```json
{
  "name": "inventory-service",
  "profiles": ["default"],
  "label": null,
  "version": "commit-hash",
  "state": null,
  "propertySources": [
    {
      "name": "https://github.com/DanLearnings/ecommerce-config-repo/inventory-service.yml",
      "source": {
        "server.port": 8081,
        "spring.application.name": "inventory-service",
        ...
      }
    }
  ]
}
```

---

### 2. Verify Service Configuration

Check service logs on startup:

```
Fetching config from server at : http://config-server:8888
Located environment: name=inventory-service, profiles=[default], label=null, version=abc123, state=
```

---

## 🔄 Updating Configurations

### Without Restarting Services (Dynamic Refresh)

**1. Update configuration file in this repo:**
```bash
git add inventory-service.yml
git commit -m "Update inventory service port"
git push origin main
```

**2. Trigger Config Server refresh:**
```bash
curl -X POST http://localhost:8888/actuator/refresh
```

**3. Trigger service refresh (if @RefreshScope is configured):**
```bash
curl -X POST http://localhost:8081/actuator/refresh
```

**Note:** This requires `@RefreshScope` annotation on beans and `spring-boot-starter-actuator` dependency.

---

### With Service Restart (Standard)

```bash
# Update config
git push origin main

# Restart Config Server (to pull latest changes)
docker restart config-server

# Restart affected service
docker restart inventory-service
```

---

## 🌍 Multi-Environment Support

### Setup

Create environment-specific configuration files:

```
ecommerce-config-repo/
├── inventory-service.yml           # Default/shared
├── inventory-service-dev.yml       # Development
├── inventory-service-prod.yml      # Production
└── inventory-service-staging.yml   # Staging
```

### Usage

**Activate profile in service:**
```yaml
spring:
  profiles:
    active: prod
```

**Or via environment variable:**
```bash
docker run -e SPRING_PROFILES_ACTIVE=prod inventory-service
```

**Config Server will automatically merge:**
1. `inventory-service.yml` (base)
2. `inventory-service-prod.yml` (override)

---

## 📊 Configuration Priority

Spring Cloud Config merges configurations in this order (highest priority first):

1. **Environment variables** (runtime)
2. **Command-line arguments**
3. **Config Server** (this repo)
4. **Local application.yml**
5. **Defaults in code**

**Example:**
```yaml
# config-repo/inventory-service.yml
server:
  port: 8081

# local application.yml
server:
  port: 9999  # This will be OVERRIDDEN by Config Server!
```

---

## 🐛 Troubleshooting

### Issue: Config Server cannot clone repository

**Solution:**
1. Verify Git URL in Config Server configuration
2. Ensure repository is public or SSH keys are configured
3. Check Config Server logs: `docker logs config-server`
4. Test Git access manually: `git clone <repo-url>`

---

### Issue: Service cannot connect to Config Server

**Solution:**
1. Verify Config Server is running: `curl http://localhost:8888/actuator/health`
2. Check service logs for connection errors
3. Ensure `spring.config.import` URL is correct
4. Try `fail-fast: false` for debugging (service will start with local config)

---

### Issue: Changes not reflected after Git push

**Solution:**
1. Restart Config Server: `docker restart config-server`
2. Trigger refresh: `curl -X POST http://localhost:8888/actuator/refresh`
3. Restart affected service
4. Verify Git commit was successful: `git log`

---

## 🔮 Future Enhancements

- [ ] **Encryption** - Encrypt sensitive properties
- [ ] **Webhooks** - Auto-refresh on Git push
- [ ] **Vault Integration** - Use HashiCorp Vault for secrets
- [ ] **Multi-repo Support** - Separate repos per environment
- [ ] **Configuration Validation** - Automated YAML validation
- [ ] **Audit Log** - Track configuration changes

---

## 📚 Additional Resources

- [Spring Cloud Config Documentation](https://cloud.spring.io/spring-cloud-config/)
- [Spring Cloud Config Server](https://github.com/DanLearnings/ecommerce-config-server)
- [Externalized Configuration Best Practices](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config)

---

## 🤝 Contributing

This repository is part of the [E-commerce Microservices Ecosystem](https://github.com/DanLearnings/ecommerce-ecosystem). When adding new services, create a corresponding `{service-name}.yml` file here.

---

## 📄 License

This project is for educational purposes.

---

## 👨‍💻 Author

**Danrley Brasil**
- GitHub: [@DanrleyBrasil](https://github.com/DanrleyBrasil)
- Organization: [DanLearnings](https://github.com/DanLearnings)

---

## 🔗 Related Repositories

- [ecommerce-ecosystem](https://github.com/DanLearnings/ecommerce-ecosystem) - Main hub
- [ecommerce-config-server](https://github.com/DanLearnings/ecommerce-config-server) - Consumes this repo
- [ecommerce-service-registry](https://github.com/DanLearnings/ecommerce-service-registry) - Eureka Server
- [ecommerce-api-gateway](https://github.com/DanLearnings/ecommerce-api-gateway) - API Gateway
- [ecommerce-inventory-service](https://github.com/DanLearnings/ecommerce-inventory-service) - Inventory Service
- [ecommerce-notification-service](https://github.com/DanLearnings/ecommerce-notification-service) - Notification Service

---

**Last Updated:** November 2025
