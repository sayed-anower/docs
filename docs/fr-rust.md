# fr-rust Documentation

>[!IMPORTANT]
> Unfortunately, the documentation is not for latest & most stable version.


**fr-rust** is a comprehensive, high-performance web utility ecosystem built on top of Actix-Web. It provides production-ready, out-of-the-box support for DDoS protection, standardized HTTP responses, room-based WebSockets, Redis connection pooling, secure cryptography, and essential authentication services (JWT, OTP, Link Verification, Email).

---

## 1. Quick Start & Prerequisites

### Ecosystem Dependencies
To use **fr-rust** effectively, you must pair it with its core foundation crates. Ensure your `Cargo.toml` includes the following adjacent dependencies:

```toml
[dependencies]
fr-rust = "0.1"
actix-web = "4"
actix-ws = "0.3"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1", features = ["full"] }
futures-util = "0.3"

```
### Server Entry Point & Configuration
Initialize shared state, establish safe application contexts, and protect your endpoints using the built-in token-bucket DDoS shield.
```rust
use fr_rust::prelude::*;

#[web_main]
async fn main() -> Main  {
  let app_state = AppState {
    db_pool,
    jwt
  };
  let ip = env_or_default("IP", "0.0.0.0");
  let port = env_or_default("PORT", "8080");
  let address = format!("{ip}:{port}");

  run_server!(
    state: app_state,
    config: config,
    addr: "0.0.0.0:8080",
    app: {
        .wrap(actix_web::middleware::Logger::default())
        .app_data(web::PayloadConfig::new(10 * 1024 * 1024)) // 10MB
    },
    server: {
        .workers(num_cpus::get() * 2)
        .shutdown_timeout(120)
    }
);
}



fn config(cfg: &mut ServiceConfig) {
    // Simple routes
    get!("/health", health_check);
    post!("/users", create_user);

    // Route with guard / extra config
    get!("/admin/dashboard", admin_dashboard, guard(::actix_web::guard::Header("role", "admin")));

    // Simple scope
    scope!("/api", {
        get!("/users", get_users);
        post!("/users", create_user);
        get!("/users/{id}", get_user);
        put!("/users/{id}", update_user, guard(::actix_web::guard::Not(::actix_web::guard::Header("readonly", "true"))));
    });

    // Scope with guard + middleware
    scope!("/admin", guard(::actix_web::guard::Header("admin", "true")), {
        get!("/stats", get_stats);
        post!("/users", admin_create_user);
    });
}
```
## 2. Framework Type Aliases
**fr-rust** provides semantic type wrappers over base actix-web engines to accelerate development flow:
| Type Alias | Native Equivalent | Context/Description |
|---|---|---|
| Rsp | HttpResponse | Default wrapper for outbound server responses. |
| Rqs | HttpRequest | Standard request metadata representation. |
| Rlt | Result<(), actix_web::Error> | Universal Fallible Internal Actix action. |
| Main | std::io::Result<()> | Standard runtime entry return contract. |
| FileRlt | Result<actix_files::NamedFile, actix_web::Error> | Clean targeted static file streamer token. |
## 3. Unified Responses & Routing
### Efficient Static File Delivery
```rust
#[get("/")]
pub async fn index_file() -> FileRlt {
    send_file("./static/index.html").await
}

```
### Global Response Matrix
Use these optimized abstractions instead of manually assembling raw HttpResponse states:

### Basic Response Helpers

