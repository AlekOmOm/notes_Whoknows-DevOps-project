# 6. Environment Configuration Management 🔐

[<- Back: GitHub Actions Workflows](./05-github-actions-workflows.md) | [Next: Deployment Strategies ->](./07-deployment-strategies.md)

---
- [6a. Environment Variables Strategy](./06a-environment-variables-strategy.md) 
- [6b. Secrets Handling](./06b-secrets-handling.md) 
- [6c. Configuration Files](./06c-configuration-files.md) 
- [6d. Dynamic Configuration](./06d-dynamic-configuration.md) 
---

## Table of Contents

- [Introduction](#introduction)
- [Configuration Layers](#configuration-layers)
- [Environment Variable Management](#environment-variable-management)
- [Secrets Strategy](#secrets-strategy)
- [Configuration Injection](#configuration-injection)
- [Verification and Validation](#verification-and-validation)

## Introduction

Environment configuration management is a critical aspect of modern Rust application deployment. It addresses the challenge of configuring applications differently across various environments (development, staging, production) while maintaining security and consistency. In the WhoKnows project, a comprehensive configuration management strategy ensures that applications receive the right settings at runtime without exposing sensitive information.

This approach encompasses environment variables, configuration files, secrets management, and dynamic configuration injection through the CI/CD pipeline.

## Configuration Layers

The WhoKnows project implements a layered configuration approach:

1. **Code defaults**: Fallback values defined in the application code
2. **Environment variables**: Runtime configuration injected via container environment
3. **Configuration files**: Structured settings in `.env` files
4. **Secrets**: Sensitive values stored securely in GitHub Secrets
5. **Dynamic configuration**: Values generated during deployment (like image tags)

### Configuration Priority

```
┌───────────────────┐
│ Dynamic/Runtime   │ ← Highest priority
├───────────────────┤
│ GitHub Secrets    │
├───────────────────┤
│ .env files        │
├───────────────────┤
│ Docker Compose    │
├───────────────────┤
│ Code Defaults     │ ← Lowest priority
└───────────────────┘
```

## Environment Variable Management

Environment variables provide a flexible, platform-agnostic way to configure applications.

### Key Patterns

```rust
// From main.rs - Environment variable reading with fallback
let port_str = env::var("BACKEND_INTERNAL_PORT").unwrap_or_else(|_| "8080".to_string());
let port = port_str
    .parse::<u16>()
    .expect("BACKEND_INTERNAL_PORT must be a valid port number");

// Required environment variable (fails if missing)
let database_url =
    env::var("DATABASE_URL").expect("DATABASE_URL must be set in environment or .env file");

// Loading dotenv file for local development
dotenv::dotenv().ok();
```

### Configuration Files

The WhoKnows project uses `.env` files for environment-specific configuration:

```bash
# .env.development
COMPOSE_PROJECT_NAME=whoknows
NODE_ENV=development
HOST_PORT_FRONTEND=8080

# Backend Configuration
BACKEND_INTERNAL_PORT=92
RUST_LOG=debug
DATABASE_URL=sqlite:/app/data/whoknows.db?mode=rwc
SQLX_OFFLINE=TRUE
SESSION_SECRET_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Frontend Configuration
FRONTEND_INTERNAL_PORT=91

# Deployment Variables (dynamic)
IMAGE_TAG_BACKEND=ghcr.io/debugger-demons/whoknows/backend:latest
IMAGE_TAG_FRONTEND=ghcr.io/debugger-demons/whoknows/frontend:latest
```

These files serve different purposes:

1. **Local Development**: Direct developer use for local environments
2. **Template for Deployment**: Base for CI/CD to generate runtime configuration
3. **Documentation**: Self-documenting the expected configuration parameters

## Secrets Strategy

Sensitive information requires special handling to maintain security.

### GitHub Secrets Integration

The WhoKnows project stores sensitive information in GitHub Secrets:

1. **SSH Keys**: For secure server access
2. **API Tokens**: For registry authentication
3. **Database Credentials**: For database access
4. **Session Keys**: For secure sessions
5. **Environment Files**: Complete environment files with sensitive data

```yaml
# Example from GitHub Actions workflow
env:
  # Environment Variables
  GHCR_REGISTRY: ghcr.io
  IMAGE_BASENAME: ${{ github.repository_owner }}/${{ github.event.repository.name }}
  # Secrets:
  GHCR_PAT_OR_TOKEN: ${{ secrets.DEV_GHCR_PAT_OR_TOKEN }}
  SERVER_USER: ${{ secrets.DEV_SERVER_USER }} 
  SERVER_HOST: ${{ secrets.DEV_SERVER_HOST }}
  SERVER_PORT: ${{ secrets.DEV_SERVER_PORT }}
  DATABASE_URL: ${{ secrets.DB_PATH}}
  # SSH:
  SSH_PRIVATE_KEY: ${{ secrets.DEV_SSH_PRIVATE_KEY }}
  ENV_FILE_CONTENT: ${{ secrets.DEV_ENV_FILE }}
```

## Configuration Injection

The CI/CD pipeline dynamically injects configuration into the deployment environment.

### Configuration Flow

```
┌─────────────┐        ┌─────────────┐        ┌─────────────┐
│   GitHub    │        │  Workflow   │        │  Docker     │
│   Secrets   │───────►│  Generated  │───────►│  Container  │
│             │        │  .env File  │        │  Environment│
└─────────────┘        └─────────────┘        └─────────────┘
                              ▲
                              │
                      ┌───────┴───────┐
                      │    Dynamic    │
                      │Configuration  │
                      │(Image Tags)   │
                      └───────────────┘
```

### Implementation Example

```yaml
# From cd.dev.yml - Creating .env file
- name: Create .env file
  run: |
    echo "${{ env.ENV_FILE_CONTENT }}" > ${{ env.ENV_FILE }}
    dos2unix ${{ env.ENV_FILE }}
    
    # Add image tags to .env file
    echo "IMAGE_TAG_BACKEND=${{ needs.build-push.outputs.backend_image_sha }}" >> ${{ env.ENV_FILE }}
    echo "IMAGE_TAG_FRONTEND=${{ needs.build-push.outputs.frontend_image_sha }}" >> ${{ env.ENV_FILE }}
```

This approach combines static configuration (from secrets) with dynamic values generated during the pipeline execution.

## Docker Compose Integration

Docker Compose serves as a configuration aggregator, passing environment variables to containers:

```yaml
# From docker-compose.dev.yml
services:
  backend:
    container_name: ${COMPOSE_PROJECT_NAME:-mywebapp}_backend_dev
    image: ${IMAGE_TAG_BACKEND}
    environment:
      - COMPOSE_PROJECT_NAME=${COMPOSE_PROJECT_NAME}
      - BACKEND_INTERNAL_PORT=${BACKEND_INTERNAL_PORT}
      - RUST_LOG=${RUST_LOG}
      - DATABASE_URL=${DATABASE_URL}
      - SESSION_SECRET_KEY=${SESSION_SECRET_KEY}
```

Key patterns in this integration:

1. **Default Values**: Using `${VAR:-default}` syntax
2. **Variable Reuse**: Using the same variable in multiple contexts
3. **Isolation**: Each service receives only its required variables
4. **Consistency**: Same variables used in multiple files

## Verification and Validation

Ensuring configuration correctness is critical for reliable deployments.

### Validation Strategies

1. **Early Validation**: Checking for required secrets before deployment starts
2. **Application Validation**: Verifying environment at runtime
3. **Explicit Failures**: Clear error messages for missing configuration
4. **Version Tracking**: VERSION file with deployment details

```yaml
# From cd.dev.yml - Validation workflow
validate-config:
  uses: ./.github/workflows/validate.env_and_secrets.yml
  with:
    environment: development
  secrets:
    DEV_ENV_FILE: ${{ secrets.DEV_ENV_FILE }}
    DEV_SSH_PRIVATE_KEY: ${{ secrets.DEV_SSH_PRIVATE_KEY }}
    DEV_GHCR_PAT_OR_TOKEN: ${{ secrets.DEV_GHCR_PAT_OR_TOKEN }}
```

## Best Practices Implemented

The WhoKnows project adheres to several configuration management best practices:

1. **Environment Abstraction**: Code should never know which environment it's running in
2. **Secrets Isolation**: Sensitive information never stored in code
3. **Validation First**: Fail early if configuration is incomplete
4. **Audit Trail**: Track which configuration was deployed when
5. **Configuration as Code**: All configuration patterns are version controlled
6. **Defaults Where Appropriate**: Sensible defaults for non-critical settings

## Example: Session Key Management

The handling of the session secret key demonstrates a complete configuration flow:

1. **Value Storage**: Secret value stored in GitHub Secrets
2. **Injection**: Value included in `.env` file during deployment
3. **Container Access**: Value passed to container via environment variables
4. **Application Usage**: Application reads and validates the value

```rust
// From main.rs - Session key handling
let session_secret_key_hex =
    env::var("SESSION_SECRET_KEY").expect("SESSION_SECRET_KEY must be set...");

let decoded_key_bytes = match hex::decode(&session_secret_key_hex) {
    Ok(bytes) => bytes,
    Err(e) => {
        panic!("SESSION_SECRET_KEY decoding failed: {:?}", e);
    }
};

let session_secret_key = Key::derive_from(&key_array);
```

This pattern ensures security while maintaining flexibility across environments.

---

[<- Back: GitHub Actions Workflows](./05-github-actions-workflows.md) | [Next: Deployment Strategies ->](./07-deployment-strategies.md)
