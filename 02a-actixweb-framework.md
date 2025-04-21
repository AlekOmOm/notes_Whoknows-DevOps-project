# 2a. ActixWeb Framework 🌐

[<- Back to Main Topic](./02-rust-backend-development.md) | [Next Sub-Topic: Database Integration ->](./02b-database-integration.md)

## Overview

ActixWeb stands as one of Rust's premier web frameworks, offering high performance, an elegant API, and strong type safety. In the WhoKnows project, ActixWeb serves as the foundation for building a robust, scalable backend service. This note explores the core concepts of ActixWeb as implemented in the project, focusing on architectural patterns, middleware usage, request handling, and integration with the broader deployment pipeline.

## Key Concepts

### The Actor Model

ActixWeb is built on the actor model through the Actix runtime, though direct interaction with actors is uncommon in typical web applications.

Key characteristics include:
- Asynchronous request handling
- Thread-safe state management
- Efficient resource utilization
- Isolated error handling

### Application Structure

The ActixWeb application follows a structured initialization pattern:

```rust
// From main.rs - Main application structure
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // Configuration and setup...
    
    // Create and start the HTTP server
    HttpServer::new(move || {
        // Middleware configuration
        let cors = Cors::default()
            .allow_any_origin()
            .allow_any_method()
            .allow_any_header();
        
        // Application factory pattern
        App::new()
            // Shared state
            .app_data(web::Data::new(pool.clone()))
            // Middleware
            .wrap(cors)
            .wrap(message_framework.clone())
            .wrap(session_middleware)
            // Routes and handlers
            .service(hello)
            .service(config)
            .service(get_about)
            .service(post_login)
            .service(post_register)
            .service(get_logout)
            .service(get_search)
            .service(get_weather)
    })
    .bind((HOST_NAME, port))?
    .run()
    .await
}
```

The structure follows several important patterns:
- **Application Factory**: New App instance created for each worker thread
- **Middleware Wrapping**: Applied in outside-in order
- **Service Registration**: Handlers attached with their routes
- **Shared State**: Database connections and other shared resources

## Implementation Patterns

### Pattern 1: Handler Macros

ActixWeb uses attribute macros to define route handlers:

```rust
// Basic GET handler
#[get("/")]
async fn hello() -> impl Responder {
    HttpResponse::Ok().body("Hello from Actix Backend!")
}

// GET handler with query parameters
#[get("/api/search")]
async fn get_search(pool: web::Data<SqlitePool>, query: web::Query<SearchQuery>) -> impl Responder {
    // Handler implementation...
}

// POST handler with JSON body
#[post("/api/login")]
async fn post_login(
    pool: web::Data<SqlitePool>,
    payload: web::Json<LoginForm>,
    session: Session,
) -> impl Responder {
    // Handler implementation...
}
```

**When to use this pattern:**
- For straightforward route definitions
- When path and HTTP method are tightly coupled
- For simple, focused handlers
- When the handler name should match its function

### Pattern 2: Shared Application State

ActixWeb provides a typed mechanism for sharing state across handlers:

```rust
// State initialization
let pool = SqlitePoolOptions::new()
    .max_connections(5)
    .connect(&database_url)
    .await?;

// State registration
App::new()
    .app_data(web::Data::new(pool.clone()))
    // Other app configuration...

// State access in handlers
async fn get_search(pool: web::Data<SqlitePool>, query: web::Query<SearchQuery>) -> impl Responder {
    // Use the pool to execute database queries
    match sqlx::query!(
        "SELECT title, url, language, last_updated, content FROM pages WHERE language = ?1 AND content LIKE ?2",
        language,
        pattern
    )
    .fetch_all(pool.get_ref())
    .await
    {
        // Result handling...
    }
}
```

**When to use this pattern:**
- For sharing database connections
- For configuration that's needed across handlers
- For thread-safe, read-only application state
- For resources that should be initialized once

### Pattern 3: Middleware Layers

ActixWeb uses a layered middleware approach:

```rust
// CORS middleware configuration
let cors = Cors::default()
    .allow_any_origin()
    .allow_any_method()
    .allow_any_header();

// Session middleware configuration
let session_middleware = SessionMiddleware::builder(
    CookieSessionStore::default(),
    session_secret_key.clone(),
)
.cookie_secure(true)
.cookie_same_site(SameSite::Lax)
.cookie_http_only(true)
.build();

// Flash messages middleware
let message_store = CookieMessageStore::builder(session_secret_key.clone()).build();
let message_framework = FlashMessagesFramework::builder(message_store).build();

// Application with middleware
App::new()
    .wrap(cors)
    .wrap(message_framework.clone())
    .wrap(session_middleware)
    // Services and routes...
```

**When to use this pattern:**
- For cross-cutting concerns
- For authentication and authorization
- For request/response transformation
- For logging, metrics, and monitoring

## Common Challenges and Solutions

### Challenge 1: Extracting Multiple Data Sources

Handlers often need to extract data from different parts of the request.

**Solution:**

```rust
// Multiple extractor pattern
#[post("/api/update-profile")]
async fn update_profile(
    session: Session,                            // Extract session
    pool: web::Data<SqlitePool>,                 // Extract shared state
    path: web::Path<UserId>,                     // Extract path parameters
    query: web::Query<UpdateOptions>,            // Extract query parameters
    payload: web::Json<ProfileUpdateRequest>,    // Extract JSON body
) -> impl Responder {
    // First, check session authentication
    let user_id: Option<i64> = session.get("user_id").unwrap_or(None);
    
    if user_id.is_none() {
        return HttpResponse::Unauthorized().json(serde_json::json!({
            "error": "Authentication required"
        }));
    }
    
    // Then use other extractors
    let user_id = user_id.unwrap();
    let path_id = path.into_inner();
    
    if user_id != path_id && !query.admin_override {
        return HttpResponse::Forbidden().json(serde_json::json!({
            "error": "Cannot modify another user's profile"
        }));
    }
    
    // Process update with database pool and payload
    match update_user_profile(pool, path_id, payload).await {
        Ok(_) => HttpResponse::Ok().json(serde_json::json!({"status": "updated"})),
        Err(e) => HttpResponse::InternalServerError().json(serde_json::json!({"error": e.to_string()}))
    }
}
```

