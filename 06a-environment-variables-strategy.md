# 6a. Environment Variables Strategy 🔧

[<- Back to Main Topic](./06-environment-configuration-management.md) | [Next Sub-Topic: Secrets Handling ->](./06b-secrets-handling.md)

## Overview

A well-designed environment variables strategy forms the backbone of configurable Rust applications in modern deployment pipelines. It allows applications to adjust behavior based on context without requiring code changes or rebuilds. The WhoKnows project implements a comprehensive environment variable strategy that spans from local development to production deployment, encompassing both the application code and the deployment infrastructure.

## Key Concepts

### Environment Variable Categories

Environment variables in the WhoKnows project fall into distinct categories:

1. **Runtime Configuration**: Direct application behavior (ports, logging levels)
2. **Connection Strings**: Database URLs, external service endpoints
3. **Secrets**: Authentication tokens, encryption keys
4. **Build Settings**: Compilation flags, feature toggles
5. **Service Discovery**: How components find each other

Each category requires specific handling strategies for management, security, and deployment.

### Variable Scoping

```
┌─────────────────────────────────────────┐
│ Global Environment Variables            │
│ (PROJECT_NAME, ENV, VERSION)            │
│  ┌─────────────────────────────────┐    │
│  │ Service-specific Variables      │    │
│  │ (BACKEND_PORT, DATABASE_URL)    │    │
│  │   ┌─────────────────────────┐   │    │
│  │   │ Feature-specific Vars   │   │    │
│  │   │ (AUTH_MODE, LOG_LEVEL)  │   │    │
│  │   └─────────────────────────┘   │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

## Implementation Patterns

### Pattern 1: Default with Override

The WhoKnows Rust backend implements a pattern of code defaults with environment overrides:

```rust
// From main.rs
// Default with override pattern
let port_str = env::var("BACKEND_INTERNAL_PORT").unwrap_or_else(|_| "8080".to_string());
let port = port_str
    .parse::<u16>()
    .expect("BACKEND_INTERNAL_PORT must be a valid port number");

// Log the resolved configuration
log::info!("Server starting at http://{}:{}", HOST_NAME, port);
```

**When to use this pattern:**
- When providing sensible defaults for development
- For non-sensitive configuration
- When gracefully handling missing configuration
- For parameters that have logical defaults

### Pattern 2: Required Variables

For critical configuration without sensible defaults, the pattern enforces presence:

```rust
// From main.rs - Required variable pattern
let database_url =
    env::var("DATABASE_URL").expect("DATABASE_URL must be set in environment or .env file");

// For sensitive data like session keys
let session_secret_key_hex =
    env::var("SESSION_SECRET_KEY").expect("SESSION_SECRET_KEY must be set...");
```

**When to use this pattern:**
- For essential configuration without reasonable defaults
- When missing configuration would cause security issues
- For credentials and connection strings
- When explicit failure is better than silent default

### Pattern 3: Conditional Configuration

Some variables enable or disable entire features based on their presence:

```rust
// Example of conditional feature based on environment
if let Ok(analytics_endpoint) = env::var("ANALYTICS_ENDPOINT") {
    log::info!("Analytics enabled, sending to {}", analytics_endpoint);
    // Initialize analytics with the provided endpoint
    init_analytics(analytics_endpoint);
} else {
    log::info!("Analytics disabled, no endpoint configured");
    // Skip analytics initialization
}
```

**When to use this pattern:**
- For optional features or integrations
- When entire components are environment-specific
- For gradual feature rollouts
- For environment-specific plugins or extensions

## Common Challenges and Solutions

### Challenge 1: Type Conversion

Environment variables are always strings, requiring safe conversion to appropriate types.

**Solution:**

```rust
// Safe parsing with explicit error handling
fn get_env_as_u16(key: &str, default: u16) -> u16 {
    match env::var(key) {
        Ok(value) => {
            match value.parse::<u16>() {
                Ok(parsed) => parsed,
                Err(e) => {
                    log::warn!("Failed to parse {} as u16: {}. Using default: {}", key, e, default);
                    default
                }
            }
        }
        Err(_) => {
            log::debug!("{} not set, using default: {}", key, default);
            default
        }
    }
}

