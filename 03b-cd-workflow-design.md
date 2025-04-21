# 3b. CD Workflow Design 🔄

[<- Back to Main Topic](./03-ci-cd.md) | [Next Sub-Topic: Environment-Specific Deployments ->](./03c-environment-specific-deployments.md)

## Overview

Continuous Deployment (CD) workflow design is a critical element of modern Rust application delivery, establishing the automated processes that move code from repository to production environments. The WhoKnows project implements a sophisticated CD workflow using GitHub Actions, container registries, and scripted deployment mechanisms that ensure reliable, consistent deployments while maintaining security and configuration integrity.

## Key Concepts

### Workflow Triggers

CD workflows are triggered by specific Git events that signify code is ready for deployment.

```yaml
# From cd.dev.yml - Workflow trigger definition
on:
  pull_request:
    types: [closed] # on PR closed (i.e. merged)
    branches: [development]

jobs:
  deploy:
    if: github.event.pull_request.merged == true
    # Job definition...
```

The key elements in this trigger configuration:
- **Event type**: Pull request closing event
- **Branch filter**: Only applies to the `development` branch
- **Conditional execution**: Only proceeds if the PR was actually merged (`merged == true`)

This approach ensures deployments only occur when code has been properly reviewed and accepted through the PR process, not when PRs are closed without merging.

### Job Organization

CD workflows are organized into logical jobs with clear dependencies:

1. **Pre-Deployment Validation**: Verifying configuration and prerequisites
2. **Build & Publish**: Creating container images and publishing to registry
3. **Deployment**: Executing the actual deployment operations

```yaml
jobs:
  validate-config:
    uses: ./.github/workflows/validate.env_and_secrets.yml
    with:
      environment: development
    secrets:
      DEV_ENV_FILE: ${{ secrets.DEV_ENV_FILE }}
      DEV_SSH_PRIVATE_KEY: ${{ secrets.DEV_SSH_PRIVATE_KEY }}
      DEV_GHCR_PAT_OR_TOKEN: ${{ secrets.DEV_GHCR_PAT_OR_TOKEN }}

  build-push:
    name: Build & Push Docker Images
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      backend_image_sha: ${{ env.GHCR_REGISTRY }}/${{ steps.lowercaser.outputs.image_base }}/backend:${{ steps.image_tags.outputs.tag_sha }}
      frontend_image_sha: ${{ env.GHCR_REGISTRY }}/${{ steps.lowercaser.outputs.image_base }}/frontend:${{ steps.image_tags.outputs.tag_sha }}
    steps:
      # Build and push steps...

  deploy:
    name: Deploy to Server
    if: github.event.pull_request.merged == true
    needs: build-push
    runs-on: ubuntu-latest
    steps:
      # Deployment steps...
```

## Implementation Patterns

### Pattern 1: Immutable Artifacts with Deterministic Tagging

The WhoKnows CD workflow creates immutable container images with deterministic tags based on the Git commit SHA:

```yaml
# Example from build-push job
- name: Define Image Tags
  id: image_tags
  run: |
    TAG_SHA=$(echo ${{ github.sha }} | cut -c1-7)
    echo "tag_sha=${TAG_SHA}" >> $GITHUB_OUTPUT
    echo "tag_latest=latest" >> $GITHUB_OUTPUT

- name: Build and Push Backend Image
  uses: docker/build-push-action@v5
  with:
    context: ${{ env.BACKEND_PATH }}
    file: ${{ env.BACKEND_PATH }}/Dockerfile
    push: true
    tags: |
      ${{ env.GHCR_REGISTRY }}/${{ steps.lowercaser.outputs.image_base }}/backend:${{ steps.image_tags.outputs.tag_latest }}
      ${{ env.GHCR_REGISTRY }}/${{ steps.lowercaser.outputs.image_base }}/backend:${{ steps.image_tags.outputs.tag_sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

**When to use this pattern:**
- When you need to maintain traceability between Git commits and deployed artifacts
- When rollbacks might be necessary
- When multiple environments might use different versions simultaneously
- When you want to ensure the exact same artifact moves through environments

### Pattern 2: Dynamic Environment Configuration

The CD workflow dynamically creates environment-specific configuration by combining static secrets with runtime information:

```yaml
# Example from deploy job
- name: Create .env file
  run: |
    echo "${{ env.ENV_FILE_CONTENT }}" > ${{ env.ENV_FILE }}
    dos2unix ${{ env.ENV_FILE }}
    
    # Add image tags to .env file
    echo "IMAGE_TAG_BACKEND=${{ needs.build-push.outputs.backend_image_sha }}" >> ${{ env.ENV_FILE }}
    echo "IMAGE_TAG_FRONTEND=${{ needs.build-push.outputs.frontend_image_sha }}" >> ${{ env.ENV_FILE }}
