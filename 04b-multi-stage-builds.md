# 4b. Multi-stage Builds 🔍

[<- Back to Main Topic](./04-docker-containerization.md) | [Next Sub-Topic: Image Optimization ->](./04c-image-optimization.md)

## Overview

Multi-stage builds represent a powerful Docker pattern that addresses the challenge of creating minimal production images while maintaining a comprehensive build environment. For Rust applications, where compilation requires numerous build tools and dependencies not needed at runtime, this approach is particularly valuable. The WhoKnows project implements multi-stage builds to create efficient, secure containers for both backend and frontend components.

## Key Concepts

### The Problem of Single-Stage Builds

Traditional Docker builds often suffer from several issues:

1. **Bloated images**: All build tools and intermediary files remain in the final image
2. **Security vulnerabilities**: Unnecessary tools increase the attack surface
3. **Large image sizes**: Longer transfer times and increased storage costs
4. **Inefficient caching**: Changes in source code invalidate caching for dependencies

### Multi-stage Build Structure

Multi-stage builds separate the build process into distinct stages:

1. **Builder stage**: Contains all necessary build tools and dependencies
2. **Runtime stage**: Contains only the compiled application and runtime dependencies

```dockerfile
# Stage 1: Builder
FROM rust:1.81 AS builder
# Build tools and compilation here...

# Stage 2: Runtime
FROM debian:bookworm-slim AS production
# Copy only the executable and runtime dependencies
COPY --from=builder /app/target/debug/backend .
```

### Layer Caching Strategy

Effective layer caching is crucial for efficient builds:

1. **Dependency layers**: Should change infrequently
2. **Source code layers**: Change more frequently
3. **Build layers**: Rebuild only when dependencies or source change

## Implementation Patterns

### Pattern 1: Dependency-First Copying

In the WhoKnows Rust backend Dockerfile, the pattern of copying dependencies before source code optimizes Docker layer caching:

```dockerfile
# Copy manifests for better caching
COPY Cargo.toml ./
COPY Cargo.lock ./
RUN if [ ! -f Cargo.lock ]; then cargo generate-lockfile; fi

# Copy source and build
COPY src ./src
COPY .sqlx ./.sqlx

RUN cargo build
```

**When to use this pattern:**
- When dependencies change less frequently than source code
- When build times are significant
- When CI/CD pipelines need optimization
- When iterative development requires frequent rebuilds

### Pattern 2: Staged Dependency Installation

For complex build environments, installing system dependencies in stages improves readability and maintainability:

```dockerfile
# Builder stage with dependencies
FROM rust:1.81 AS builder

# Install essential build dependencies first
RUN apt-get update && apt-get install -y --no-install-recommends \
    libssl-dev \
    pkg-config

# Install optional tools for development/debugging if needed
RUN if [ "$BUILD_ENV" = "development" ]; then \
    apt-get install -y --no-install-recommends \
    valgrind \
    gdb; \
fi

WORKDIR /app
# Rest of build process...
```

**When to use this pattern:**
- When build dependencies are complex or numerous
- When conditional tooling is needed based on build type
- When documenting the purpose of each dependency is important
- When dependencies might change independently

## Common Challenges and Solutions

### Challenge 1: Build Artifacts Location

Ensuring the runtime stage can find the correct build artifacts from the builder stage.

**Solution:**

```dockerfile
# In builder stage
RUN cargo build --release
# Explicitly show where the artifact will be
RUN ls -la /app/target/release/

# In runtime stage
WORKDIR /app
# Be explicit about the source and destination paths
COPY --from=builder /app/target/release/backend /app/backend
```

### Challenge 2: Optimizing for Development vs. Production

Balancing different needs for development and production builds.

**Solution:**

```dockerfile
# Use build arguments to customize the build
ARG BUILD_ENV=production

# Builder stage
FROM rust:1.81 AS builder
ARG BUILD_ENV
RUN echo "Building for ${BUILD_ENV} environment"

# Conditional build command based on environment
RUN if [ "$BUILD_ENV" = "production" ]; then \
    cargo build --release; \
else \
    cargo build; \
fi

# Runtime stage selection based on environment
FROM debian:bookworm-slim AS production
FROM debian:bookworm AS development

# Use the appropriate base image
FROM ${BUILD_ENV}
# Continue with runtime setup...
```

## Practical Example

Let's analyze the WhoKnows backend Dockerfile in detail:

```dockerfile
# Stage 1: Builder - using full Rust image with all build tools
FROM rust:1.81 AS builder

# Install build dependencies - only what's needed for compilation
RUN apt-get update && apt-get install -y --no-install-recommends libssl-dev pkg-config

WORKDIR /app

# Step 1: Copy only the manifest files for dependency resolution
# This creates a cacheable layer that only changes when dependencies change
COPY Cargo.toml ./
COPY Cargo.lock ./
RUN if [ ! -f Cargo.lock ]; then cargo generate-lockfile; fi

# Step 2: Copy application source code
# This layer changes when source code changes but doesn't invalidate dependency caching
COPY src ./src
COPY .sqlx ./.sqlx

# Step 3: Build the application
# Debug build for faster compilation during development
RUN cargo build

# Stage 2: Runtime - using minimal Debian image
FROM debian:bookworm-slim AS production

# Install only the absolute minimum required runtime dependencies
# Clean up in the same layer to keep the image small
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy ONLY the compiled binary from the builder stage
# Nothing else from the build environment is included
COPY --from=builder /app/target/debug/backend .

# Set proper permissions
RUN chmod +x ./backend

# Configure container runtime
EXPOSE ${BACKEND_INTERNAL_PORT}

# Define the command to run
CMD ["./backend"]
```

### Key Benefits in this Implementation

1. **Separation of Concerns**: 
   - Build environment contains all tools and source code
   - Runtime environment contains only the executable

2. **Image Size Optimization**:
   - Runtime image excludes Rust compiler, build tools, source code
   - Only necessary runtime dependencies are installed

3. **Security Improvement**:
   - Reduced attack surface in production image
   - No build tools available in production container

4. **Build Cache Efficiency**:
   - Dependencies layer changes only when Cargo files change
   - Source code changes don't invalidate dependency caching

5. **Clean Runtime Environment**:
   - No build artifacts or temporary files in final image
   - Proper permissions and configuration

## Performance Comparison

| Build Approach | Image Size | Build Time | Security Profile |
|----------------|------------|------------|------------------|
| Single-stage (Rust image) | ~1.2GB | Baseline | High risk (includes compiler, build tools) |
| Multi-stage (Debug) | ~200MB | Similar to baseline | Improved (runtime dependencies only) |
| Multi-stage (Release) | ~80MB | 2-3x longer | Best (optimized binary, minimal dependencies) |

The significant size reduction combined with improved security posture makes multi-stage builds the preferred approach for Rust container deployments.

## Summary

1. Multi-stage builds separate compilation from runtime environments
2. Strategic file copying optimizes build caching
3. Final images are significantly smaller and more secure
4. Build artifacts are precisely controlled between stages
5. Environment-specific optimizations can be incorporated

## Next Steps

With multi-stage builds providing efficient container images, the next logical step is exploring more specific image optimization techniques that can further reduce size, improve security, and enhance performance for Rust applications.

---

[<- Back to Main Topic](./04-docker-containerization.md) | [Next Sub-Topic: Image Optimization ->](./04c-image-optimization.md)
