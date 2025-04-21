# 5. Frontend Static Content with Actix Web 🌐

[<- Back: Docker & Containerization](./04-docker-containerization.md) | [Next: Environment Configuration Management ->](./06-environment-configuration-management.md)

---
- [5a. Actix Web for Static Content](./05a-actix-web-static-content.md) 
- [5b. Project Structure](./05b-project-structure.md) 
- [5c. Multi-stage Docker Builds](./05c-multi-stage-docker-builds.md) 
- [5d. Development Workflow](./05d-development-workflow.md) 
---

## Table of Contents

- [Introduction](#introduction)
- [Architecture Overview](#architecture-overview)
- [Actix Web Implementation](#actix-web-implementation)
- [Static Content Organization](#static-content-organization)
- [Containerization Strategy](#containerization-strategy)
- [Development Patterns](#development-patterns)

## Introduction

Modern web applications often separate frontend and backend concerns. The WhoKnows project implements a lightweight approach using Rust's Actix Web framework to serve static content (HTML, CSS, JavaScript) while delegating API calls to a separate backend service. This strategy offers simplicity, clear separation of concerns, and leverages client-side rendering without introducing complex frontend build toolchains.

This section explores the implementation details, architectural choices, and workflow patterns for developing and deploying the frontend component.

## Architecture Overview

The frontend architecture follows a minimalist approach:

1. **Actix Web Server**: Serving static files and providing health check endpoints
2. **Client-side JavaScript**: Handling templating, API calls, and user interactions
3. **Static Assets**: Organized HTML, CSS, and images
4. **No Server-side Rendering**: Clean separation between content delivery and business logic

### Component Interaction

```
┌─────────────┐      REST API      ┌─────────────┐
│             │◄──────────────────►│             │
│  Frontend   │                    │  Backend    │
│ Actix Web   │                    │ Actix Web   │
│             │                    │             │
└─────────────┘                    └─────────────┘
      │                                   │
      │                                   │
      ▼                                   ▼
┌─────────────┐                    ┌─────────────┐
│   Static    │                    │  Database   │
│   Content   │                    │             │
└─────────────┘                    └─────────────┘
```

## Actix Web Implementation

The Actix Web implementation focuses on serving static content efficiently:

```rust
// From main.rs - Core implementation
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv::dotenv().ok();

    env_logger::init_from_env(env_logger::Env::new().default_filter_or("info"));

    let frontend_port = env::var("FRONTEND_INTERNAL_PORT").unwrap_or_else(|_| "4040".to_string());
    let backend_port = env::var("BACKEND_INTERNAL_PORT").unwrap_or_else(|_| "5050".to_string());
    let host = "http://localhost:";

    // Get backend URL from environment or use default
    let backend_url =
        env::var("BACKEND_URL").unwrap_or_else(|_| format!("{}{}", host, backend_port));

    info!("Starting server at http://0.0.0.0:8080");
    info!("Using backend at {}", backend_url);

    HttpServer::new(move || {
        App::new()
            .wrap(middleware::Logger::default())
            .service(health_check)
            // Serve static files from the 'static' directory
            .service(fs::Files::new("/static", "./static").show_files_listing())
            // Serve HTML files from the root directory
            .service(
                fs::Files::new("/", "./static/html")
                    .index_file("search.html")
                    .default_handler(web::to(|| async {
                        HttpResponse::NotFound().body("Not Found")
                    })),
            )
    })
    .bind(format!("0.0.0.0:{}", frontend_port))?
    .run()
    .await
}
```

### Key Server Features

1. **Environment Configuration**: Using `.env` files and environment variables
2. **Static Files Service**: `actix_files` for serving static content
3. **Default Document**: Configured index file (search.html)
4. **HTTP Logging**: Middleware for request logging
5. **Health Check API**: Endpoint for monitoring
6. **Error Handling**: Custom 404 responses

## Static Content Organization

The frontend organizes static content in a structured manner:

```
static/
├── css/           # Stylesheets
│   ├── main.css   # Core styles
│   └── search.css # Search-specific styles
├── js/            # JavaScript files
│   ├── api.js     # Backend API client
│   ├── search.js  # Search functionality
│   └── utils.js   # Utility functions
├── images/        # Static images
│   └── logo.png   # Site logo
└── html/          # HTML templates
    ├── search.html   # Search page (index)
    └── results.html  # Results page
```

### Client-Side JavaScript Pattern

```javascript
// Example from static/js/api.js - Client-side API handling
async function searchApi(query, options = {}) {
  const params = new URLSearchParams({ q: query, ...options });
  const response = await fetch(`/api/search?${params}`);
  
  if (!response.ok) {
    throw new Error(`Search failed: ${response.status}`);
  }
  
  return await response.json();
}

// Usage in static/js/search.js
document.getElementById('search-form').addEventListener('submit', async (e) => {
  e.preventDefault();
  const query = document.getElementById('search-input').value;
  
  try {
    const results = await searchApi(query);
    displayResults(results);
  } catch (error) {
    showError(error.message);
  }
});
```

This pattern delegates API handling and DOM manipulation to the client side, simplifying the server implementation.

## Containerization Strategy

The frontend uses a multi-stage Docker build to create efficient, production-ready images:

```dockerfile
# Build stage
FROM rust:1.81-slim AS builder

WORKDIR /usr/src/app

COPY Cargo.toml ./
COPY Cargo.lock ./

# Create empty src/main.rs to build dependencies
RUN mkdir -p src && echo "fn main() {}" > src/main.rs
RUN cargo build --release

# Now copy the actual source code and build again
COPY src ./src
RUN touch src/main.rs && cargo build --release

# -------------------------
# Dev stage: full Rust toolchain + source for local development
FROM rust:1.81-slim AS dev
WORKDIR /usr/src/app
# Install native dependencies (e.g., SSL libs) for development
RUN apt-get update && apt-get install -y \
  pkg-config \
  libssl-dev \
  && rm -rf /var/lib/apt/lists/*
# Copy everything (source, static, config)
COPY . .
# Install development tooling: cargo-watch, dotenv-cli, cargo-make
RUN cargo install cargo-watch dotenv-cli cargo-make
# Expose port for dev server
EXPOSE 8080
# Default command: hot-reload dev server
CMD ["cargo", "make", "dev"]

# Production stage: minimal runtime image
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

### Multi-stage Advantages

1. **Dependency Caching**: Building dependencies separately for faster builds
2. **Development Environment**: Complete toolchain for local development
3. **Minimal Production Image**: Only binary and static files in final image
4. **Separate Concerns**: Build process isolated from runtime environment
5. **Security**: Reduced attack surface in production image

## Development Patterns

The WhoKnows frontend implements several development patterns for efficient workflow:

### Hot Reloading

```toml
# Development dependencies in Cargo.toml
[dependencies]
actix-web = "4.4.0"
actix-files = "0.6.2"
actix-rt = "2.9.0"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
env_logger = "0.10.0"
log = "0.4"
dotenv = "0.15.0"
```

With development tooling installed in the dev container:

```bash
# Hot reload during development
cargo watch -x run

# Using cargo-make for standardized tasks
cargo make dev
```

### Environment Configuration

```
FRONTEND_INTERNAL_PORT=4040   # Port for server binding
BACKEND_URL=http://backend:5050  # Backend service URL
RUST_LOG=info                 # Logging level
```

## Integration with Backend

The frontend communicates with the backend service through REST API calls:

1. **API Discovery**: Backend URL from environment variables
2. **Client-side Communication**: JavaScript fetch API for REST calls
3. **Error Handling**: Graceful handling of backend failures
4. **Content Rendering**: Client-side processing of API responses

```rust
// Backend URL configuration in main.rs
let backend_url = 
    env::var("BACKEND_URL").unwrap_or_else(|_| format!("{}{}", host, backend_port));

info!("Using backend at {}", backend_url);
```

This URL can then be injected into client-side scripts or used for server-side proxying if needed.

## Benefits of This Approach

The static content approach with Actix Web offers several advantages:

1. **Simplicity**: No complex build system or frontend frameworks
2. **Performance**: Fast serving of static content
3. **Separation of Concerns**: Clear boundary between presentation and logic
4. **Developer Experience**: Familiar tooling for Rust developers
5. **Deployment Flexibility**: Easy containerization and integration with CI/CD
6. **Scalability**: Independent scaling of frontend and backend services

## Challenges and Solutions

### Challenge: Client-Side Templating

With no server-side rendering, templating must be handled client-side.

**Solution:**
- Simple JavaScript-based templating or DOM manipulation
- Or use lightweight client libraries if needed

### Challenge: Environment-specific Configuration

Frontend needs to know about backend services.

**Solution:**
- Environment variables injected at container runtime
- Optional: Generate environment-specific JavaScript at startup

### Challenge: Development Experience

**Solution:**
- Multi-stage Docker builds with development-focused image
- Hot-reloading with cargo-watch
- Standardized tasks with cargo-make

---

[<- Back: Docker & Containerization](./04-docker-containerization.md) | [Next: Environment Configuration Management ->](./06-environment-configuration-management.md)