```

The `.env.development` file contains the base configuration:

```bash
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

# --- Deployment Variables ---
# These are dynamically added by the CD pipeline
IMAGE_TAG_BACKEND=...
IMAGE_TAG_FRONTEND=...
```

**When to use this pattern:**
- When deploying configurations that combine static and dynamic elements
- When configuration should reference the specific artifacts being deployed
- When maintaining sensitive information securely in GitHub Secrets
- When environment files are used by Docker Compose or application configuration

## Common Challenges and Solutions

### Challenge 1: Secure Credential Management

Securely handling credentials for deployment servers and registries.

**Solution:**

```yaml
# Secure SSH configuration for deployment
- name: Set up SSH key
  run: |
    mkdir -p ~/.ssh
    echo "${{ env.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
    chmod 600 ~/.ssh/id_rsa
    ssh-keyscan -p ${{ env.SERVER_PORT }} -H ${{ env.SERVER_HOST }} >> ~/.ssh/known_hosts

# Secure Docker registry authentication
- name: Log in to GitHub Container Registry
  uses: docker/login-action@v3
  with:
    registry: ${{ env.GHCR_REGISTRY }}
    username: ${{ github.actor }}
    password: ${{ env.GHCR_PAT_OR_TOKEN }}
```

### Challenge 2: File Transfer and Remote Execution

Transferring configuration and deployment scripts securely to the deployment server.

**Solution:**

```yaml
- name: Transfer files to server
  run: |
    SERVER_DEST_BASE="${{ env.SERVER_USER }}@${{ env.SERVER_HOST }}:${{ env.DEPLOY_DIR }}"

    # mkdir
    ${{ env.SSH_CMD }} "mkdir -p ${{ env.DEPLOY_DIR }}" 

    # scp directly to final names
    ${{ env.SCP_CMD }} ${{ env.DOCKER_COMPOSE_FILE }} ${SERVER_DEST_BASE}/docker-compose.yml
    ${{ env.SCP_CMD }} ${{ env.ENV_FILE }} ${SERVER_DEST_BASE}/.env
    ${{ env.SCP_CMD }} ${{ env.NGINX_FILE_PATH }} ${SERVER_DEST_BASE}/nginx.conf
    ${{ env.SCP_CMD }} ${{ env.DEPLOY_SCRIPT }} ${SERVER_DEST_BASE}/deploy.sh
    ${{ env.SCP_CMD }} ${{ env.DOCKER_LOGIN_SCRIPT }} ${SERVER_DEST_BASE}/docker-login.sh
