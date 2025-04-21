# 5A. Actix Web for Static Content 🌐

[<- Back to Main Topic](./05-frontend-static-content.md) | [Next Sub-Topic: Project Structure ->](./05b-project-structure.md)

## Overview

Actix Web provides robust capabilities for serving static content in Rust applications. This sub-topic examines how the WhoKnows project leverages Actix Web to serve HTML, CSS, and JavaScript files efficiently, creating a lightweight frontend server without complex frameworks or build systems.

## Key Concepts

### Static File Serving

Actix Web's `actix_files` module specializes in serving files from the filesystem, making it ideal for static content delivery:

```rust
use actix_files as fs;
use actix_web::{App, HttpServer};

// In the main application builder
App::new()
    .service(fs::Files::new("/static", "./static").show_files_listing())
    .service(
        fs::Files::new("/", "./static/html")
            .index_file("search.html")
            .default_handler(web::to(|| async {
                HttpResponse::NotFound().body("Not Found")
            })),
    )
```

This setup handles:
- Serving files from `./static` directory at `/static` path
- Making HTML files available at the root path
- Setting `search.html` as the default document
- Providing custom 404 responses for missing files

### HTTP Server Configuration

The WhoKnows frontend uses Actix Web's `HttpServer` for runtime configuration:

```rust
HttpServer::new(move || {
    App::new()
        .wrap(middleware::Logger::default())
        .service(health_check)
        // Static file services defined here
})
.bind(format!("0.0.0.0:{}", frontend_port))?
.run()
.await
```

Key aspects:
- Binding to a configurable port from environment variables
- Application factory pattern (closure passed to `HttpServer::new`)
- Middleware integration for logging
- Async execution with `.await`

## Implementation Patterns

### Pattern 1: Environment-Aware Configuration

```rust
// Environment configuration with fallbacks
let frontend_port = env::var("FRONTEND_INTERNAL_PORT").unwrap_or_else(|_| "4040".to_string());
let backend_port = env::var("BACKEND_INTERNAL_PORT").unwrap_or_else(|_| "5050".to_string());
let host = "http://localhost:";

// Get backend URL from environment or use default
let backend_url =
    env::var("BACKEND_URL").unwrap_or_else(|_| format!("{}{}", host, backend_port));

info!("Starting server at http://0.0.0.0:8080");
info!("Using backend at {}", backend_url);
```

**When to use this pattern:**
- When you need runtime configuration without rebuilding
- For different deployment environments (dev, staging, production)
- To maintain separation between code and configuration

### Pattern 2: Health Check Endpoints

```rust
#[get("/api/health")]
async fn health_check() -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({
        "status": "ok",
        "service": "frontend"
    }))
}
```

**When to use this pattern:**
- For monitoring service health in production
- For integration with container orchestration systems
- To verify the application is running correctly

### Pattern 3: Middleware Integration

```rust
App::new()
    .wrap(middleware::Logger::default())
    // Other services
```

**When to use this pattern:**
- For request logging
- For cross-cutting concerns like authentication
- To add consistent behavior across all endpoints

## Common Challenges and Solutions

### Challenge 1: Path Configuration

When serving static files, path configuration can be confusing, especially with nested routes.

**Solution:**

```rust
// Hierarchical path configuration
App::new()
    // Assets at /static/css, /static/js, etc.
    .service(fs::Files::new("/static", "./static"))
    // HTML files at root level
    .service(
        fs::Files::new("/", "./static/html")
            .index_file("search.html")
    )
```

This approach allows for:
- Clean URL structure (no `.html` extension in URLs)
- Default document handling
- Separation between assets and pages

### Challenge 2: MIME Type Handling

Browsers rely on correct MIME types to process files properly.

**Solution:**

Actix Web's `Files` service automatically:
- Determines MIME types based on file extensions
- Sets appropriate headers in HTTP responses
- Handles byte ranges for partial content requests

For custom MIME types:

```rust
.service(
    fs::Files::new("/downloads", "./files")
        .prefer_utf8(true)
        .use_etag(true)
        .use_last_modified(true)
)
```

## Practical Example

A complete implementation of the static file server with additional features:

```rust
use actix_files as fs;
use actix_web::{get, middleware, web, App, HttpResponse, HttpServer};
use log::info;
use std::env;

// Health check API endpoint
#[get("/api/health")]
async fn health_check() -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({
        "status": "ok",
        "service": "frontend",
        "version": env!("CARGO_PKG_VERSION")
    }))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // Load environment variables
    dotenv::dotenv().ok();
    env_logger::init_from_env(env_logger::Env::new().default_filter_or("info"));

    // Configuration from environment
    let frontend_port = env::var("FRONTEND_INTERNAL_PORT").unwrap_or_else(|_| "4040".to_string());
    let backend_url = env::var("BACKEND_URL").unwrap_or_else(|_| "http://localhost:5050".to_string());

    info!("Starting frontend server at http://0.0.0.0:{}", frontend_port);
    info!("Using backend at {}", backend_url);

    // Build and run HTTP server
    HttpServer::new(move || {
        App::new()
            // Add logger middleware
            .wrap(middleware::Logger::default())
            // Add compression middleware (optional)
            .wrap(middleware::Compress::default())
            // Health check endpoint
            .service(health_check)
            // Static assets (CSS, JS, Images)
            .service(
                fs::Files::new("/static", "./static")
                    .use_etag(true)
                    .use_last_modified(true)
            )
            // HTML pages at root
            .service(
                fs::Files::new("/", "./static/html")
                    .index_file("search.html")
                    .default_handler(web::to(|| async {
                        HttpResponse::NotFound().body("Page not found")
                    }))
            )
            // Store backend URL in app data for JavaScript to access
            .app_data(web::Data::new(backend_url.clone()))
    })
    .bind(format!("0.0.0.0:{}", frontend_port))?
    .run()
    .await
}
```

## Summary

1. Actix Web provides a robust, high-performance platform for serving static content
2. The `actix_files` module offers specialized functionality for file serving
3. Environment variables enable flexible configuration across deployment environments
4. Middleware adds cross-cutting functionality like logging and compression
5. Health check endpoints support monitoring and orchestration systems

## Next Steps

In the next sub-topic, we'll explore the project structure for organizing static content, including HTML, CSS, and JavaScript files, and how they interact with the Actix Web server.

---

[<- Back to Main Topic](./05-frontend-static-content.md) | [Next Sub-Topic: Project Structure ->](./05b-project-structure.md)
