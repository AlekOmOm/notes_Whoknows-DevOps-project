# 2d. Testing Strategies 🧪

[<- Back to Main Topic](./02-rust-backend-development.md) | [Next Topic: CI/CD ->](./03-ci-cd.md)

## Overview

Testing is essential for maintaining reliability and preventing regressions in Rust backend applications. While the WhoKnows project doesn't explicitly showcase testing implementations in the provided code, this note outlines minimalist but effective testing strategies for Rust backend services that can be integrated with CI/CD pipelines. The focus is on practical, straightforward approaches that provide high value with minimal complexity.

## Key Concepts

### Minimalist Testing Pyramid

A simplified testing approach focuses on the most valuable test types:

```
    ┌───────────┐
    │ API Tests │ ← Test HTTP endpoints
    ├───────────┤
    │  Domain   │ ← Test business logic
    ├───────────┤
    │   Unit    │ ← Test isolated functions
    └───────────┘
```

### Rust Testing Fundamentals

Rust's built-in testing capabilities provide most of what you need:

```rust
// File: src/lib.rs or any module file
// Unit test in the same file as the code under test
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_function_behavior() {
        // Arrange
        let input = "test input";
        
        // Act
        let result = function_under_test(input);
        
        // Assert
        assert_eq!(result, expected_output);
    }
}

// File: tests/integration_test.rs
// Integration test in a separate directory
use my_crate::my_module;

#[test]
fn test_module_integration() {
    // Test that uses the public API of the crate
}
```

## Implementation Patterns

### Pattern 1: Pure Function Testing

The simplest and most valuable tests validate core business logic:

```rust
// Example based on the password verification function
fn verify_password(
    stored_hash: &str,
    password_provided: &str,
) -> Result<bool, argon2::password_hash::Error> {
    let parsed_hash = PasswordHash::new(stored_hash)?;
    Ok(Argon2::default()
        .verify_password(password_provided.as_bytes(), &parsed_hash)
        .is_ok())
}

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_verify_password_valid() {
        // This would require a pre-computed hash for a known password
        let hash = "$argon2id$v=19$m=4096,t=3,p=1$somesalt$hashvalue";
        let result = verify_password(hash, "correct_password").unwrap();
        assert!(result);
    }
    
    #[test]
    fn test_verify_password_invalid() {
        let hash = "$argon2id$v=19$m=4096,t=3,p=1$somesalt$hashvalue";
        let result = verify_password(hash, "wrong_password").unwrap();
        assert!(!result);
    }
    
    #[test]
    fn test_verify_password_invalid_hash() {
        let result = verify_password("invalid_hash", "any_password");
        assert!(result.is_err());
    }
}
```

### Pattern 2: Simple Mocking for Domain Logic

For business logic that depends on external systems, simple mocks are effective:

```rust
// Domain service that needs database access
struct UserService<R: UserRepository> {
    repo: R,
}

impl<R: UserRepository> UserService<R> {
    fn new(repo: R) -> Self {
        Self { repo }
    }
    
    async fn authenticate_user(&self, username: &str, password: &str) -> Result<bool, Error> {
        match self.repo.find_by_username(username).await? {
            Some(user) => {
                verify_password(&user.password, password)
                    .map_err(|_| Error::AuthenticationError)
            }
            None => Ok(false),
        }
    }
}

// Test with a simple mock
#[cfg(test)]
mod tests {
    use super::*;
    
    struct MockRepo {
        users: Vec<User>,
    }
    
    #[async_trait]
    impl UserRepository for MockRepo {
        async fn find_by_username(&self, username: &str) -> Result<Option<User>, Error> {
            Ok(self.users.iter()
                .find(|u| u.username == username)
                .cloned())
        }
        
        // Other required implementations...
    }
    
    #[tokio::test]
    async fn test_authenticate_user_success() {
        // Arrange
        let mock_repo = MockRepo {
            users: vec![
                User {
                    id: 1,
                    username: "test_user".to_string(),
                    // Pre-computed hash for "correct_password"
                    password: "$argon2id$v=19$m=4096,t=3,p=1$somesalt$hashvalue".to_string(),
                    email: "test@example.com".to_string(),
                },
            ],
        };
        
        let service = UserService::new(mock_repo);
        
        // Act
        let result = service.authenticate_user("test_user", "correct_password").await;
        
        // Assert
        assert!(result.unwrap());
    }
}
```