// Usage
let backend_port = get_env_as_u16("BACKEND_INTERNAL_PORT", 8080);
```

### Challenge 2: Configuration Hierarchy

Managing multiple sources of environment variables with proper precedence.

**Solution:**

```rust
// Configuration hierarchy implementation
fn get_config_value(key: &str) -> Option<String> {
    // Check environment first (highest precedence)
    if let Ok(value) = env::var(key) {
        return Some(value);
    }
    
    // Check .env file next
    if let Some(dotenv_value) = dotenv::var(key).ok() {
        return Some(dotenv_value);
    }
    
    // Check configuration file
    if let Some(config_value) = read_from_config_file(key) {
        return Some(config_value);
    }
    
    // No value found
    None
}

// Usage with fallback
let log_level = get_config_value("RUST_LOG").unwrap_or_else(|| "info".to_string());
```

## Docker Compose Integration

Docker Compose provides a way to manage environment variables across services:

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

Key patterns in the Docker Compose configuration:

1. **Default Values**: Using `${VAR:-default}` syntax for fallbacks
2. **Environment Passing**: Passing variables from host to container
3. **Centralized Configuration**: Single source of truth for service configuration
4. **Variable Substitution**: Using variables within other variable definitions

## GitHub Actions Integration

The CI/CD pipeline manages environment variables throughout the deployment process:

```yaml
# From cd.dev.yml
env:
  # Static environment variables
  GHCR_REGISTRY: ghcr.io
  IMAGE_BASENAME: ${{ github.repository_owner }}/${{ github.event.repository.name }}
  
  # Dynamic variables from secrets
  GHCR_PAT_OR_TOKEN: ${{ secrets.DEV_GHCR_PAT_OR_TOKEN }}
  SERVER_USER: ${{ secrets.DEV_SERVER_USER }}
  
  # Paths and configurations
  ENV_FILE: .env.development
  DEPLOY_DIR: ./deployment/whoknows
```

This approach provides:

1. **Workflow-Level Variables**: Consistent values across all jobs
2. **Secret Integration**: Secure access to sensitive variables
3. **Dynamic Values**: Values based on repository context
4. **Path Configuration**: Standardized paths for files and directories

## Environment Variable Templating

The WhoKnows project uses the `.env.development` file as a template for deployment:

```bash
# .env.development
# --- Core Configuration ---
COMPOSE_PROJECT_NAME=whoknows
NODE_ENV=development
HOST_PORT_FRONTEND=8080

# --- Backend Configuration ---
BACKEND_INTERNAL_PORT=92
RUST_LOG=debug
DATABASE_URL=sqlite:/app/data/whoknows.db?mode=rwc
SQLX_OFFLINE=TRUE
SESSION_SECRET_KEY=b59a8ecb692e66a1de06afad00dffaa77a8a922c7b54275c2b88662419c04e68

# --- Frontend Configuration ---
FRONTEND_INTERNAL_PORT=91
```

This template approach provides:

1. **Documentation**: Self-documenting expected variables
2. **Defaults**: Reasonable values for local development
3. **Structure**: Organized by component and purpose
4. **Template**: Base for CI/CD to generate environment-specific values

## Practical Example

The complete workflow for environment variable handling in WhoKnows demonstrates the integration of these concepts:

1. **Local Development**: Developers use `.env` files for local testing
2. **CI/CD Pipeline**: GitHub Actions reads secrets and builds dynamic configuration
3. **Deployment**: Configuration transferred to deployment server
4. **Runtime**: Docker uses environment variables to configure containers
5. **Application**: Rust code reads environment variables for behavior control

This end-to-end approach ensures consistent, secure configuration across all environments.

## Summary

1. Environment variables provide a flexible mechanism for application configuration
2. Different categories of variables require specific handling strategies
3. Design patterns like defaults, required variables, and conditional configuration address various needs
4. Integration with Docker Compose and GitHub Actions creates a cohesive system
5. Templating approach provides documentation and consistency

## Next Steps

With a solid environment variables strategy in place, the next crucial aspect is secrets handling - ensuring that sensitive configuration values are managed securely throughout the development and deployment lifecycle.

---

[<- Back to Main Topic](./06-environment-configuration-management.md) | [Next Sub-Topic: Secrets Handling ->](./06b-secrets-handling.md)
