# 2. Rust Backend Development 🦀

[<- Back: Project Setup & Architecture](./01-project-setup-architecture.md) | [Next: CI/CD ->](./03-ci-cd.md)

---
- [2a. ActixWeb Framework](./02a-actixweb-framework.md) 
- [2b. Database Integration](./02b-database-integration.md) 
- [2c. API Design](./02c-api-design.md) 
- [2d. Testing Strategies](./02d-testing-strategies.md) 
---

## Table of Contents

- [Introduction](#introduction)
- [ActixWeb Architecture](#actixweb-architecture)
- [Database Integration](#database-integration)
- [Error Handling](#error-handling)
- [Authentication and Security](#authentication-and-security)
- [Configuration Management](#configuration-management)

## Introduction

Rust has emerged as a powerful language for backend development, offering memory safety without garbage collection, zero-cost abstractions, and fearless concurrency. The WhoKnows project leverages Rust's strengths through the ActixWeb framework to create a high-performance, reliable backend service that integrates with modern deployment pipelines.

This section explores the core elements of Rust backend development in the WhoKnows project, focusing on architecture, patterns, and implementation details that enable efficient development and deployment.

## ActixWeb Architecture

ActixWeb is an asynchronous, high-performance web framework for Rust, built on the actor model through the Actix runtime. The WhoKnows implementation follows a structured approach to ActixWeb architecture:

### Core Application Structure

```rust
// From main.rs - Application setup
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // Environment and configuration setup
    dotenv::dotenv().ok();
    env_logger::init_from_env(env_logger::Env::new().default_filter_or("info"));
    
    // Port configuration with fallback
    let port_str = env::var("BACKEND_INTERNAL_PORT").unwrap_or_else(|_| "8080".to_string());
    let port = port_str
        .parse::<u16>()
        .expect("BACKEND_INTERNAL_PORT must be a valid port number");
    
    // Database connection pool initialization
    let database_url =
        env::var("DATABASE_URL").expect("DATABASE_URL must be set in environment or .env file");
    let pool = SqlitePoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await?;
    
    // Server setup and execution
    HttpServer::new(move || {
        // Middleware configuration
        let cors = Cors::default()
            .allow_any_origin()
            .allow_any_method()
            .allow_any_header();
            
        // Application assembly
        App::new()
            .app_data(web::Data::new(pool.clone()))
            .wrap(cors)
            .wrap(message_framework.clone())
            .wrap(session_middleware)
            .service(hello)
            .service(config)
            .service(get_about)
            .service(post_login)
            // Additional services...
    })
    .bind((HOST_NAME, port))?
    .run()
    .await
}
```

### Handler Pattern

ActixWeb uses function handlers annotated with macros for routing:

```rust
// GET handler example
#[get("/api/search")]
async fn get_search(pool: web::Data<SqlitePool>, query: web::Query<SearchQuery>) -> impl Responder {
    let search_term = query.q.as_deref().unwrap_or("");
    let language = query.language.as_deref().unwrap_or("en");
    
    // Query processing...
    
    HttpResponse::Ok().json(serde_json::json!({ "search_results": pages }))
}

// POST handler example
#[post("/api/login")]
async fn post_login(
    pool: web::Data<SqlitePool>,
    payload: web::Json<LoginForm>,
    session: Session,
) -> impl Responder {
    // Authentication logic...
    
    HttpResponse::Ok().json(serde_json::json!({
        "message": "Login successful",
        "user": {
            "id": user_id,
            "username": user_username,
            "email": user_email
        }
    }))
}
```

## Database Integration

The WhoKnows project uses SQLx for type-safe SQL interactions with a SQLite database:

### Connection Pool Management

```rust
// Database connection pool initialization
let pool = match SqlitePoolOptions::new()
    .max_connections(5)
    .connect(&database_url)
    .await
{
    Ok(p) => {
        log::info!("Successfully connected to the database at {}", database_url);
        p
    }
    Err(e) => {
        log::error!("Failed to connect to the database: {}", e);
        std::process::exit(1);
    }
};
```

### Type-Safe Queries

```rust
// Example of SQLx query! macro for type safety
match sqlx::query!(
    "SELECT id, username, email, password FROM users WHERE username = ?",
    username
)
.fetch_optional(pool.get_ref())
.await
{
    Ok(Some(record)) => {
        // Type-safe record access
        let user_id = record.id.expect("DB schema violation: id is NULL");
        let user_password = record.password;
        // Authentication logic...
    }
    // Error handling...
}
```

### Data Models

```rust
// From models.rs - Data model with SQLx integration
#[derive(Serialize, Deserialize, FromRow, Debug, Clone)]
pub struct User {
    pub id: i64,
    pub username: String,
    pub email: String,
    pub password: String,
}

#[derive(Serialize, Deserialize, FromRow, Debug, Clone)]
pub struct Page {
    pub title: String,
    pub url: String,
    pub language: String,
    pub last_updated: Option<DateTime<Utc>>,
    pub content: String,
}
```

## Error Handling

The WhoKnows backend implements structured error handling for robust operation:

### Logging Strategy

```rust
// Tiered logging based on severity
log::info!("User '{}' registered successfully.", username);
log::warn!("Failed login attempt: Username '{}' not found.", username);
log::error!(
    "Database error during login for username '{}': {:?}",
    username,
    e
);
```

### HTTP Response Errors

```rust
// Structured HTTP error responses
HttpResponse::BadRequest()
    .json(serde_json::json!({"error": "Username and password cannot be empty"}))

HttpResponse::Unauthorized()
    .json(serde_json::json!({"error": "Invalid username or password"}))

HttpResponse::InternalServerError()
    .json(serde_json::json!({"error": "Login failed (database error)"}))
```

### Result Handling

```rust
// Rust's Result pattern for error propagation
match verify_password(&user_password, &login_data.password) {
    Ok(true) => {
        // Success path
        log::info!("User '{}' logged in successfully.", user_username);
        // Session management...
    }
    Ok(false) => {
        // Authentication failure
        log::warn!("Failed login attempt for user '{}': Invalid password.", user_username);
        HttpResponse::Unauthorized().json(/* ... */)
    }
    Err(e) => {
        // Technical error
        log::error!("Password verification process failed: {:?}", e);
        HttpResponse::InternalServerError().json(/* ... */)
    }
}
```

## Authentication and Security

The WhoKnows backend implements several security features:

### Password Hashing

```rust
// Argon2 password hashing implementation
fn hash_password(password: &str) -> Result<String, argon2::password_hash::Error> {
    let salt = SaltString::generate(&mut OsRng);
    let argon2 = Argon2::default();
    let password_hash = argon2
        .hash_password(password.as_bytes(), &salt)?
        .to_string();
    Ok(password_hash)
}

fn verify_password(
    stored_hash: &str,
    password_provided: &str,
) -> Result<bool, argon2::password_hash::Error> {
    let parsed_hash = PasswordHash::new(stored_hash)?;
    Ok(Argon2::default()
        .verify_password(password_provided.as_bytes(), &parsed_hash)
        .is_ok())
}
```

### Session Management

```rust
// Session middleware configuration
let session_middleware = SessionMiddleware::builder(
    CookieSessionStore::default(),
    session_secret_key.clone(),
)
.cookie_secure(true) // For HTTPS
.cookie_same_site(SameSite::Lax)
.cookie_http_only(true)
.build();

// Session usage in handlers
if let Err(e) = session.insert("user_id", user_id) {
    log::error!("Failed to insert user_id into session: {:?}", e);
    return HttpResponse::InternalServerError().json(/* ... */);
}

// Session invalidation (logout)
session.purge();
```

## Configuration Management

The WhoKnows backend uses a flexible configuration approach:

### Environment Variables

```rust
// From main.rs - Environment variable handling
dotenv::dotenv().ok(); // Load .env file for local development

// Configuration with fallbacks
let port_str = env::var("BACKEND_INTERNAL_PORT").unwrap_or_else(|_| "8080".to_string());

// Required configuration
let database_url =
    env::var("DATABASE_URL").expect("DATABASE_URL must be set in environment or .env file");

// Configuration debugging endpoint
#[get("/config")]
async fn config() -> impl Responder {
    let db_url = env::var("DATABASE_URL").unwrap_or_else(|_| "Not Set".to_string());
    let port = env::var("BACKEND_INTERNAL_PORT").unwrap_or_else(|_| "Not Set".to_string());
    let environment = env::var("RUST_LOG").unwrap_or_else(|_| "Not Set".to_string());
    let build_version = env::var("BUILD_VERSION").unwrap_or_else(|_| "dev".to_string());

    web::Json(ConfigResponse {
        db_url,
        port,
        environment,
        build_version,
    })
}
```

### Dependency Management

The WhoKnows backend manages dependencies through Cargo:

```toml
# From Cargo.toml - Dependency management
[dependencies]
actix-web = "4"
actix-cors = "0.6"
env_logger = "0.10"
log = "0.4"
dotenv = "0.15" # For loading .env during local development
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
chrono = { version = "0.4", features = ["serde"] }
argon2 = "0.5"
rand = "0.8"
actix-session = { version = "0.9", features = ["cookie-session"] }
actix-web-flash-messages = { version = "0.5", features = ["cookies"] }
sqlx = { version = "0.8", features = [ "runtime-tokio-rustls", "sqlite", "macros", "migrate", "chrono" ] }
hex = "0.4.3"
```

## Integration with DevOps Pipeline

The Rust backend is designed for seamless integration with the CI/CD pipeline:

1. **Environment Awareness**: Reading configuration from environment variables
2. **Docker Compatibility**: Clean separation of build and runtime dependencies
3. **Logging**: Structured logs for monitoring and debugging
4. **Error Handling**: Graceful failure modes with informative messages
5. **Security**: Proper handling of secrets and credentials

```rust
// Example of environment-aware service configuration
let rust_log = env::var("RUST_LOG").unwrap_or_else(|_| "info".to_string());
env_logger::init_from_env(env_logger::Env::new().default_filter_or(&rust_log));

log::info!("Server starting at http://{}:{} in {} mode", 
           HOST_NAME, port, 
           env::var("NODE_ENV").unwrap_or_else(|_| "development".to_string()));
```

---

[<- Back: Project Setup & Architecture](./01-project-setup-architecture.md) | [Next: CI/CD ->](./03-ci-cd.md)
