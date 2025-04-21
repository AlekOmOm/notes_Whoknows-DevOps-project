# 1a. Repository Structure 📁

[<- Back to Main Topic](./01-project-setup-architecture.md) | [Next Sub-Topic: Documentation Organization ->](./01b-documentation-organization.md)

## Overview

Repository structure forms the foundation of any modern Rust project with CI/CD integration. A well-designed repository structure enables efficient development workflows, clear separation of concerns, and streamlined deployment processes. The WhoKnows project employs a carefully organized structure that supports both Rust backend development and robust DevOps practices.

## Key Concepts

### Monorepo vs. Polyrepo

The WhoKnows project uses a monorepo approach, keeping related components in a single repository.

```
whoknows/
├── backend/       # Rust ActixWeb backend
├── frontend/      # Frontend application
├── deployment/    # Deployment scripts and configurations
├── .github/       # GitHub Actions workflows
└── docs/          # Documentation
```

**Benefits of monorepo:**
- Single source of truth for all project components
- Atomic changes across multiple components
- Simplified dependency management
- Centralized CI/CD configuration
- Unified issue tracking and project management

**Trade-offs:**
- Potentially larger repository size
- Git history complexity
- Access control granularity challenges
- Increased risk of merge conflicts in busy teams

### Component Directory Structure

Within the backend component, the repository follows Rust conventions while organizing code for clarity:

```
backend/
├── Cargo.toml       # Rust package manifest
├── Cargo.lock       # Dependency lock file
├── Dockerfile       # Container definition
├── src/             # Application source code
│   ├── main.rs      # Application entry point
│   ├── api/         # API endpoints and handlers
│   ├── domain/      # Core business logic and models
│   ├── config/      # Configuration management
│   ├── db/          # Database access layer
│   └── utils/       # Shared utilities
├── tests/           # Integration tests
└── migrations/      # Database migrations
```

## Implementation Patterns

### Pattern 1: Service-Oriented Structure

The repository structure aligns with service boundaries:

```rust
// src/api/search.rs - Example API module showing service-oriented structure
use crate::domain::search::{SearchRequest, SearchResult, SearchError};
use crate::domain::services::SearchService;

pub async fn search_endpoint(
    req: web::Json<SearchRequest>,
    search_service: web::Data<SearchService>,
) -> Result<web::Json<SearchResult>, SearchError> {
    let result = search_service.execute_search(req.into_inner()).await?;
    Ok(web::Json(result))
}
```

**When to use this pattern:**
- When distinct functional domains exist within the application
- When separation of business logic from API concerns is important
- When testing requires mocking of service dependencies

### Pattern 2: Feature-Based Organization

Repository components can be organized by features rather than technical layers:

```
src/
├── features/
│   ├── auth/         # Authentication feature
│   │   ├── api.rs    # Auth API endpoints
│   │   ├── models.rs # Auth data models
│   │   └── service.rs # Auth business logic
│   ├── search/       # Search feature
│   │   ├── api.rs    # Search API endpoints
│   │   ├── models.rs # Search data models
│   │   └── service.rs # Search business logic
│   └── ...
└── shared/          # Shared utilities and components
```

**When to use this pattern:**
- For larger applications with many distinct features
- When features have minimal overlap in functionality
- When different teams work on different features
- When feature toggles or conditional compilation is needed

## Common Challenges and Solutions

### Challenge 1: Dependency Management

Managing dependencies across different components in a monorepo can be challenging.

**Solution:**

```toml
# backend/Cargo.toml example showing workspace-aware dependency management
[workspace]
members = [
    "core",
    "api",
    "utils",
]

[dependencies]
# Shared dependencies defined once for the workspace
actix-web = "4.3.1"
tokio = { version = "1.28.0", features = ["full"] }
serde = { version = "1.0.160", features = ["derive"] }
```

### Challenge 2: CI/CD Integration

Ensuring CI/CD workflows can efficiently build and test specific components without rebuilding everything.

**Solution:**

```yaml
# .github/workflows/ci.yml showing path-based workflow triggers
name: CI

on:
  push:
    branches: [ main, development ]
    paths:
      - 'backend/**'
      - '.github/workflows/ci.yml'
  pull_request:
    branches: [ main, development ]
    paths:
      - 'backend/**'
      - '.github/workflows/ci.yml'

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
      - name: Build and test
        run: |
          cd backend
          cargo test --all-features
```

## Practical Example

A complete repository structure example for a modern Rust project with CI/CD:

```
.
├── .github/
│   ├── workflows/
│   │   ├── ci.yml               # CI workflow
│   │   ├── cd.dev.yml           # Development deployment
│   │   └── cd.prod.yml          # Production deployment
│   └── ISSUE_TEMPLATE/          # Issue templates
├── backend/
│   ├── Cargo.toml               # Rust package manifest
│   ├── Dockerfile               # Backend container definition
│   └── src/                     # Rust source code
├── frontend/
│   ├── package.json             # Frontend dependencies
│   ├── Dockerfile               # Frontend container definition
│   └── src/                     # Frontend source code
├── deployment/
│   ├── docker-compose.dev.yml   # Development compose file
│   ├── docker-compose.prod.yml  # Production compose file
│   └── scripts/                 # Deployment scripts
├── docs/
│   ├── architecture/            # Architecture documentation
│   ├── api/                     # API documentation
│   └── development/             # Development guides
├── .gitignore                   # Git ignore file
└── README.md                    # Project README
```

## Summary

1. A well-designed repository structure is foundational for modern Rust projects with CI/CD integration
2. Monorepo approaches offer streamlined workflows but require thoughtful organization
3. Component structures should reflect both Rust conventions and service/feature boundaries
4. CI/CD integration is simplified with clear path-based workflow triggers
5. Repository layout should support both development efficiency and operational needs

## Next Steps

Understanding the repository structure provides the foundation for exploring how documentation is organized within the project, which ensures that all team members have a clear understanding of the codebase, architecture, and operational procedures.

---

[<- Back to Main Topic](./01-project-setup-architecture.md) | [Next Sub-Topic: Documentation Organization ->](./01b-documentation-organization.md)
