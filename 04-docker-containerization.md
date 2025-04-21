# 4. Docker & Containerization 🐳

[<- Back: CI/CD](./03-ci-cd.md) | [Next: GitHub Actions Workflows ->](./05-github-actions-workflows.md)

---
- [4a. Dockerfile Best Practices](./04a-dockerfile-best-practices.md) 
- [4b. Multi-stage Builds](./04b-multi-stage-builds.md) 
- [4c. Image Optimization](./04c-image-optimization.md) 
- [4d. Container Registry Management](./04d-container-registry-management.md) 
---

## Table of Contents

- [Introduction](#introduction)
- [Docker in Rust Development](#docker-in-rust-development)
- [Container Architecture](#container-architecture)
- [Image Building Strategy](#image-building-strategy)
- [Orchestration with Docker Compose](#orchestration-with-docker-compose)
- [Security Considerations](#security-considerations)

## Introduction

Containerization, primarily through Docker, has become a cornerstone of modern application deployment. For Rust applications like WhoKnows, containers provide consistent, isolated runtime environments that enable seamless transitions from development to production. This approach eliminates "it works on my machine" problems and supports efficient scaling and deployment.

The WhoKnows project leverages Docker containers for both backend and frontend components, with careful attention to build efficiency, security, and runtime performance.

## Docker in Rust Development

Rust applications benefit from containerization in several ways:

1. **Consistent Compilation Environment**: Rust's strict compiler behavior is identical across environments
2. **Dependency Management**: All system-level dependencies are captured in the container
3. **Optimized Binaries**: Rust's static linking works well with minimal container images
4. **Cross-Platform Deployment**: Build once, deploy anywhere functionality

### Rust Container Considerations

```rust
// Example of Rust code that interacts with environment-specific paths
fn get_data_directory() -> std::path::PathBuf {
    std::env::var("DATA_DIR")
        .map(std::path::PathBuf::from)
        .unwrap_or_else(|_| std::path::PathBuf::from("/app/data"))
}
```

## Container Architecture

The WhoKnows project implements a multi-container architecture:

1. **Backend Container**: Rust ActixWeb application
2. **Frontend Container**: Web frontend interface
3. **Database**: Persisted SQLite data (via volume mount)

This separation of concerns allows each component to be developed, tested, and deployed independently.

### Container Communication

```
┌────────────┐      ┌────────────┐      ┌────────────┐
│            │      │            │      │            │
│  Frontend  │◄────►│  Backend   │◄────►│  Database  │
│            │      │            │      │            │
└────────────┘      └────────────┘      └────────────┘
     Port 91             Port 92           Mounted
                                          Volume
```

## Image Building Strategy

The WhoKnows project uses multi-stage Docker builds to create efficient, secure container images.

### Backend Dockerfile Analysis

```dockerfile
# Stage 1: Builder
FROM rust:1.81 AS builder

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends libssl-dev pkg-config

WORKDIR /app

# Copy manifests for better caching
COPY Cargo.toml ./
COPY Cargo.lock ./
RUN if [ ! -f Cargo.lock ]; then cargo generate-lockfile; fi

# Copy source and build
COPY src ./src
COPY .sqlx ./.sqlx

RUN cargo build

# Stage 2: Runtime
FROM debian:bookworm-slim AS production

RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY --from=builder /app/target/debug/backend .

RUN chmod +x ./backend

EXPOSE ${BACKEND_INTERNAL_PORT}

CMD ["./backend"]
```

Key strategies in this Dockerfile:

1. **Multi-stage build**: Separate build environment from runtime environment
2. **Dependency caching**: Copy manifests first to leverage Docker layer caching
3. **Minimal runtime image**: Use slim base image with only required packages
4. **Security**: Clean up package lists, minimal permissions
5. **Configuration via environment**: Use environment variables for configuration

## Orchestration with Docker Compose

Docker Compose provides service orchestration for local development and production deployment.

### Compose Configuration

```yaml
# Simplified docker-compose.dev.yml
services:
  backend:
    container_name: ${COMPOSE_PROJECT_NAME:-mywebapp}_backend_dev
    image: ${IMAGE_TAG_BACKEND}
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - SESSION_SECRET_KEY=${SESSION_SECRET_KEY}
    volumes:
      - /home/deployer/deployment/app/data:/app/data
    expose:
      - "${BACKEND_INTERNAL_PORT}"
    networks:
      - app-network

  frontend:
    container_name: ${COMPOSE_PROJECT_NAME:-mywebapp}_frontend_dev
    image: ${IMAGE_TAG_FRONTEND}
    ports:
      - "${HOST_PORT_FRONTEND:-8080}:${FRONTEND_INTERNAL_PORT:-91}"
    environment:
      - BACKEND_INTERNAL_PORT=${BACKEND_INTERNAL_PORT:-92}
    networks:
      - app-network
    depends_on:
      - backend

networks:
  app-network:
    name: ${COMPOSE_PROJECT_NAME}_network_dev
    driver: bridge
```

Key aspects of this configuration:

1. **Service Naming**: Dynamic container names based on environment variables
2. **Image Selection**: Variable-based image tags for different environments
3. **Configuration Injection**: Environment variables passed to containers
4. **Volume Mounting**: Persistent data storage across container restarts
5. **Network Definition**: Isolated network for inter-service communication
6. **Dependency Management**: Services start in the correct order

## Security Considerations

Container security is paramount in modern deployments. The WhoKnows project implements several security best practices:

1. **Minimal Base Images**: Using slim variants to reduce attack surface
2. **Non-Root Users**: Running applications with least privilege (when possible)
3. **Dependency Scanning**: Checking for known vulnerabilities
4. **Secret Management**: Keeping sensitive data out of images
5. **Image Signing**: Ensuring image integrity

### Security Implementation Example

```yaml
# Example security-focused Docker Compose configuration
services:
  backend:
    # Other configuration...
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
    volumes:
      - /home/deployer/deployment/app/data:/app/data:rw
```

## Practical Implementation

The complete containerization approach for WhoKnows demonstrates several important patterns:

1. **Environment Abstraction**: Applications read configuration from environment variables
2. **Artifact Immutability**: Images are tagged with Git SHAs for immutable deployments
3. **Data Persistence**: Volume mounts for database and other stateful components
4. **Network Isolation**: Services communicate over defined networks
5. **Build Optimization**: Multi-stage builds and layer caching for efficiency

```rust
// Example of environment-aware Rust code (from main.rs)
let port_str = env::var("BACKEND_INTERNAL_PORT").unwrap_or_else(|_| "8080".to_string());
let port = port_str
    .parse::<u16>()
    .expect("BACKEND_INTERNAL_PORT must be a valid port number");

let database_url =
    env::var("DATABASE_URL").expect("DATABASE_URL must be set in environment or .env file");
```

---

[<- Back: CI/CD](./03-ci-cd.md) | [Next: GitHub Actions Workflows ->](./05-github-actions-workflows.md)
