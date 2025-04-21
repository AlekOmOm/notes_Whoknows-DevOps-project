# 2b. Database Integration 🗃️

[<- Back to Main Topic](./02-rust-backend-development.md) | [Next Sub-Topic: API Design ->](./02c-api-design.md)

## Overview

Database integration is a critical component of any backend system. In the WhoKnows project, SQLx provides a type-safe, async-aware database interface for working with SQLite. This note explores how the project integrates database functionality, implements queries, and manages database connections within the ActixWeb application context, focusing on patterns that ensure reliability, performance, and maintainability.

## Key Concepts

### SQLx as a Rust Database Toolkit

SQLx is a Rust library that provides asynchronous, compile-time verified interactions with databases. Key features include:

- **Type-safe queries**: Compile-time verification of SQL statements
- **Async support**: First-class async/await integration
- **Multiple database backends**: Support for PostgreSQL, MySQL, SQLite, and others
- **Migrations**: Database schema versioning and evolution
- **Macros**: Simplified query execution and result mapping

The WhoKnows project uses SQLx with SQLite, leveraging its simple deployment model while maintaining a robust query interface.

### Connection Pooling

Database connections are managed through a connection pool, which:

- Reuses connections to reduce overhead
- Limits the total number of connections
- Handles connection timeouts and failures
- Provides connection lifecycle management

```rust
// From main.rs - Connection pool setup
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

## Implementation Patterns

### Pattern 1: Type-Safe Query Macros

SQLx provides the `query!` macro for compile-time verified SQL queries:

```rust
// From main.rs - Example of SQLx query! macro
match sqlx::query!(
    "SELECT id, username, email, password FROM users WHERE username = ?",
    username
)
.fetch_optional(pool.get_ref())
.await
{
    Ok(Some(record)) => {
        // Directly access typed fields
        let user_id = record.id.expect("DB schema violation: id is NULL");
        let user_password = record.password;
        let user_username = record.username;
        let user_email = record.email;
        
        // Authentication logic...
    }
    // Error handling...
}
```

**When to use this pattern:**
- When you need compile-time SQL verification
- For queries with static structure
- When you want direct field access with proper types
- To catch SQL or schema errors at compile time

### Pattern 2: Data Models with FromRow

SQLx integrates with Rust structs using the `FromRow` trait:

```rust
// From models.rs - Data model with SQLx FromRow integration
#[derive(Serialize, Deserialize, FromRow, Debug, Clone)]
pub struct User {
    pub id: i64,
    pub username: String,
    pub email: String,
    pub password: String,
}

// Usage example retrieving a collection of models
match sqlx::query!(
    "SELECT title, url, language, last_updated, content FROM pages WHERE language = ?1 AND content LIKE ?2",
    language,
    pattern
)
.fetch_all(pool.get_ref())
.await
{
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
        
        HttpResponse::Ok().json(serde_json::json!({ "search_results": pages }))
    },
    // Error handling...
}
```

**When to use this pattern:**
- For mapping database rows to domain models
- When working with collections of results
- To integrate database results with API responses
- When you need full control over the mapping process

### Pattern 3: Transaction Management

SQLx provides transaction support for operations that need atomicity:

```rust
// Example of transaction management (not shown in the provided code)
let mut tx = match pool.begin().await {
    Ok(tx) => tx,
    Err(e) => {
        log::error!("Failed to start transaction: {:?}", e);
        return HttpResponse::InternalServerError().json(serde_json::json!({
            "error": "Database error"
        }));
    }
};

// Perform multiple operations within the transaction
match sqlx::query!(
    "INSERT INTO user_activity (user_id, activity_type, timestamp) VALUES (?, ?, ?)",
    user_id,
    "login",
    chrono::Utc::now()
).execute(&mut tx).await {
    Ok(_) => {},
    Err(e) => {
        log::error!("Failed to insert activity: {:?}", e);
        let _ = tx.rollback().await;
        return HttpResponse::InternalServerError().json(serde_json::json!({
            "error": "Database error"
        }));
    }
};

// More operations...

// Commit the transaction
match tx.commit().await {
    Ok(_) => {
        HttpResponse::Ok().json(serde_json::json!({
            "status": "success"
        }))
    },
    Err(e) => {
        log::error!("Failed to commit transaction: {:?}", e);
        HttpResponse::InternalServerError().json(serde_json::json!({
            "error": "Database error"
        }))
    }
}
```

**When to use this pattern:**
- For operations requiring atomicity
- When multiple database changes must succeed or fail together
- For maintaining data consistency across related tables
- When implementing complex business operations

## Common Challenges and Solutions

### Challenge 1: Handling NULL Values

Database NULL values require careful handling in Rust.

**Solution:**

```rust
// Pattern for handling potentially NULL values
match sqlx::query!(
    "SELECT title, url, language, last_updated, content FROM pages WHERE id = ?",
    page_id
)
.fetch_optional(pool.get_ref())
.await
{
    Ok(Some(record)) => {
        // Handle required fields
        let title = match record.title {
            Some(t) => t,
            None => {
                log::error!("Database inconsistency: NULL title for page {}", page_id);
                return HttpResponse::InternalServerError().json(serde_json::json!({
                    "error": "Database inconsistency"
                }));
            }
        };
        
        // Handle optional fields with defaults or transformations
        let last_updated_display = record.last_updated
            .map(|dt| dt.format("%Y-%m-%d %H:%M").to_string())
            .unwrap_or_else(|| "Never".to_string());
            
        // Create response...
    },
    // Other cases...
}
```

### Challenge 2: Database Migrations

Managing schema evolution in a deployed application.

**Solution:**

```rust
// Example of SQLx migrations setup (conceptual, not in the provided code)
async fn run_migrations(pool: &SqlitePool) -> Result<(), sqlx::Error> {
    let migrator = sqlx::migrate!("./migrations");
    migrator.run(pool).await
}

