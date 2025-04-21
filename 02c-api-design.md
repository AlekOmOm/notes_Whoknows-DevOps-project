# 2c. API Design 🔌

[<- Back to Main Topic](./02-rust-backend-development.md) | [Next Sub-Topic: Testing Strategies ->](./02d-testing-strategies.md)

## Overview

API design forms the interface contract between the backend service and its clients. In the WhoKnows project, a RESTful API built with ActixWeb provides structured endpoints for search, authentication, and configuration operations. This note explores the API design principles, patterns, and implementation details used in the project, focusing on creating a consistent, maintainable, and robust API surface.

## Key Concepts

### RESTful API Structure

The WhoKnows API follows REST principles with resource-oriented endpoints:

- **Resource-Based URLs**: Endpoints organized around resources
- **HTTP Methods**: Using appropriate verbs for operations
- **Status Codes**: Semantic HTTP response codes
- **JSON Responses**: Consistent response format
- **Query Parameters**: For filtering and search operations

### Request and Response Models

Strong typing for request and response data ensures consistency:

```rust
// Request model for login
#[derive(Deserialize, Debug)]
struct LoginForm {
    username: String,
    password: String,
}

// Response model example (conceptual)
#[derive(Serialize)]
struct ApiResponse<T> {
    success: bool,
    data: Option<T>,
    error: Option<String>,
}
```

## Implementation Patterns

### Pattern 1: Route Grouping and Prefixing

While not explicitly shown in the provided code, a common pattern is organizing routes by resource or feature:

```rust
// Example of API scope organization (conceptual)
App::new()
    // API scope with common prefix
    .service(
        web::scope("/api")
            // Auth endpoints
            .service(post_login)
            .service(post_register)
            .service(get_logout)
            // Search endpoints
            .service(get_search)
            // Other API endpoints
            .service(get_weather)
    )
    // Non-API endpoints
    .service(hello)
    .service(config)
```

**When to use this pattern:**
- When organizing a larger API surface
- For applying common middleware to groups of routes
- When versioning the API
- For clear separation of API and non-API endpoints

### Pattern 2: Structured Error Responses

The WhoKnows API uses consistent JSON error responses:

```rust
// Pattern for error responses
HttpResponse::BadRequest()
    .json(serde_json::json!({"error": "Username and password cannot be empty"}))

HttpResponse::Unauthorized()
    .json(serde_json::json!({"error": "Invalid username or password"}))

HttpResponse::InternalServerError()
    .json(serde_json::json!({"error": "Login failed (database error)"}))
```

**When to use this pattern:**
- For all error responses in the API
- When clients need to programmatically handle errors
- To provide consistent user-facing error messages
- For standardizing error handling across the application

### Pattern 3: Query Parameter Extraction

The API uses typed query parameter extraction for search operations:

```rust
// Query parameter model
#[derive(Deserialize, Debug)]
struct SearchQuery {
    q: Option<String>,
    language: Option<String>,
}

// Handler with query extraction
#[get("/api/search")]
async fn get_search(pool: web::Data<SqlitePool>, query: web::Query<SearchQuery>) -> impl Responder {
    // Extract with defaults for optional parameters
    let search_term = query.q.as_deref().unwrap_or("");
    let language = query.language.as_deref().unwrap_or("en");
    
    // Handler implementation...
}
```

**When to use this pattern:**
- For endpoints with optional query parameters
- When parameters have default values
- For type-safe parameter handling
- When parameters need validation

## Common Challenges and Solutions

### Challenge 1: Authentication and Authorization

Securing API endpoints requires proper authentication and authorization.

**Solution:**

```rust
// Login endpoint that sets up authentication state
#[post("/api/login")]
async fn post_login(
    pool: web::Data<SqlitePool>,
    payload: web::Json<LoginForm>,
    session: Session,
) -> impl Responder {
    // Authentication logic...
    
    // Set session state upon successful login
    if let Err(e) = session.insert("user_id", user_id) {
        log::error!("Failed to insert user_id into session: {:?}", e);
        return HttpResponse::InternalServerError()
            .json(serde_json::json!({"error": "Login failed (session error)"}));
    }
    
    // Success response...
}

// Example of an authenticated endpoint (conceptual)
#[get("/api/profile")]
async fn get_profile(session: Session, pool: web::Data<SqlitePool>) -> impl Responder {
    // Check authentication
    let user_id: Option<i64> = session.get("user_id").unwrap_or(None);
    
    if user_id.is_none() {
        return HttpResponse::Unauthorized()
            .json(serde_json::json!({"error": "Authentication required"}));
    }
    
    // Proceed with authenticated operation...
}
```

### Challenge 2: Validation and Error Handling

Comprehensive input validation is essential for robust APIs.

**Solution:**

```rust
// Pattern from user registration endpoint
#[post("/api/register")]
async fn post_register(
    pool: web::Data<SqlitePool>,
    payload: web::Json<RegistrationForm>,
) -> impl Responder {
    let registration_data = payload.into_inner();

    // Empty username validation
    if registration_data.username.trim().is_empty() {
        return HttpResponse::BadRequest()
            .json(serde_json::json!({"error": "Username cannot be empty"}));
    }
    
    // Email format validation
    if registration_data.email.trim().is_empty() || !registration_data.email.contains('@') {
        return HttpResponse::BadRequest()
            .json(serde_json::json!({"error": "Invalid email address"}));
    }
    
    // Password validation
    if registration_data.password.is_empty() {
        return HttpResponse::BadRequest()
            .json(serde_json::json!({"error": "Password cannot be empty"}));
    }
    
    // Password confirmation validation
    if registration_data.password != registration_data.password2 {
        return HttpResponse::BadRequest()
            .json(serde_json::json!({"error": "Passwords do not match"}));
    }
    
    // Additional logic...
}
```