### Challenge 2: Error Handling

Properly handling and reporting errors is critical in web applications.

**Solution:**

```rust
// Tiered error handling pattern from post_login
match sqlx::query!(
    "SELECT id, username, email, password FROM users WHERE username = ?",
    username
)
.fetch_optional(pool.get_ref())
.await
{
    // Database successful, user found
    Ok(Some(record)) => {
        // Password check
        match verify_password(&user_password, &login_data.password) {
            // Password valid
            Ok(true) => {
                // Session management
                if let Err(e) = session.insert("user_id", user_id) {
                    log::error!("Failed to insert user_id into session: {:?}", e);
                    return HttpResponse::InternalServerError()
                        .json(serde_json::json!({"error": "Login failed (session error)"}));
                }
                // Success response
                HttpResponse::Ok().json(serde_json::json!({
                    "message": "Login successful",
                    "user": { /* user data */ }
                }))
            },
            // Password invalid
            Ok(false) => {
                log::warn!("Failed login attempt: Invalid password.");
                HttpResponse::Unauthorized()
                    .json(serde_json::json!({"error": "Invalid username or password"}))
            },
            // Password verification error
            Err(e) => {
                log::error!("Password verification process failed: {:?}", e);
                HttpResponse::InternalServerError()
                    .json(serde_json::json!({"error": "Login failed (verification error)"}))
            }
        }
    },
    // Database successful, no user found
    Ok(None) => {
        log::warn!("Failed login attempt: Username not found.");
        HttpResponse::Unauthorized()
            .json(serde_json::json!({"error": "Invalid username or password"}))
    },
    // Database error
    Err(e) => {
        log::error!("Database error during login: {:?}", e);
        HttpResponse::InternalServerError()
            .json(serde_json::json!({"error": "Login failed (database error)"}))
    }
}
```

## Practical Example

Let's examine the search implementation in the WhoKnows project:

```rust
// Search query structure
#[derive(Deserialize, Debug)]
struct SearchQuery {
    q: Option<String>,
    language: Option<String>,
}

// Page data model
#[derive(Serialize, FromRow, Debug, Clone)]
struct Page {
    title: String,
    url: String,
    language: String,
    last_updated: Option<NaiveDateTime>,
    content: String,
}

// Search handler
#[get("/api/search")]
async fn get_search(pool: web::Data<SqlitePool>, query: web::Query<SearchQuery>) -> impl Responder {
    // Extract and provide defaults for optional parameters
    let search_term = query.q.as_deref().unwrap_or("");
    let language = query.language.as_deref().unwrap_or("en");

    // Early return for empty search
    if search_term.is_empty() {
        return HttpResponse::Ok().json(serde_json::json!({ "search_results": [] }));
    }

    // Prepare search pattern with wildcards
    let pattern = format!("%{}%", search_term);

    // Execute database query with error handling
    match sqlx::query!(
        "SELECT title, url, language, last_updated, content FROM pages WHERE language = ?1 AND content LIKE ?2",
        language,
        pattern
    )
    .fetch_all(pool.get_ref())
    .await
    {
        // Transform database records to domain objects
        Ok(records) => {
            let pages: Vec<Page> = records.into_iter().map(|rec| {
                Page {
                    title: rec.title.expect("Database schema violation: title should be NOT NULL"),
                    url: rec.url,
                    language: rec.language,
                    last_updated: rec.last_updated,
                    content: rec.content,
                }
            }).collect();

            // Return successful JSON response
            HttpResponse::Ok().json(serde_json::json!({ "search_results": pages }))
        },
        // Handle database errors
        Err(e) => {
            log::error!("Failed to execute search query: {:?}", e);
            HttpResponse::InternalServerError().json(serde_json::json!({ "error": "Database query failed" }))
        }
    }
}
```

This implementation demonstrates several key patterns:
1. **Strong typing** for request parameters and response data
2. **Default values** for optional parameters
3. **Early returns** for edge cases
4. **Database query** execution with proper error handling
5. **Data transformation** from database records to domain objects
6. **Structured JSON responses** for both success and error cases

## Integration with DevOps

The ActixWeb application is designed to fit seamlessly into the DevOps pipeline:

1. **Environment Configuration**: Reading settings from environment variables
2. **Containerization**: Clean separation of build and runtime dependencies
3. **Health Checks**: Endpoints for monitoring application health
4. **Graceful Shutdown**: Proper handling of termination signals
5. **Structured Logging**: Machine-parseable log output for monitoring systems

```rust
// Configuration endpoint for deployment verification
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

## Summary

1. ActixWeb provides a powerful foundation for building web services in Rust
2. The framework's handler-based approach promotes clean, modular code organization
3. Middleware layers enable cross-cutting concerns to be handled consistently
4. Strong typing throughout the application prevents many common errors
5. Integration with async Rust enables efficient resource utilization
6. The architecture fits well with modern DevOps practices

## Next Steps

With the web framework foundation in place, the next step is understanding how the application integrates with databases, focusing on the SQLx library and how it provides type-safe database access in the WhoKnows project.

---

[<- Back to Main Topic](./02-rust-backend-development.md) | [Next Sub-Topic: Database Integration ->](./02b-database-integration.md)