### Pattern 3: Minimalist API Testing

Testing HTTP endpoints with a running test server:

```rust
#[cfg(test)]
mod api_tests {
    use super::*;
    use actix_web::{test, App};
    
    #[actix_web::test]
    async fn test_get_about_endpoint() {
        // Arrange - Create test app
        let app = test::init_service(
            App::new()
                .service(get_about)
        ).await;
        
        // Act - Send request to the test server
        let req = test::TestRequest::get().uri("/api/about").to_request();
        let resp = test::call_service(&app, req).await;
        
        // Assert - Check response
        assert_eq!(resp.status(), http::StatusCode::OK);
        
        let body = test::read_body(resp).await;
        assert_eq!(body, "About page placeholder");
    }
    
    #[actix_web::test]
    async fn test_search_endpoint_empty() {
        // Similar test for search endpoint with empty query
        let app = test::init_service(
            App::new()
                .app_data(web::Data::new(get_test_db_pool().await))
                .service(get_search)
        ).await;
        
        let req = test::TestRequest::get().uri("/api/search").to_request();
        let resp = test::call_service(&app, req).await;
        
        assert_eq!(resp.status(), http::StatusCode::OK);
        
        let body: serde_json::Value = test::read_body_json(resp).await;
        assert_eq!(body, serde_json::json!({"search_results": []}));
    }
}
```

## Integration with CI/CD Pipeline

The testing strategy should integrate seamlessly with the CI/CD pipeline:

### 1. Automated Test Runs

```yaml
# Extract from a GitHub Actions workflow
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          override: true
      
      - name: Run Tests
        run: |
          cd backend
          cargo test --all-features
      
      # Optional - Run with code coverage
      - name: Run Tests with Coverage
        run: |
          cargo install cargo-tarpaulin
          cargo tarpaulin --out Xml
      
      # Optional - Upload coverage report
      - name: Upload Coverage to Codecov
        uses: codecov/codecov-action@v3
```

### 2. Test Database Setup

For tests that require a database:

```rust
// Helper for test database setup
async fn get_test_db_pool() -> SqlitePool {
    // Create an in-memory SQLite database for testing
    let pool = SqlitePoolOptions::new()
        .max_connections(5)
        .connect("sqlite::memory:")
        .await
        .expect("Failed to create test database");
    
    // Run migrations to set up schema
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .expect("Failed to run migrations");
    
    // Optional: Seed with test data
    sqlx::query!(
        "INSERT INTO users (username, email, password) VALUES (?, ?, ?)",
        "test_user",
        "test@example.com",
        "$argon2id$v=19$m=4096,t=3,p=1$somesalt$hashvalue"
    )
    .execute(&pool)
    .await
    .expect("Failed to seed test data");
    
    pool
}
```

### 3. Simplified Test Organization

Organize tests to mirror the application structure:

```
backend/
├── src/
│   ├── main.rs
│   ├── api/
│   │   ├── mod.rs
│   │   ├── auth.rs
│   │   └── search.rs
│   ├── domain/
│   │   ├── mod.rs
│   │   ├── user.rs
│   │   └── search.rs
│   └── infrastructure/
│       ├── mod.rs
│       └── db.rs
└── tests/
    ├── api/
    │   ├── auth_tests.rs
    │   └── search_tests.rs
    ├── domain/
    │   ├── user_tests.rs
    │   └── search_tests.rs
    └── integration/
        └── full_flow_tests.rs
```

## Common Challenges and Solutions

### Challenge 1: Async Testing

Testing async code requires special handling.

**Solution:**

```rust
// Use the #[tokio::test] or #[actix_web::test] attribute
#[tokio::test]
async fn test_async_function() {
    // Arrange
    let input = "test_input";
    
    // Act - await the async function
    let result = async_function_under_test(input).await;
    
    // Assert
    assert_eq!(result, expected_output);
}
```

### Challenge 2: Database Testing Without Complexity

Testing database operations without complex setup.

**Solution:**

