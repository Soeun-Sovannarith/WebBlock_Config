# Externalized Configuration Repository

This folder contains the centralized, externalized configuration files for all WebBlock microservices.

> **Secrets:** the values here are local-dev defaults only (`webblock`/`webblock`). Every value is a
> `${ENV_VAR:default}` placeholder, so real credentials are injected at runtime via environment variables or
> Kubernetes Secrets. Never commit real passwords or tokens to this repository.

```
config/
├── .gitignore                     # Keeps OS files and real secret overrides out of git
├── README.md                      # Guide and usage instructions
├── application.yaml               # Shared baseline configuration (Spring Cloud Config Server)
├── user-service.yaml              # Spring Boot: user-service (:8081, user_db)
├── project-service.yaml           # Spring Boot: project-service (:8082, project_db, user-service client)
├── payment-service.yaml           # Spring Boot: payment-service (:8085, payment_db)
├── notification-service.yaml      # Spring Boot: notification-service (:8086, notification_db)
├── aicode-service.yaml            # FastAPI: aicode-service (:8083, aicode_db)
├── aicode-service.env             # .env format for aicode-service
├── provisioning-service.yaml      # FastAPI: provisioning-service (:8084, provisioning_db)
└── provisioning-service.env       # .env format for provisioning-service
```

---

## 1. How Configuration is Structured

### Spring Boot Services
Spring Boot services load configurations hierarchically:
1. **`application.yaml`**: Contains common settings shared across all Spring Boot services:
   - RabbitMQ connection (`${RABBITMQ_HOST}`, `${RABBITMQ_PORT}`, credentials)
   - Keycloak JWT Resource Server (`${KEYCLOAK_ISSUER_URI}`, `${KEYCLOAK_JWK_SET_URI}`)
   - Actuator probes and Prometheus metrics (`/actuator/health/liveness`, `/actuator/health/readiness`)
   - Virtual threads and problem details
   - Graceful shutdown and framework forward headers
2. **`<service-name>.yaml`**: Contains service-specific settings:
   - `spring.application.name`
   - Dedicated PostgreSQL database URL (`user_db`, `project_db`, `payment_db`, `notification_db`)
   - Dedicated local HTTP port (`8081`, `8082`, `8085`, `8086`)
   - Downstream service HTTP client base URLs (e.g. `user-service` URL inside `project-service`)

### FastAPI (Python) Services
FastAPI services use Pydantic Settings (`BaseSettings`):
- YAML format: `aicode-service.yaml` and `provisioning-service.yaml`
- Dotenv format: `aicode-service.env` and `provisioning-service.env`

---

## 2. Usage Scenarios

### Option A: Spring Cloud Config Server
If you run a Spring Cloud Config Server:
- **Native File System**: Point the config server to this folder:
  ```yaml
  spring:
    profiles:
      active: native
    cloud:
      config:
        server:
          native:
            search-locations: file:./config
  ```
- **Git Backend**: Point the config server at this repository (the files live at the repo root, so no `search-paths` is needed):
  ```yaml
  spring:
    cloud:
      config:
        server:
          git:
            uri: <git-repo-url>
            default-label: main
  ```
- Each client service retrieves its configuration at startup via:
  ```yaml
  spring:
    config:
      import: "optional:configserver:http://localhost:8888"
  ```

### Option B: Spring Boot `spring.config.additional-location`
You can supply this external folder directly to any Spring Boot service without a config server:
```bash
java -jar build/libs/user-service-0.0.1-SNAPSHOT.jar \
  --spring.config.additional-location=file:../config/
```

### Option C: Kubernetes ConfigMaps
Create ConfigMaps from this directory:
```bash
# Shared configuration
kubectl create configmap common-config --from-file=application.yaml=config/application.yaml

# Service-specific configurations
kubectl create configmap user-service-config --from-file=user-service.yaml=config/user-service.yaml
kubectl create configmap project-service-config --from-file=project-service.yaml=config/project-service.yaml
kubectl create configmap payment-service-config --from-file=payment-service.yaml=config/payment-service.yaml
kubectl create configmap notification-service-config --from-file=notification-service.yaml=config/notification-service.yaml
kubectl create configmap aicode-service-config --from-env-file=config/aicode-service.env
kubectl create configmap provisioning-service-config --from-env-file=config/provisioning-service.env
```

### Option D: Python FastAPI Local Development
For Python services, copy the environment file into the service root:
```bash
cp config/aicode-service.env aicode-service/.env
cp config/provisioning-service.env provisioning-service/.env
```
