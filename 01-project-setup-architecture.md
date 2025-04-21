# 1. Project Setup & Architecture 🏗️

[<- Back: Main Note](./README.md) | [Next: Rust Backend Development ->](./02-rust-backend-development.md)

---
- [1a. Repository Structure](./01a-repository-structure.md) 
- [1b. Documentation Organization](./01b-documentation-organization.md) 
- [1c. System Components & Interactions](./01c-system-components-interactions.md) 
- [1d. Configuration Management](./01d-configuration-management.md) 
---

## Table of Contents

- [Introduction](#introduction)
- [Key Architecture Considerations](#key-architecture-considerations)
- [Project Structure Overview](#project-structure-overview)
- [Component Separation](#component-separation)
- [Configuration Strategy](#configuration-strategy)

## Introduction

Modern Rust backend systems require careful architectural planning to leverage Rust's strengths while supporting efficient DevOps workflows. The WhoKnows project demonstrates a structured approach to building maintainable, scalable Rust services with ActixWeb, supported by automated deployment pipelines.

The architecture follows a multi-tier design pattern that separates concerns between web services, business logic, and data layers, while ensuring each component follows Rust's ownership and borrowing rules.

## Key Architecture Considerations

Architecture decisions for Rust-based systems should reflect:

- **Safety & Performance**: Leveraging Rust's guarantees while optimizing for throughput
- **Modularity**: Clean separation of concerns with well-defined interfaces
- **Deployment Efficiency**: Supporting containerization and automated deployment
- **Configuration Flexibility**: Environment-specific configuration with proper secrets management
- **Testability**: Architecting for comprehensive testing at all levels

### Code Organization Approach

```rust
// Example of code organization pattern in the project
mod api {
    mod handlers;
    mod routes;
    mod middleware;
}

mod domain {
    mod models;
    mod services;
    mod repositories;
}

mod infrastructure {
    mod database;
    mod config;
    mod logging;
}
```

## Project Structure Overview

The WhoKnows project uses a multi-component structure:

| Component | Purpose | Tech Stack | Deployment Pattern |
|-----------|---------|------------|-------------------|
| Backend | Core business logic & API | Rust, ActixWeb | Docker container |
| Frontend | User interface | JavaScript/Framework | Docker container |
| Database | Data persistence | PostgreSQL | Managed service or container |
| CI/CD Pipeline | Automation | GitHub Actions | GitHub-hosted runners |

The overall structure emphasizes clear boundaries between components with well-defined interfaces.

## Component Separation

The system's design maintains separation between components through:

1. **Repository division**: Core components exist in their own directories
2. **Interface contracts**: Defined API boundaries between services
3. **Containerization**: Each component runs in its own environment
4. **Configuration isolation**: Environment-specific variables

### Implementation Example

```rust
// Handler that demonstrates separation of concerns
async fn search_handler(
    query: web::Query<SearchQuery>,
    data: web::Data<AppState>,
) -> impl Responder {
    // API layer only handles request/response formatting
    let search_term = query.term.clone();
    
    // Business logic delegated to service layer
    match data.search_service.execute_search(search_term).await {
        Ok(results) => HttpResponse::Ok().json(results),
        Err(e) => {
            log::error!("Search error: {}", e);
            HttpResponse::InternalServerError().json(ErrorResponse {
                message: "Search operation failed".to_string(),
            })
        }
    }
}
```

## Configuration Strategy

The project implements a multi-layered configuration approach, allowing for:

- Development vs. production settings
- Container-specific environment variables
- Secrets management without embedding sensitive data in code
- Dynamic configuration at deployment time

### Best Practices

1. **Environment Abstraction**: Code should never need to know which environment it's running in
2. **Validation at Startup**: Fail early if required configuration is missing
3. **Sensible Defaults**: Provide reasonable defaults where possible
4. **Separation of Concerns**: Configuration hierarchy should reflect component boundaries
5. **Secrets Management**: Never commit secrets to code, use GitHub Secrets and environment variables

---

[<- Back: Main Note](./README.md) | [Next: Rust Backend Development ->](./02-rust-backend-development.md)