## Practical Example

Let's examine the search API implementation in the WhoKnows project:

```rust
#[get("/api/search")]
async fn get_search(pool: web::Data<SqlitePool>, query: web::Query<SearchQuery>) -> impl Responder {
    // 1. Extract and provide defaults for query parameters
    let search_term = query.q.as_deref().unwrap_or("");
    let language = query.language.as_deref().unwrap_or("en");

    // 2. Early validation and return
    if search_term.is_empty() {
        return HttpResponse::Ok().json(serde_json::json!({ "search_results": [] }));
    }

    // 3. Prepare search parameters
    let pattern = format!("%{}%", search_term);

    // 4. Execute search query with proper error handling
    match sqlx::query!(
        "SELECT title, url, language, last_updated, content FROM pages WHERE language = ?1 AND content LIKE ?2",
        language,
        pattern
    )
    .fetch_all(pool.get_ref())
    .await
    {
        // 5. Transform database results to API response
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

            // 6. Return successful response with consistent structure
            HttpResponse::Ok().json(serde_json::json!({ "search_results": pages }))
        },
        // 7. Handle errors with appropriate status code and message
        Err(e) => {
            log::error!("Failed to execute search query: {:?}", e);
            HttpResponse::InternalServerError().json(serde_json::json!({ "error": "Database query failed" }))
        }
    }
}
```

This implementation demonstrates several API design principles:
1. **Typed query parameters** with defaults for optional fields
2. **Early returns** for edge cases (empty search)
3. **Consistent response structure** for both success and error cases
4. **Appropriate status codes** for different scenarios
5. **Error logging** for debugging issues
6. **Transformation of internal models** to appropriate API response format

## API Documentation

While not shown in the provided code, a complete API design would include documentation. Common approaches include:

### OpenAPI/Swagger Documentation

```rust
// Example of OpenAPI documentation with utoipa (conceptual)
#[utoipa::path(
    get,
    path = "/api/search",
    params(
        ("q" = Option<String>, Query, description = "Search term"),
        ("language" = Option<String>, Query, description = "Content language", default = "en")
    ),
    responses(
        (status = 200, description = "Search results", body = SearchResponse),
        (status = 500, description = "Internal server error", body = ErrorResponse)
    )
)]
#[get("/api/search")]
async fn get_search(pool: web::Data<SqlitePool>, query: web::Query<SearchQuery>) -> impl Responder {
    // Implementation...
}
```

### API Endpoints Overview

The WhoKnows API includes these key endpoints:

| Endpoint | Method | Purpose | Auth Required |
|----------|--------|---------|--------------|
| `/api/login` | POST | User authentication | No |
| `/api/register` | POST | User registration | No |
| `/api/logout` | GET | End user session | Yes |
| `/api/search` | GET | Search content | No |
| `/api/weather` | GET | Weather information | No |
| `/config` | GET | System configuration | No* |

*The `/config` endpoint exposes configuration information and might be restricted in production.

## Integration with Frontend

The API is designed with the frontend application in mind, providing:

1. **CORS Configuration**: Allowing cross-origin requests for development
   ```rust
   let cors = Cors::default()
       .allow_any_origin()
       .allow_any_method()
       .allow_any_header();
   ```

2. **JSON Content Type**: Using JSON for all API responses
   ```rust
   HttpResponse::Ok().json(serde_json::json!({ "search_results": pages }))
   ```

3. **Session Management**: Cookie-based sessions for maintaining state
   ```rust
   let session_middleware = SessionMiddleware::builder(
       CookieSessionStore::default(),
       session_secret_key.clone(),
   )
   .cookie_secure(true)
   .cookie_same_site(SameSite::Lax)
   .cookie_http_only(true)
   .build();
   ```

## API Versioning Strategy

While not implemented in the code provided, API versioning is an important consideration for evolving APIs. Common approaches include:

1. **URL Versioning**: `/api/v1/search`, `/api/v2/search`
2. **Header Versioning**: Using `Accept` or custom headers
3. **Query Parameter Versioning**: `/api/search?version=1`

The simplest approach for the WhoKnows project would be URL versioning with scopes:

```rust
// Example of versioned API with scopes (conceptual)
App::new()
    .service(
        web::scope("/api/v1")
            .service(post_login_v1)
            .service(get_search_v1)
            // Other v1 endpoints...
    )
    .service(
        web::scope("/api/v2")
            .service(post_login_v2)
            .service(get_search_v2)
            // Other v2 endpoints...
    )
```

## Summary

1. The WhoKnows API follows RESTful principles with resource-oriented endpoints
2. Strong typing for request and response data ensures consistency
3. Structured error responses provide clear feedback to clients
4. Authentication is implemented through session management
5. Input validation prevents invalid data entry
6. The API is designed for seamless frontend integration

## Next Steps

With a well-designed API in place, the next topic explores testing strategies for Rust backend services, focusing on unit tests, integration tests, and mocking approaches to ensure reliability and maintainability.

---

[<- Back to Main Topic](./02-rust-backend-development.md) | [Next Sub-Topic: Testing Strategies ->](./02d-testing-strategies.md)
