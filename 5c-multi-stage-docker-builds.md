# 5C. Multi-stage Docker Builds 🐳

[<- Back to Main Topic](./05-frontend-static-content.md) | [Next Sub-Topic: Development Workflow ->](./05d-development-workflow.md)

## Overview

Multi-stage Docker builds are essential for creating efficient, production-ready containers while maintaining a good development experience. The WhoKnows frontend implements a sophisticated multi-stage build process that separates build-time dependencies from runtime requirements, optimizing both container size and build performance. This approach is particularly valuable for Rust applications, where compile-time dependencies differ significantly from runtime needs.

## Key Concepts

### Multi-stage Build Pattern

Docker multi-stage builds use multiple `FROM` statements in a single Dockerfile, allowing artifacts to be passed between stages while discarding unnecessary components:

```dockerfile
# Stage 1: Build environment
FROM rust:1.81-slim AS builder
# Build commands...

# Stage 2: Runtime environment
FROM debian:bookworm-slim AS production
# Copy only what's needed from builder
COPY --from=builder /app/target/release/myapp .
```

This pattern enables:
- Smaller final images (by excluding build tools)
- Improved security (fewer packages = smaller attack surface)
- Better layer caching (separate build and runtime layers)
- Different optimizations for different environments

### Build Caching

The WhoKnows Dockerfile implements sophisticated caching strategies to speed up builds:

```dockerfile
# Copy only dependency specifications first
COPY Cargo.toml ./
COPY Cargo.lock ./

# Create empty src/main.rs to build dependencies
RUN mkdir -p src && echo "fn main() {}" > src/main.rs
RUN cargo build --release

# Now copy actual source code and rebuild
COPY src ./src
RUN touch src/main.rs && cargo build --release
```

This technique leverages Docker's layer caching to avoid recompiling dependencies when only source code changes.

## Implementation Patterns

### Pattern 1: Staged Dependency Building

```dockerfile
# From Dockerfile - Builder stage
FROM rust:1.81-slim AS builder

WORKDIR /usr/src/app

# Copy dependency manifests
COPY Cargo.toml ./
COPY Cargo.lock ./

# Trick to build dependencies separately
RUN mkdir -p src && echo "fn main() {}" > src/main.rs
RUN cargo build --release

# Now copy the actual source and rebuild
COPY src ./src
RUN touch src/main.rs && cargo build --release
```

**When to use this pattern:**
- For Rust applications with many dependencies
- When builds happen frequently (CI/CD pipelines)
- To reduce build times during development
- When dependencies change less often than source code

### Pattern 2: Development-Specific Stage

```dockerfile
# From Dockerfile - Development stage
FROM rust:1.81-slim AS dev
WORKDIR /usr/src/app

# Install native dependencies for development
RUN apt-get update && apt-get install -y \
  pkg-config \
  libssl-dev \
  && rm -rf /var/lib/apt/lists/*

# Copy everything (source, static, config)
COPY . .

# Install development tooling
RUN cargo install cargo-watch dotenv-cli cargo-make

# Expose port for dev server
EXPOSE 8080

# Default command: hot-reload dev server
CMD ["cargo", "make", "dev"]
```

**When to use this pattern:**
- For local development environments
- When you need hot-reloading or other dev tools
- To maintain consistency between local and CI environments
- When development workflow differs from production

### Pattern 3: Minimal Production Image

```dockerfile
# From Dockerfile - Production stage
FROM debian:bookworm-slim AS production

RUN apt-get update && apt-get install -y \
  ca-certificates \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY --from=builder /usr/src/app/target/release/frontend .

# Copy static files
COPY static ./static

# Expose the port the app runs on
EXPOSE 8080

# Command to run the application
CMD ["./frontend"]
```

**When to use this pattern:**
- For production deployments
- To minimize container size and attack surface
- When runtime dependencies differ from build dependencies
- For optimized startup and runtime performance

## Common Challenges and Solutions

### Challenge 1: Static Files Management

Static files need to be included in the production image without the build toolchain.

**Solution:**

```dockerfile
# From Dockerfile - Production stage
# Copy only the compiled binary from builder
COPY --from=builder /usr/src/app/target/release/frontend .

# Copy static files directly from build context
COPY static ./static
```

This approach:
- Keeps static files separate from the build process
- Ensures content is included in the final image
- Allows for selective copying of only needed files
- Maintains proper directory structure for server paths

### Challenge 2: Runtime Dependencies

Rust binaries often require system libraries at runtime.

**Solution:**

