# 5B. Project Structure 📁

[<- Back to Main Topic](./05-frontend-static-content.md) | [Next Sub-Topic: Multi-stage Docker Builds ->](./05c-multi-stage-docker-builds.md)

## Overview

An organized project structure is crucial for maintainable frontend applications, even when serving static content. The WhoKnows frontend implements a clean separation between server code and static assets, making development, testing, and deployment more efficient. This sub-topic explores the structure patterns that enable this separation while supporting a streamlined workflow.

## Key Concepts

### Root Project Organization

The WhoKnows frontend follows a logical organization that separates Rust server code from static content:

```
frontend/
├── Cargo.toml         # Rust dependencies and package metadata
├── Cargo.lock         # Locked dependency versions
├── .env               # Environment variables for local development
├── Dockerfile         # Container build instructions
├── src/               # Rust server code
│   └── main.rs        # Actix Web application entry point
├── static/            # Static content files
│   ├── css/           # Stylesheets
│   ├── js/            # JavaScript files
│   ├── images/        # Image assets
│   └── html/          # HTML templates
└── .dockerignore      # Files to exclude from Docker builds
```

This organization creates a clear boundary between:
- Server components (Rust code in `src/`)
- Content files (static assets in `static/`)
- Build configuration (Cargo.toml, Dockerfile)

### Static Content Structure

The static content follows web standards best practices:

```
static/
├── css/              # Stylesheet directory
│   ├── normalize.css # CSS reset/normalization
│   ├── main.css      # Core styles
│   └── search.css    # Feature-specific styles
├── js/               # JavaScript directory
│   ├── api.js        # Backend communication
│   ├── ui.js         # UI interactions
│   └── search.js     # Search functionality
├── images/           # Image assets
│   ├── logo.png      # Site logo
│   ├── icons/        # UI icons
│   └── backgrounds/  # Background images
└── html/             # HTML templates
    ├── search.html   # Search page (index)
    ├── results.html  # Results page
    └── about.html    # About page
```

This structure supports:
- Logical grouping by file type
- Clear responsibility separation
- Efficient caching strategies
- Easy maintenance and updates

## Implementation Patterns

### Pattern 1: Separation by Responsibility

The WhoKnows frontend separates files based on their role in the application:

- **Server Code**: `src/` directory contains all Rust code
- **Static Assets**: `static/` directory contains all content files
- **Build Configuration**: Root directory contains build and deployment files

**When to use this pattern:**
- When serving static content with minimal server logic
- For projects where multiple people may work on different aspects
- To enable independent updates to server or content components

### Pattern 2: Feature-based Organization

Within static content, files can be organized by feature or functionality:

```
static/js/
├── common/           # Shared utilities
│   ├── api.js        # API client
│   └── utils.js      # Helper functions
├── search/           # Search feature
│   ├── index.js      # Main search functionality
│   ├── results.js    # Results handling
│   └── filters.js    # Search filters
└── user/             # User features
    ├── auth.js       # Authentication
    └── profile.js    # User profile
```

**When to use this pattern:**
- For larger applications with distinct feature sets
- When different team members own different features
- To support modular functionality with lower coupling

### Pattern 3: Modular CSS Architecture

CSS organization follows component-based principles:

```
static/css/
├── base/             # Base styles
│   ├── reset.css     # CSS reset
│   ├── typography.css # Text styles
│   └── variables.css # CSS variables
├── components/       # Reusable components
│   ├── buttons.css   # Button styles
│   ├── forms.css     # Form styles
│   └── cards.css     # Card styles
└── pages/            # Page-specific styles
    ├── search.css    # Search page
    └── results.css   # Results page
```

**When to use this pattern:**
- For maintaining consistent design systems
- When styles need to be reused across multiple pages
- To prevent style conflicts and specificity issues

## Common Challenges and Solutions

### Challenge 1: Content Path Resolution

Server code needs to correctly resolve paths to static content.

**Solution:**

```rust
// In main.rs - Path configuration
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
```

This approach:
- Assumes a relative path structure from the executable
- Maps URL paths to filesystem paths
- Provides fallback handling for missing files

### Challenge 2: Asset References in HTML

HTML files need to reference CSS and JavaScript files with the correct paths.

**Solution:**

```html
<!-- In search.html - Using absolute paths from domain root -->
<head>
  <link rel="stylesheet" href="/static/css/main.css">
  <script src="/static/js/search.js" defer></script>
</head>
```

Using `/static/` prefix ensures:
- Assets are correctly located regardless of HTML file location
- Paths work consistently across all pages
- Browser caching works as expected

## Practical Example