// Usage during application startup
let pool = SqlitePoolOptions::new()
    .max_connections(5)
    .connect(&database_url)
    .await?;
    
if let Err(e) = run_migrations(&pool).await {
    log::error!("Failed to run database migrations: {:?}", e);
    std::process::exit(1);
}
```

## Practical Example

The WhoKnows project uses SQLx for user authentication in the login handler:

```rust
#[post("/api/login")]
async fn post_login(
    pool: web::Data<SqlitePool>,
    payload: web::Json<LoginForm>,
    session: Session,
) -> impl Responder {
    let login_data = payload.into_inner();

    // Basic validation
    if login_data.username.trim().is_empty() || login_data.password.is_empty() {
        return HttpResponse::BadRequest()
            .json(serde_json::json!({"error": "Username and password cannot be empty"}));
    }

    let username = login_data.username.trim();

    // Database query with error handling
    match sqlx::query!(
        "SELECT id, username, email, password FROM users WHERE username = ?",
        username
    )
    .fetch_optional(pool.get_ref())
    .await
    {
        Ok(Some(record)) => {
            // User found, check password
            let user_id = record.id.expect("DB schema violation: id is NULL");
            let user_password = record.password;
            let user_username = record.username;
            let user_email = record.email;

            match verify_password(&user_password, &login_data.password) {
                Ok(true) => {
                    // Password valid, set session
                    log::info!("User '{}' logged in successfully.", user_username);
                    if let Err(e) = session.insert("user_id", user_id) {
                        log::error!("Failed to insert user_id into session: {:?}", e);
                        return HttpResponse::InternalServerError()
                            .json(serde_json::json!({"error": "Login failed (session error)"}));
                    }
                    
                    // Success response
                    FlashMessage::info("You were logged in!").send();
                    HttpResponse::Ok().json(serde_json::json!({
                        "message": "Login successful",
                        "user": {
                            "id": user_id,
                            "username": user_username,
                            "email": user_email
                        }
                    }))
                }
                Ok(false) => {
                    // Password invalid
                    log::warn!(
                        "Failed login attempt for user '{}': Invalid password.",
                        user_username
                    );
                    HttpResponse::Unauthorized()
                        .json(serde_json::json!({"error": "Invalid username or password"}))
                }
                Err(e) => {
                    // Password verification error
                    log::error!(
                        "Password verification process failed for user '{}': {:?}",
                        user_username,
                        e
                    );
                    HttpResponse::InternalServerError()
                        .json(serde_json::json!({"error": "Login failed (verification error)"}))
                }
            }
        }
        Ok(None) => {
            // User not found
            log::warn!("Failed login attempt: Username '{}' not found.", username);
            HttpResponse::Unauthorized()
                .json(serde_json::json!({"error": "Invalid username or password"}))
        }
        Err(e) => {
            // Database error
            log::error!(
                "Database error during login for username '{}': {:?}",
                username,
                e
            );
            HttpResponse::InternalServerError()
                .json(serde_json::json!({"error": "Login failed (database error)"}))
        }
    }
}
```

This example demonstrates:
1. **Query execution** with proper error handling
2. **Type-safe field access** from the result
3. **Layered error handling** for different failure scenarios
4. **Appropriate HTTP responses** based on the result
5. **Logging** at different severity levels

## Integration with DevOps Pipeline

SQLite is particularly well-suited for containerized deployments due to its file-based nature. The WhoKnows project leverages this with:

1. **Volume Mounting**: Database file persistence across container restarts
2. **Connection Configuration**: Environment variable configuration of database path
3. **Initialization Checks**: Ensuring database and tables exist at startup

```yaml
# From docker-compose.dev.yml - Volume mounting for database
services:
  backend:
    # Other configuration...
    environment:
      - DATABASE_URL=${DATABASE_URL}
    volumes:
      - /home/deployer/deployment/app/data:/app/data
```

The connection string is specified in the environment:

```
# From .env.development
DATABASE_URL=sqlite:/app/data/whoknows.db?mode=rwc # read/write/create
```

## Summary

1. SQLx provides type-safe database access for Rust applications
2. Connection pooling manages database connections efficiently
3. Query macros enable compile-time verification of SQL
4. FromRow trait integrates database results with domain models
5. Error handling is critical for robust database operations
6. SQLite's file-based approach simplifies containerized deployments

## Next Steps

With the database integration foundation established, the next topic explores API design in Rust, focusing on how to structure endpoints, handle requests and responses, and design a coherent API that serves both frontend clients and potential external consumers.

---

[<- Back to Main Topic](./02-rust-backend-development.md) | [Next Sub-Topic: API Design ->](./02c-api-design.md)