```dockerfile
# From Dockerfile - Production stage runtime dependencies
RUN apt-get update && apt-get install -y \
  ca-certificates \
  # Add any other runtime dependencies
  && rm -rf /var/lib/apt/lists/*
```

Key aspects:
- Includes only essential runtime packages
- Cleans up package lists to reduce image size
- Separates these from development dependencies
- Keeps the runtime environment minimal

## Practical Example

A complete multi-stage Dockerfile for the WhoKnows frontend:

```dockerfile
###############################
# Builder stage - compiles Rust application
###############################
FROM rust:1.81-slim AS builder

WORKDIR /usr/src/app

# Install build dependencies if needed
RUN apt-get update && apt-get install -y \
  pkg-config \
  libssl-dev \
  && rm -rf /var/lib/apt/lists/*

# Copy dependency manifests first for better caching
COPY Cargo.toml ./
COPY Cargo.lock ./

# Build dependencies separately
RUN mkdir -p src && echo "fn main() {}" > src/main.rs
RUN cargo build --release

# Copy source code and rebuild
COPY src ./src
RUN touch src/main.rs && cargo build --release

###############################
# Development stage - for local development with hot-reloading
###############################
FROM rust:1.81-slim AS dev

WORKDIR /usr/src/app

# Install development dependencies
RUN apt-get update && apt-get install -y \
  pkg-config \
  libssl-dev \
  && rm -rf /var/lib/apt/lists/*

# Install development tools
RUN cargo install cargo-watch dotenv-cli cargo-make

# Copy all project files
COPY . .

# Expose development port
EXPOSE 8080

# Command for development with hot-reloading
CMD ["cargo", "make", "dev"]

###############################
# Production stage - minimal runtime image
###############################
FROM debian:bookworm-slim AS production

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
  ca-certificates \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy compiled binary from builder
COPY --from=builder /usr/src/app/target/release/frontend .
RUN chmod +x ./frontend

# Copy static content
COPY static ./static

# Create version file for tracking deployments
ARG BUILD_VERSION=dev
RUN echo "${BUILD_VERSION}" > ./VERSION

# Expose application port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/api/health || exit 1

# Runtime command
CMD ["./frontend"]
```

## Docker Compose Integration

The multi-stage Dockerfile integrates with Docker Compose for different environments:

```yaml
# docker-compose.yml example
version: '3.8'

services:
  # Development environment
  frontend-dev:
    build:
      context: ./frontend
      target: dev
    volumes:
      - ./frontend:/usr/src/app
      - /usr/src/app/target
    ports:
      - "8080:8080"
    environment:
      - RUST_LOG=debug
      - FRONTEND_INTERNAL_PORT=8080
      - BACKEND_URL=http://backend:5050
    depends_on:
      - backend

  # Production environment
  frontend:
    build:
      context: ./frontend
      target: production
      args:
        - BUILD_VERSION=${BUILD_VERSION:-local}
    ports:
      - "${HOST_PORT_FRONTEND:-8080}:8080"
    environment:
      - RUST_LOG=info
      - FRONTEND_INTERNAL_PORT=8080
      - BACKEND_URL=http://backend:5050
    depends_on:
      - backend
```

This configuration allows for:
- Selecting the appropriate build stage for each environment
- Volume mounting for development hot-reloading
- Environment-specific variables
- Service dependencies

## Build Optimization Techniques

The WhoKnows frontend implements several techniques to optimize Docker builds:

1. **Dependency Caching**: Building dependencies separately from application code
2. **Layer Optimization**: Ordering commands from least to most frequently changed
3. **Multi-stage Approach**: Using separate stages for build and runtime
4. **Minimal Base Images**: Using slim variants to reduce size
5. **Cleanup**: Removing unnecessary files after installation

Example of layer optimization:

```dockerfile
# Install packages (changes infrequently)
RUN apt-get update && apt-get install -y \
  ca-certificates \
  && rm -rf /var/lib/apt/lists/*

# Copy binary (changes with each build)
COPY --from=builder /usr/src/app/target/release/frontend .

# Copy static files (changes frequently during development)
COPY static ./static
```

## Summary

1. Multi-stage Docker builds separate compilation from runtime environments
2. The builder stage compiles Rust code with all build dependencies
3. The development stage provides tools for local development and hot-reloading
4. The production stage contains only the compiled binary and static files
5. Caching strategies optimize build performance by leveraging Docker's layer system

## Next Steps

The next sub-topic explores the development workflow, including how to effectively use the multi-stage Docker builds for local development, testing, and deploying the frontend application.

---

[<- Back to Main Topic](./05-frontend-static-content.md) | [Next Sub-Topic: Development Workflow ->](./05d-development-workflow.md)