| Response Helper | HTTP Status | Payload Type | When to Use |
|-----------------|-------------|--------------|-------------|
| `http_ok(msg)` | 200 OK | Plain text / String slice | Simple success messages, status updates |
| `http_ok_static(msg)` | 200 OK | Static string (&'static str) | **Fastest** - Error messages, constants |
| `http_bad(msg)` | 400 Bad Request | Error Message String slice | Validation failures, malformed requests |
| `http_bad_static(msg)` | 400 Bad Request | Static string (&'static str) | **Fastest** - Common error messages |
| `send_str(msg)` | 200 OK | Evaluated String | Dynamic string content from variables |
| `send_json(data)` | 200 OK | Structs/Vectors with Serialize | **Preferred** - Type-safe JSON responses |
| `http_ok_json(data)` | 200 OK | Ad-hoc serde_json::json! | Quick prototyping, dynamic structures |
| `http_bad_json(data)` | 400 Bad Request | JSON-formatted errors | Structured error responses |

### Advanced Status Codes

| Response Helper | HTTP Status | When to Use |
|-----------------|-------------|-------------|
| `http_no_content()` | 204 No Content | PUT/POST updates with no response body |
| `http_created(location)` | 201 Created | Resource creation (returns Location header) |
| `http_accepted()` | 202 Accepted | Async operations, batch processing |
| `http_partial_content(data, range, total)` | 206 Partial Content | Video streaming, resume downloads |
| `http_unauthorized(realm)` | 401 Unauthorized | Missing/invalid authentication |
| `http_forbidden(msg)` | 403 Forbidden | Insufficient permissions |
| `http_not_found(msg)` | 404 Not Found | Resource doesn't exist |
| `http_method_not_allowed(methods)` | 405 Method Not Allowed | Wrong HTTP method (returns Allow header) |
| `http_conflict(msg)` | 409 Conflict | Resource version conflict, duplicate entry |
| `http_unsupported_media(msg)` | 415 Unsupported Media Type | Wrong Content-Type |
| `http_too_many_requests(secs)` | 429 Too Many Requests | Rate limiting (returns Retry-After) |
| `http_server_error(msg)` | 500 Internal Server Error | Unexpected server errors |
| `http_service_unavailable(secs)` | 503 Service Unavailable | Maintenance mode (returns Retry-After) |

### Streaming & Large Data

| Response Helper | Use Case | Memory Usage |
|-----------------|----------|--------------|
| `http_ok_stream(stream)` | Large responses, real-time data | Minimal (~64KB buffer) |
| `send_file(path)` | Static files, downloads | OS-level zero-copy |
| `send_file_fast(path, req)` | **Fastest** - Static files with caching | Zero-copy with mmap |
| `send_file_range(path, range)` | Video streaming, partial downloads | Only requested bytes |

### Compression Helpers

| Response Helper | Compression Type | Best For |
|-----------------|------------------|----------|
| `http_brotli(data, quality)` | Brotli (20-30% better than gzip) | Text/JSON APIs, static assets |
| `http_lz4(data)` | LZ4 (10x faster than gzip) | Real-time APIs, low-latency |

### Upload/File Processing

| Response Helper | Use Case | Memory Pattern |
|-----------------|----------|----------------|
| `upload_file(payload, dir)` | Simple file uploads | All in memory (small files) |
| `upload_streaming(payload, dir)` | Large file uploads | Streaming to disk (low memory) |
| `upload_with_progress(payload, dir, cb)` | Uploads with progress tracking | Streaming with callbacks |
| `parse_multipart_stream(payload, handler)` | Custom multipart processing | Chunk-by-chunk processing |

---

## Framework Usage Guide

### 1. Setting Up Your Application

```rust
#[web_main]
async fn main() -> Main {
    // Set Up Your Server Here With FrRust...
}
```

### 2. Response Helper Decision Tree

```
Need to return data?
├── Yes
│   ├── JSON?
│   │   ├── Yes
│   │   │   ├── Type-safe struct? → send_json(data)
│   │   │   ├── Dynamic structure? → http_ok_json(json!({...}))
│   │   │   └── Large JSON (>1MB)? → http_brotli(data, quality) or http_lz4(data)
│   │   └── No
│   │       ├── Static string? → http_ok_static("message")
│   │       ├── Dynamic string? → send_str(variable)
│   │       └── Large text (>100KB)? → http_ok_stream(stream)
│   └── File?
│       ├── Static file? → send_file_fast(path, req)
│       ├── Video/audio? → send_file_range(path, range_header)
│       └── Large download? → send_file(path)
└── No
    ├── Resource created? → http_created("/resource/123")
    ├── Accepted async? → http_accepted()
    └── No content? → http_no_content()
```

### 3. Error Handling Patterns

```rust
// Simple error responses
async fn validate_user(id: u64) -> HttpResponse {
    if id == 0 {
        return http_bad_static("User ID must be positive");
    }
    
    match get_user_from_db(id).await {
        Ok(user) => send_json(user),
        Err(DbError::NotFound) => http_not_found("User not found"),
        Err(DbError::Permission) => http_forbidden("Access denied"),
        Err(_) => http_server_error("Database error"),
    }
}

// Structured JSON errors
async fn api_handler(data: web::Json<Request>) -> HttpResponse {
    if data.name.is_empty() {
        return http_bad_json(json!({
            "error": "Validation failed",
            "field": "name",
            "reason": "Name cannot be empty"
        }));
    }
    
    // ... processing
}
```

### 4. File Serving Implementation

```rust
use actix_web::web::Path;
use actix_web::HttpRequest;

// Simple file server
async fn serve_files(
    req: HttpRequest,
    path: Path<String>,
) -> Result<NamedFile, Error> {
    let filename = path.into_inner();
    let filepath = format!("./static/{}", filename);
    
    // Automatic: ETag, Last-Modified, cache control
    send_file_fast(&filepath, &req).await
}

// Video streaming with range support
async fn stream_video(
    req: HttpRequest,
    path: Path<String>,
) -> Result<HttpResponse, Error> {
    let video_path = format!("./videos/{}.mp4", path.into_inner());
    
    // Check for Range header
    let range_header = req.headers()
        .get(header::RANGE)
        .and_then(|h| h.to_str().ok());
    
    send_file_range(&video_path, range_header).await
}
```

### 5. Upload Implementation

```rust
use actix_multipart::Multipart;

// Standard upload (small files)
async fn upload_handler(
    mut payload: Multipart,
) -> Result<HttpResponse, Error> {
    let files = upload_file(payload, "./uploads").await?;
    
    http_ok_json(json!({
        "status": "success",
        "files": files,
        "count": files.len()
    }))
}

// Large file upload with progress
async fn upload_large_handler(
    mut payload: Multipart,
) -> Result<HttpResponse, Error> {
    let files = upload_with_progress(
        payload, 
        "./uploads",
        |filename, bytes, _total| {
            // You can send WebSocket updates here
            println!("Uploading {}: {} bytes", filename, bytes);
        }
    ).await?;
    
    http_ok_json(json!({
        "status": "success", 
        "files_uploaded": files
    }))
}

// Streaming upload (lowest memory usage)
async fn upload_stream_handler(
    mut payload: Multipart,
) -> Result<HttpResponse, Error> {
    let files = upload_streaming(payload, "./uploads").await?;
    http_ok(json!({"status": "success", "files": files}).to_string())
}
```

### 6. Streaming Large Data

```rust
use futures_util::stream;

// Stream logs in real-time
async fn stream_logs() -> HttpResponse {
    let stream = stream::unfold(0, |counter| async move {
        if counter >= 1000 {
            None
        } else {
            let line = format!("Log entry {}\n", counter);
            tokio::time::sleep(Duration::from_millis(10)).await;
            Some(Ok::<_, Error>(Bytes::from(line)))
        }
    });
    
    http_ok_stream(Box::pin(stream))
}

// Stream from database
async fn stream_users_from_db(pool: &PgPool) -> HttpResponse {
    let stream = sqlx::query_as::<_, User>("SELECT * FROM users")
        .fetch_stream(pool)
        .await
        .unwrap()
        .map_ok(|user| Bytes::from(serde_json::to_vec(&user).unwrap()));
    
    http_ok_stream(Box::pin(stream))
}
```

### 7. Compression Implementation

```rust
async fn get_compressed_data() -> HttpResponse {
    let data = generate_large_dataset(); // Your data
    
    // Check Accept-Encoding header (browser sends this)
    // Use Brotli for text (20-30% better compression)
    http_brotli(&data, 6)  // Quality 0-11, 6 is balanced
    
    // OR use LZ4 for speed (10x faster decompression)
    // http_lz4(&data)
}

// Content negotiation example
async fn smart_compression(req: HttpRequest) -> HttpResponse {
    let data = generate_2mb_response();
    let json = serde_json::to_vec(&data).unwrap();
    
    let accept_encoding = req.headers()
        .get(header::ACCEPT_ENCODING)
        .and_then(|h| h.to_str().ok())
        .unwrap_or("");
    
    if accept_encoding.contains("br") {
        http_brotli(&json, 6)
    } else if accept_encoding.contains("gzip") {
        // Use actix's built-in gzip
        HttpResponse::Ok()
            .insert_header((header::CONTENT_ENCODING, "gzip"))
            .body(compress_gzip(&json))
    } else if accept_encoding.contains("lz4") {
        http_lz4(&json)
    } else {
        send_json(&data)
    }
}
```

### 8. REST API Examples

```rust
// Create resource
async fn create_user(user: web::Json<UserCreate>) -> HttpResponse {
    let id = db.insert(user.into_inner()).await?;
    http_created(&format!("/users/{}", id))
}

// Get resource
async fn get_user(id: web::Path<u64>) -> HttpResponse {
    match db.find_user(*id).await {
        Some(user) => send_json(user),
        None => http_not_found("User not found")
    }
}

// Update resource
async fn update_user(
    id: web::Path<u64>,
    update: web::Json<UserUpdate>,
) -> HttpResponse {
    let updated = db.update(*id, update.into_inner()).await?;
    send_json(updated)
}

// Delete resource
async fn delete_user(id: web::Path<u64>) -> HttpResponse {
    db.delete(*id).await?;
    http_no_content()
}
```

### 9. Rate Limiting Implementation

```rust
use std::collections::HashMap;
use std::sync::Mutex;
use chrono::Utc;

async fn rate_limited_endpoint(req: HttpRequest) -> HttpResponse {
    let ip = req.connection_info().realip_remote_addr().unwrap_or("unknown");
    
    if !check_rate_limit(ip) {
        return http_too_many_requests(30);  // Retry after 30 seconds
    }
    
    http_ok("Request processed successfully")
}

// With custom message
async fn limited_api() -> HttpResponse {
    if is_rate_limited() {
        return http_bad_json(json!({
            "error": "Rate limit exceeded",
            "retry_after": 60,
            "limit": 100,
            "remaining": 0
        }));
    }
    
    send_json(normal_response())
}
```

---

## Performance Tips by Use Case

### Production API (~100 req/sec)
```rust
// Use these for best performance
http_ok_static("OK")        // Health checks
send_json(&data)            // All JSON responses
http_bad_static("Invalid")  // Common errors
```

### High-Throughput API (~10,000 req/sec)
```rust
// Add compression and caching
http_brotli(&data, 4)       // Lower quality for speed
send_file_fast(path, req)   // Zero-copy file serving
http_ok_stream(stream)      // Memory-efficient responses
```

### Media Server (Video/Audio)
```rust
// Use range requests
send_file_range(video_path, range)  // Supports seeking
send_file_fast(path, req)           // HTTP/2 + caching
```

### Real-Time API (WebSocket/SSE)
```rust
http_ok_stream(stream)      // Server-sent events
http_lz4(&data)             // Fastest compression
http_no_content()           // Ack responses
```

---

## Common Patterns & Anti-Patterns

### ✅ DO: Use static strings for errors
```rust
http_bad_static("Invalid input")  // Fastest
```

### ❌ DON'T: Allocate strings unnecessarily
```rust
http_bad(&format!("Error: {}", msg))  // Slower
```

### ✅ DO: Use streaming for large responses
```rust
http_ok_stream(stream_data())
```

### ❌ DON'T: Load entire file into memory
```rust
let data = std::fs::read("large.mp4")?;  // Bad for >100MB
HttpResponse::Ok().body(data)
```

### ✅ DO: Use send_file for static files
```rust
send_file_fast("static/image.jpg", &req).await
```

### ❌ DON'T: Manually set cache headers
```rust
// Let send_file_fast handle ETag/Last-Modified automatically
```

### ✅ DO: Use proper status codes
```rust
http_created("/users/123")  // Correct for creation
http_no_content()           // Correct for successful deletes
```

### ❌ DON'T: Return 200 for all responses
```rust
http_ok("Deleted")  // Wrong! Use http_no_content()
```
## 4. Multi-room WebSockets (WsManager)
Built over the actix-ws asymmetric handling architecture, the internal WsManager provides abstract, thread-safe memory tables tracking channel mappings across endpoints.
```rust
use tokio::sync::mpsc;

#[get("/ws/{user_id}")]
async fn ws_handler(
    req: HttpRequest,
    body: web::Payload,
    ws_manager: web::Data<WsManager>,
    path: web::Path<String>,
) -> Result<Rsp, actix_web::Error> {
    let user_id = path.into_inner();

    // 1. Establish an asynchronous bounded channel to prevent backpressure issues (128 depth)
    let (tx, mut rx) = mpsc::channel::<String>(128);

    // 2. Safely perform the WebSocket upgrade handshake
    let (res, mut session, mut msg_stream) = match actix_ws::handle(&req, body) {
        Ok(parts) => parts,
        Err(err) => return Ok(http_bad(&format!("Handshake failure: {}", err))), 
    };

    // 3. Thread-safe user registration inside the state manager
    ws_manager.register(&user_id, tx);

    // [Your core async read/write event loop goes here...]

    Ok(res)
}

```
### Manager Capabilities Cheat Sheet
```rust
// Room Operations
ws_manager.join_room(&user_id, "lobby_room");
ws_manager.leave_room(&user_id, "lobby_room");
ws_manager.drop_room("lobby_room");

// Disconnect Strategy
ws_manager.drop_user(&user_id);

// Targeted Pub/Sub Propagation
ws_manager.msg_user(&user_id, "Direct Message".to_string());
ws_manager.msg_room("lobby_room", UserMsg::new("System", "lobby_room", "Hello Room!"));
ws_manager.broadcast("Global maintenance alert!".to_string());

```
## 5. Non-Blocking Database Engine
Direct execution wrapper maps across your connections gracefully via AppData<DbPool>.
```rust
#[get("/test/db")]
async fn test_db(pool: web::Data<DbPool>) -> Result<Rsp, actix_web::Error> {
    // 1. Execute DDL/DML statements
    pool.execute("CREATE TABLE IF NOT EXISTS users (id SERIAL PRIMARY KEY, name TEXT);", &[]).await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;
        
    pool.execute("INSERT INTO users (name) VALUES ($1);", &[&"Alice"]).await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;
    
    // 2. Fetch Datasets
    let _rows = pool.query("SELECT id, name FROM users;", &[]).await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;
    
    // 3. Fallback Mapping Strategy for Options
    let maybe_row = pool.query_opt("SELECT name FROM users WHERE id = $1;", &[&999]).await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;
        
    let client_msg = match maybe_row {
        Some(row) => format!("Welcome back, {}!", row.get::<_, String>("name")),
        None => "Target entity sequence id (999) does not exist inside records.".to_string(),
    };
    
    Ok(http_ok(&client_msg))
}
```
## 6. Coordinated Redis Cache
Managed cleanly over deadpool_redis and redis-rs capabilities.
```rust
use deadpool_redis::redis::AsyncCommands;
use futures_util::StreamExt;

async fn process_redis_pipeline(redis_manager: web::Data<RedisManager>) -> Result<(), Box<dyn std::error::Error>> {
    let mut conn = redis_manager.get_connection().await?;

    // Standard KV cache mutation
    let _: () = conn.set("session_key", "active_payload").await?;
    
    // Pub/Sub Broadcast
    let _: () = conn.publish("event_stream", "Message data payload distributed safely").await?;

    // Streaming Event Consumer
    let mut stream = redis_manager.subscribe("event_stream").await?;
    while let Some(msg) = stream.next().await {
        let payload: String = msg.get_payload()?;
        println!("Stream event intercept: {}", payload);
    }
    
    Ok(())
}

```
## 7. Identity & Notification Services
### 7.1 JSON Web Tokens (Jwt)
## Features

- 🔐 **Multiple Algorithms**: HS256, RS256, RS384, ES256, EdDSA
- 🚫 **Token Blacklisting**: Built-in token revocation with automatic cleanup
- 📦 **Multiple Token Types**: Access, Refresh, Reset, Verify, Custom
- ⏰ **Expiration Control**: Configurable token lifetimes
- 🔄 **Token Refresh**: Seamless refresh token rotation
- 🛡️ **Security**: Support for RSA, ECDSA, and Ed25519
- 🚀 **Performance**: Sharded storage with async cleanup

## Quick Start

### 1. Basic Setup

```rust

// Create a JWT service with HS256
let jwt = JwtService::new_hs256("your-secret-key");

// Generate an access token
let token = jwt.generate("user-123", TokenType::Access)?;

// Verify the token
let claims = jwt.verify(&token)?;
println!("User: {}", claims.sub);
```

### 2. With Issuer and Audience

```rust
let jwt = JwtService::new_hs256("secret")
    .with_issuer("my-app")
    .with_audience("my-api");

let token = jwt.generate("user-123", TokenType::Access)?;
```

### 3. Using the Macro

```rust
// Quick setup with macro
let jwt = setup_jwt!("secret", "my-app", "my-api");
```

### 4. With Token Blacklisting

```rust
// Create blacklist with cleanup every 5 minutes
let blacklist = TokenBlacklist::new(300);

let jwt = JwtService::new_hs256("secret")
    .with_blacklist(blacklist);

// Generate token
let token = jwt.generate("user-123", TokenType::Access)?;

// Verify (will check blacklist)
let claims = jwt.verify(&token)?;

// Revoke token
jwt.revoke_token(&token)?;
```

## Core Concepts

### Token Types

| Type | Duration | Use Case |
|------|----------|----------|
| `TokenType::Access` | 15 minutes | Short-lived API access |
| `TokenType::Refresh` | 7 days | Long-lived refresh tokens |
| `TokenType::Reset` | 1 hour | Password reset links |
| `TokenType::Verify` | 24 hours | Email verification |
| `TokenType::Custom(u64)` | Custom | Your own use cases |

### Claims Structure

```rust
pub struct Claims {
    pub sub: String,        // Subject (user ID)
    pub exp: usize,         // Expiration timestamp
    pub iat: usize,         // Issued at timestamp
    pub jti: String,        // Unique token ID
    pub iss: Option<String>, // Issuer
    pub aud: Option<String>, // Audience
    pub nbf: Option<usize>,  // Not before timestamp
    pub custom: serde_json::Map<String, serde_json::Value>, // Custom claims
}
```

## Usage Examples

### Generating Tokens

```rust
// Generate a simple token
let token = jwt.generate("user-123", TokenType::Access)?;

// Generate with custom expiration
let token = jwt.generate_exp_token("user-123", 1234567890)?;

// Generate with custom claims
let claims = Claims::new("user-123")
    .with_issuer("my-app")
    .with_custom("role".to_string(), json!("admin"));

let token = jwt.generate_with_claims(claims, TokenType::Access)?;

// Generate both access and refresh tokens
let (access, refresh) = jwt.generate_pair("user-123")?;
```

### Verifying Tokens

```rust
// Normal verification with all checks
let claims = jwt.verify(&token)?;

// Quick check (returns bool)
let is_valid = jwt.verify_token(&token);

// Verify without expiration check
let claims = jwt.verify_without_expiry(&token)?;

// Extract claims without validation (debug only)
let claims = jwt.peek_claims(&token);
```

### Token Management

```rust
// Refresh access token using refresh token
let new_access = jwt.refresh_access(&refresh_token)?;

// Revoke a token
jwt.revoke_token(&token)?;

// Revoke by JTI (Token ID)
jwt.revoke_by_jti(&claims.jti, claims.exp)?;

// Check if token is revoked
let is_revoked = jwt.is_revoked(&claims.jti);
```

### Extracting Information

```rust
// Extract various claims without full verification
let subject = jwt.extract_subject(&token);
let expiry = jwt.get_token_expiry(&token);
let jti = jwt.get_token_jti(&token);
let issuer = jwt.get_token_issuer(&token);
let audience = jwt.get_token_audience(&token);
```

## Advanced Configuration

### RSA/ECDSA/EdDSA Support

```rust
// RSA
let private_key = std::fs::read("private.pem")?;
let public_key = std::fs::read("public.pem")?;
let jwt = JwtService::new_rs256(&private_key, &public_key)?;

// ECDSA P-256
let jwt = JwtService::new_ecdsa_p256(&private_key, &public_key)?;

// Ed25519
let jwt = JwtService::new_ed25519(&private_key, &public_key)?;
```

### Custom Validation

```rust
let jwt = JwtService::new_hs256("secret")
    .with_leeway(60)           // 60 seconds clock skew tolerance
    .disable_exp_validation()  // Disable expiration checks
    .with_issuer("my-app")
    .with_audience("my-api");
```

### Custom Token Duration

```rust
// 1 hour token
let token = jwt.generate("user-123", TokenType::Custom(3600))?;
```

## Error Handling

```rust
match jwt.verify(&token) {
    Ok(claims) => {
        // Valid token
    }
    Err(JwtError::TokenExpired) => {
        // Handle expired token
    }
    Err(JwtError::InvalidSignature) => {
        // Handle invalid signature
    }
    Err(JwtError::TokenRevoked) => {
        // Handle revoked token
    }
    Err(e) => {
        // Handle other errors
    }
}
```

## Blacklist Management

```rust
// Create with custom cleanup interval
let blacklist = TokenBlacklist::new(600); // 10 minutes

// Check blacklist stats
let count = blacklist.len();
let empty = blacklist.is_empty();

// Add to service
let jwt = JwtService::new_hs256("secret")
    .with_blacklist(blacklist);
```

## Best Practices

1. **Secret Management**: Use strong secrets (32+ bytes) and store them securely
2. **Token Expiration**: Use appropriate token lifetimes for your use case
3. **Blacklisting**: Always enable blacklisting for production applications
4. **Algorithm Selection**: Use HS256 for simple setups, RSA/ECDSA for distributed systems
5. **Error Handling**: Always handle `JwtError` variants appropriately
6. **Leeway**: Set some leeway (5-60 seconds) to handle clock skew

### 7.2 OTP Challenge Routines
```rust
let generated_otp = otp_service.generate_otp("user_12", 6, 300).await; 

// Secure verification phase
match otp_service.verify_otp("user_12", &generated_otp).await {
    Ok(true) => println!("OTP validation matching successful."),
    Ok(false) => println!("Verification complete: Incorrect client OTP sequence entry."),
    Err(e) => eprintln!("Internal Redis connectivity issue verifying OTP: {}", e),
}

```
### 7.3 Link Lifetime Token Expirations
```rust
let link_token = linkv_service.generate_token("user_12", 300)
    .map_err(|_| actix_web::error::ErrorInternalServerError("Link payload generation failed"))?;

if linkv_service.verify_token(&link_token).unwrap_or(false) {
    println!("Action URL execution approved.");
}

```
### 7.4 Safe Email Senders
```rust
#[get("/test/email")]
async fn test_email(email_service: web::Data<EmailService>) -> Rsp {
    let data = EmailData {
        to: "receiver@example.com".to_string(),
        subject: "Production System Alert".to_string(),
        body: "Automated report log payload delivery text content.".to_string(),
    };
    
    match email_service.send_email(&data).await {
        Ok(_) => http_ok("Mail dispatch queue execution tracking success."),
        Err(e) => http_bad(&format!("Mail deliverability system failure: {}", e)),
    }
}

```
## 8. Cryptography Services
Enforces password security using asynchronous Argon2/Bcrypt implementations and symmetric transformations.
```rust
#[get("/test/crypto")]
async fn test_crypto(crypto: web::Data<CryptoService>) -> Result<Rsp, actix_web::Error> {
    // 1. Symmetric Encryption
    let secret_payload = "Sensitive Data Entry Payload";
    let encrypted_bundle = crypto.encrypt_text(secret_payload)
        .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;
        
    let decrypted_raw = crypto.decrypt_text(&encrypted_bundle.encrypted_text)
        .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;
    
    // 2. High-Speed Hashing (Oneway Mapping checks)
    let checksum = crypto.sha256_hash("file_string_stream")
        .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;
    
    // 3. Async Secure Work-Factored Password Hashing
    let registration_pass = "user_plain_text_password";
    let hashed_storage_entry = crypto.hash_data(registration_pass).await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;
        
    let login_match = crypto.verify_hash(registration_pass, &hashed_storage_entry.hash).await
        .unwrap_or(false);
    
    if login_match {
        Ok(http_ok("Cryptographic validation check passed successfully."))
    } else {
        Ok(http_bad("Invalid credentials provided."))
    }
}

```

## 9. Error Handling

Streamline error handling in your HTTP routes using the built-in `try_catch!` macro. This intercepts failures and automatically formats the error response for the client (not production ready yet).

```rust
// Easy, Better & Clean Error Handling will be coming soon!
use actix_web::{get};
use fr_rust::prelude::*;

#[get("/hello")]
async fn do_something() -> Rsp {
    // Evaluates the future; if it fails, immediately returns the custom error message to the client
    let data = try_catch!(
        try my_async_function().await,
        catch error => {
            println!("Failed to process request data: {:?}", error);
            return http_bad("Error occurred!")
        }
    );
    // If Everything is fine:
    send_str("Hello, World!")
}
```

## Examples

Here are some projects demonstrating how to use this library:

* 👤 [Account Handling Routes](https://github.com/sayed-anower/fr-acc) – An advanced user account management demo featuring signup, login, and authentication flows.
* 💬 [Chat Server Handling](https://github.com/sayed-anower/chatapp) – A full-scale, production-grade chat server featuring multi-server support powered by Redis.

## Production Security Review Checklist
> [!IMPORTANT]
>  1. **No Runtime Panics:** Ensure .unwrap() and .expect() calls are entirely eliminated from request lifecycles to protect server runtime availability.
>  2. **Environment Isolation:** Do not hardcode secret string keys or connection strings. Always source them dynamically via env_var.
>  3. **Memory Limits:** Always configure the WsManager channels with reasonable bounds (e.g., 128) to prevent memory allocation exhaustion from slow client connections.
> 
 * **Encountered an implementation bug?** Please file an explicit Issue Tracker ticket or submit an isolated Pull Request fix.
 * **Inquiries or Corporate Support Requests:** Contact our engineering desk at: sayed.anower.17.2@gmail.com