```rust
// Use SQLite in-memory database for tests
#[tokio::test]
async fn test_user_creation() {
    // Arrange - Set up in-memory database
    let pool = SqlitePoolOptions::new()
        .connect("sqlite::memory:")
        .await
        .unwrap();
    
    // Create necessary tables
    sqlx::query("CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT, email TEXT, password TEXT)")
        .execute(&pool)
        .await
        .unwrap();
    
    // Act - Run the operation under test
    let result = create_user(&pool, "test_user", "test@example.com", "password123").await;
    
    // Assert
    assert!(result.is_ok());
    
    // Verify state
    let user = sqlx::query_as::<_, User>("SELECT * FROM users WHERE username = 'test_user'")
        .fetch_one(&pool)
        .await
        .unwrap();
    
    assert_eq!(user.username, "test_user");
    assert_eq!(user.email, "test@example.com");
}
```

## Practical Testing Example

A minimal but effective test for the login handler:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use actix_web::{test, web, App};
    use sqlx::sqlite::SqlitePoolOptions;
    
    async fn setup_test_app() -> impl actix_web::dev::Service<
        actix_http::Request,
        Response = actix_web::dev::ServiceResponse,
        Error = actix_web::Error,
    > {
        // Set up test database
        let pool = SqlitePoolOptions::new()
            .connect("sqlite::memory:")
            .await
            .unwrap();
        
        // Create tables
        sqlx::query("CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT, email TEXT, password TEXT)")
            .execute(&pool)
            .await
            .unwrap();
        
        // Create test user (password = "password123")
        let hashed_password = hash_password("password123").unwrap();
        sqlx::query(
            "INSERT INTO users (username, email, password) VALUES (?, ?, ?)"
        )
        .bind("test_user")
        .bind("test@example.com")
        .bind(&hashed_password)
        .execute(&pool)
        .await
        .unwrap();
        
        // Create test session key
        let key = Key::derive_from(&[0; 32]);
        let session_middleware = SessionMiddleware::builder(
            CookieSessionStore::default(),
            key.clone()
        )
        .cookie_secure(false)
        .build();
        
        // Create test application
        test::init_service(
            App::new()
                .app_data(web::Data::new(pool))
                .wrap(session_middleware)
                .service(post_login)
        )
        .await
    }
    
    #[actix_web::test]
    async fn test_login_success() {
        // Arrange
        let app = setup_test_app().await;
        
        // Create login payload
        let payload = web::Json(LoginForm {
            username: "test_user".to_string(),
            password: "password123".to_string(),
        });
        
        // Act
        let req = test::TestRequest::post()
            .uri("/api/login")
            .set_json(&payload)
            .to_request();
        
        let resp = test::call_service(&app, req).await;
        
        // Assert
        assert_eq!(resp.status(), http::StatusCode::OK);
        
        let body: serde_json::Value = test::read_body_json(resp).await;
        assert_eq!(body["message"], "Login successful");
        assert_eq!(body["user"]["username"], "test_user");
    }
    
    #[actix_web::test]
    async fn test_login_invalid_password() {
        // Arrange
        let app = setup_test_app().await;
        
        // Create login payload with wrong password
        let payload = web::Json(LoginForm {
            username: "test_user".to_string(),
            password: "wrong_password".to_string(),
        });
        
        // Act
        let req = test::TestRequest::post()
            .uri("/api/login")
            .set_json(&payload)
            .to_request();
        
        let resp = test::call_service(&app, req).await;
        
        // Assert
        assert_eq!(resp.status(), http::StatusCode::UNAUTHORIZED);
        
        let body: serde_json::Value = test::read_body_json(resp).await;
        assert_eq!(body["error"], "Invalid username or password");
    }
}
```

## Summary

1. **Start minimal**: Focus on testing the most critical parts first
2. **Test in layers**: Unit tests for functions, API tests for endpoints
3. **Use in-memory SQLite**: Simplifies database testing
4. **Integrate with CI**: Automate test execution in the pipeline
5. **Focus on value**: Test business-critical flows thoroughly
6. **Keep it simple**: Minimize test dependencies and setup complexity

## Next Steps

With a solid understanding of testing strategies, the next topic explores Continuous Integration and Continuous Deployment (CI/CD), which builds on testing to provide automated building, verification, and deployment of the application.

---

[<- Back to Main Topic](./02-rust-backend-development.md) | [Next Topic: CI/CD ->](./03-ci-cd.md)