```

## Practical Example

The complete CD workflow for the WhoKnows project demonstrates a well-architected deployment pipeline that connects multiple components:

### Workflow Overview

1. **Trigger**: PR is merged to development branch
2. **Validation**: Ensure all secrets and configurations are available
3. **Build Backend**: Compile Rust code and create Docker image
4. **Build Frontend**: Build frontend and create Docker image
5. **Push Images**: Push both images to GitHub Container Registry with SHA tags
6. **Prepare Config**: Create deployment-specific .env file
7. **Transfer Files**: Copy all necessary files to deployment server
8. **Execute Deployment**: Run scripts on server to update containers

### Deployment Configuration Integration

A key aspect of the workflow is how the Docker Compose file references environment variables that are dynamically set in the deployment process:

```yaml
# From docker-compose.dev.yml
services:
  backend:
    container_name: ${COMPOSE_PROJECT_NAME:-mywebapp}_backend_dev
    image: ${IMAGE_TAG_BACKEND}
    build:
      context: ./backend
      args:
        - APP_NAME=whoknows_dev
        - RUST_LOG=${RUST_LOG}
        - COMPOSE_PROJECT_NAME=${COMPOSE_PROJECT_NAME}
        - BACKEND_INTERNAL_PORT=${BACKEND_INTERNAL_PORT}
    environment:
      - COMPOSE_PROJECT_NAME=${COMPOSE_PROJECT_NAME}
      - BACKEND_INTERNAL_PORT=${BACKEND_INTERNAL_PORT}
      - RUST_LOG=${RUST_LOG}
      - DATABASE_URL=${DATABASE_URL}
      - SESSION_SECRET_KEY=${SESSION_SECRET_KEY}
    volumes:
      - /home/deployer/deployment/app/data:/app/data
```

These variables are then used by the Rust application to configure its behavior:

```rust
// From main.rs
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv::dotenv().ok();

    // Get port from environment variable
    let port_str = env::var("BACKEND_INTERNAL_PORT").unwrap_or_else(|_| "8080".to_string());
    let port = port_str
        .parse::<u16>()
        .expect("BACKEND_INTERNAL_PORT must be a valid port number");
    
    // Get database URL from environment variable
    let database_url =
        env::var("DATABASE_URL").expect("DATABASE_URL must be set in environment or .env file");
        
    // Get session key from environment variable
    let session_secret_key_hex =
        env::var("SESSION_SECRET_KEY").expect("SESSION_SECRET_KEY must be set...");
        
    // ...
}
```

### Deployment Flow Diagram

```
┌─────────────┐         ┌───────────────┐         ┌───────────────┐
│  PR Merged  │         │ GitHub Secrets │         │ GitHub Cache  │
└──────┬──────┘         └───────┬───────┘         └───────┬───────┘
       │                        │                         │
       ▼                        ▼                         ▼
┌─────────────┐         ┌───────────────┐         ┌───────────────┐
│  Validate   │         │  Build Images  │◄────────│Docker Buildx  │
│   Config    │         │   with SHA     │         │    Cache      │
└──────┬──────┘         └───────┬───────┘         └───────────────┘
       │                        │                         
       ▼                        ▼                         
┌─────────────┐         ┌───────────────┐         ┌───────────────┐
│ Prepare ENV │◄────────│   Push to     │         │   Version     │
│    File     │         │     GHCR      │         │    File       │
└──────┬──────┘         └───────────────┘         └───────┬───────┘
       │                                                  │
       ▼                                                  ▼
┌─────────────┐         ┌───────────────┐         ┌───────────────┐
│  Transfer   │────────►│  Deployment   │◄────────│  Environment  │
│ Files to VM │         │    Server     │         │ Configuration │
└─────────────┘         └───────┬───────┘         └───────────────┘
                                │
                                ▼
                        ┌───────────────┐
                        │  Docker Pull  │
                        │  & Restart    │
                        └───────────────┘
```

## Summary

1. Well-designed CD workflows create a reliable, automated pipeline for Rust application deployment
2. Deterministic artifact tagging with Git SHAs ensures traceability and reproducible deployments
3. Dynamic environment configuration combines static secrets with runtime values
4. Secure credential handling is critical for both registry access and server operations
5. Clear job organization with proper dependencies ensures workflows execute correctly

## Next Steps

With an understanding of CD workflow design, the next step is exploring environment-specific deployments, which focuses on how to adapt deployment processes for different environments (development, staging, production) while maintaining consistency and security.

---

[<- Back to Main Topic](./03-ci-cd.md) | [Next Sub-Topic: Environment-Specific Deployments ->](./03c-environment-specific-deployments.md)