A complete example of the project structure with sample files:

### Directory Structure

```
frontend/
├── src/
│   └── main.rs        # Server entry point
└── static/
    ├── css/
    │   └── main.css   # Main stylesheet
    ├── js/
    │   └── app.js     # Main JavaScript
    └── html/
        └── index.html # Main HTML page
```

### Sample Files

**src/main.rs**
```rust
use actix_files as fs;
use actix_web::{middleware, App, HttpServer};
use std::env;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    env_logger::init_from_env(env_logger::Env::new().default_filter_or("info"));
    
    HttpServer::new(|| {
        App::new()
            .wrap(middleware::Logger::default())
            .service(fs::Files::new("/static", "./static"))
            .service(
                fs::Files::new("/", "./static/html")
                    .index_file("index.html")
            )
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

**static/html/index.html**
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>WhoKnows Search</title>
    <link rel="stylesheet" href="/static/css/main.css">
    <script src="/static/js/app.js" defer></script>
</head>
<body>
    <header>
        <h1>WhoKnows Search</h1>
    </header>
    <main>
        <form id="search-form">
            <input type="text" id="query" placeholder="Search...">
            <button type="submit">Search</button>
        </form>
        <div id="results"></div>
    </main>
    <footer>
        <p>© 2024 WhoKnows Project</p>
    </footer>
</body>
</html>
```

**static/js/app.js**
```javascript
document.addEventListener('DOMContentLoaded', () => {
    const searchForm = document.getElementById('search-form');
    const resultsContainer = document.getElementById('results');
    
    searchForm.addEventListener('submit', async (e) => {
        e.preventDefault();
        const query = document.getElementById('query').value;
        
        try {
            const response = await fetch(`/api/search?q=${encodeURIComponent(query)}`);
            const data = await response.json();
            
            displayResults(data);
        } catch (error) {
            resultsContainer.innerHTML = `<p class="error">Search failed: ${error.message}</p>`;
        }
    });
    
    function displayResults(data) {
        if (!data.items || data.items.length === 0) {
            resultsContainer.innerHTML = '<p>No results found</p>';
            return;
        }
        
        const resultsList = data.items.map(item => `
            <div class="result-item">
                <h3><a href="${item.url}">${item.title}</a></h3>
                <p>${item.snippet}</p>
            </div>
        `).join('');
        
        resultsContainer.innerHTML = resultsList;
    }
});
```

**static/css/main.css**
```css
/* Base styles */
:root {
    --primary-color: #4a6fdc;
    --text-color: #333;
    --background-color: #f5f5f5;
}

body {
    font-family: Arial, sans-serif;
    line-height: 1.6;
    color: var(--text-color);
    background-color: var(--background-color);
    margin: 0;
    padding: 0;
}

header {
    background-color: var(--primary-color);
    color: white;
    padding: 1rem;
    text-align: center;
}

main {
    max-width: 800px;
    margin: 0 auto;
    padding: 1rem;
}

/* Search form */
#search-form {
    display: flex;
    margin-bottom: 2rem;
}

#query {
    flex: 1;
    padding: 0.5rem;
    font-size: 1rem;
    border: 1px solid #ddd;
    border-radius: 4px 0 0 4px;
}

button {
    background-color: var(--primary-color);
    color: white;
    border: none;
    padding: 0.5rem 1rem;
    font-size: 1rem;
    cursor: pointer;
    border-radius: 0 4px 4px 0;
}

/* Results styling */
.result-item {
    margin-bottom: 1.5rem;
    padding-bottom: 1rem;
    border-bottom: 1px solid #eee;
}

.result-item h3 {
    margin-bottom: 0.5rem;
}

.result-item a {
    color: var(--primary-color);
    text-decoration: none;
}

.result-item a:hover {
    text-decoration: underline;
}

.error {
    color: #e74c3c;
    font-weight: bold;
}

/* Footer */
footer {
    text-align: center;
    padding: 1rem;
    margin-top: 2rem;
    font-size: 0.8rem;
    color: #666;
}
```

## Summary

1. A well-organized project structure separates server code from static content
2. Files are grouped by type (HTML, CSS, JS) for clear organization
3. Within each type, files can be organized by feature or responsibility
4. Path resolution is handled consistently for proper asset loading
5. The structure supports independent development of server and content

## Next Steps

The next sub-topic explores multi-stage Docker builds, focusing on how the project structure influences container creation and optimization for both development and production environments.

---

[<- Back to Main Topic](./05-frontend-static-content.md) | [Next Sub-Topic: Multi-stage Docker Builds ->](./05c-multi-stage-docker-builds.md)
