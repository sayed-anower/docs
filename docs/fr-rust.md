# fr-rust Documentation

**fr-rust** is a comprehensive, high-performance web utility ecosystem built on top of Actix-Web. It provides production-ready, out-of-the-box support for DDoS protection, standardized HTTP responses, room-based WebSockets, Redis connection pooling, secure cryptography, and essential authentication services (JWT, OTP, Link Verification, Email).

---

## 1. Foundation & Initialization
The framework relies on a shared, thread-safe application state and a highly declarative routing configuration.
### 1.1 Prerequisites
Add the ecosystem dependencies to your Cargo.toml. fr-rust requires seamless interoperability with Tokio and Actix async runtimes.
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
### 1.2 Application State & Routing
Instead of manual App::new() closures, fr-rust uses a macro-driven approach (run_server!) to inject state and configure scopes.
```rust
use fr_rust::prelude::*;

// 1. Define Thread-Safe Application State
#[derive(Clone)]
pub struct AppState {
    pub email_service: EmailService,
    pub pool: DbPool,
    pub redis: RedisManager,
    pub crypto_service: CryptoService,
    pub otp_service: OtpService,
    pub linkv_service: LinkV,
    pub jwt: JwtService,
    pub ws: WsManager,
}

// 2. Configure Scopes and Guards
fn config(cfg: &mut ServiceConfig) {
    get!("/health", health_check);
    post!("/users", create_user);

    // Guarded Scope
    scope!("/admin", guard(::actix_web::guard::Header("admin", "true")), {
        get!("/stats", get_stats);
        post!("/users", admin_create_user);
    });
}

```
### 1.3 Server Bootstrapping & DDoS Shield
The web_main macro initializes the Tokio runtime. You must instantiate your services, configure the DdosShield middleware, and bind the server.
```rust
#[web_main]
async fn main() -> Main {
    load_env();

    // Configure Token-Bucket Rate Limiting
    let ddos_shield = DdosShield::builder()
        .max_requests(100)
        .window_secs(60)
        .ban_duration_secs(3600)
        .block_agent("badbot")
        .cleanup_interval_secs(300)
        .max_ip_records(5000)
        .build();

    // Initialize Services (Error mapping omitted for brevity)
    let jwt = JwtService::new(env_var("JWT_SECRET"));
    let pool = DbPool::new(&env_var("DATABASE_URL"), 3).unwrap();
    let redis = RedisManager::new(&env_var("REDIS_URL")).unwrap();
    
    let app_state = AppState { /* ... */ };

    // Execute Server
    run_server!(
        state: app_state,
        config: config,
        addr: "0.0.0.0:8080",
        app: {
            .wrap(ddos_shield)
            .wrap(actix_web::middleware::Logger::default())
            .app_data(web::PayloadConfig::new(10 * 1024 * 1024)) // 10MB Limit
        },
        server: {
            .workers(num_cpus::get() * 2)
            .shutdown_timeout(120)
        }
    );
}

```
## 2. The Unified Response Matrix
fr-rust abstracts raw HttpResponse construction into highly optimized, allocation-aware helper functions.
### 2.1 Standard Data Responses
| Helper | HTTP | Payload Type | Allocation Profile | Ideal Use Case |
|---|---|---|---|---|
| http_ok_static(msg) | 200 | &'static str | Zero-allocation | Health checks, constants. |
| send_json(data) | 200 | impl Serialize | Serde serialization | Type-safe struct returns. |
| http_ok_json(data) | 200 | serde_json::json! | Dynamic allocation | Ad-hoc or dynamic JSON. |
| http_bad_static(msg) | 400 | &'static str | Zero-allocation | Standard validation failures. |
| http_bad_json(data) | 400 | JSON | Dynamic allocation | Structured API error bodies. |
### 2.2 Advanced & Status-Specific Responses
| Helper | HTTP | Purpose |
|---|---|---|
| http_created(loc) | 201 | Resource creation. Returns a Location header. |
| http_no_content() | 204 | Successful updates/deletes with no body needed. |
| http_unauthorized() | 401 | Missing or invalid authentication credentials. |
| http_forbidden(msg) | 403 | Insufficient permissions for the requested resource. |
| http_not_found(msg) | 404 | Target resource does not exist in the datastore. |
| http_too_many_requests(s) | 429 | Rate limit exceeded. Returns Retry-After: {s}. |
### 2.3 Streaming, Files, and Compression
File handling in fr-rust utilizes OS-level optimizations (like mmap where applicable) to reduce memory overhead.
```rust
// 1. Static File Serving (Zero-copy with ETag/Cache handling)
#[get("/static/{file}")]
async fn serve_static(req: HttpRequest, path: web::Path<String>) -> Result<NamedFile, Error> {
    send_file_fast(&format!("./static/{}", path.into_inner()), &req).await
}

// 2. Range Requests (For Video/Audio Streaming)
async fn stream_media(req: HttpRequest, path: web::Path<String>) -> Result<HttpResponse, Error> {
    let range = req.headers().get(header::RANGE).and_then(|h| h.to_str().ok());
    send_file_range(&format!("./media/{}", path.into_inner()), range).await
}

// 3. Compression (Brotli for Text/JSON, LZ4 for Real-time speed)
async fn get_compressed_data() -> HttpResponse {
    let raw_data = generate_large_json();
    http_brotli(&raw_data, 6) // Quality 6 offers optimal size/speed ratio
}

```
### 2.4 Multipart Upload Implementation
```rust
// Streaming upload prevents memory exhaustion on large files
async fn upload_handler(mut payload: actix_multipart::Multipart) -> Result<HttpResponse, Error> {
    let files = upload_streaming(payload, "./uploads").await?;
    http_ok_json(json!({ "status": "success", "files": files }))
}

```
## 3. Real-Time Communications: WsManager
WsManager bypasses the traditional Actix actor model in favor of a thread-safe, asymmetric channel mapping architecture.
### 3.1 WebSocket Handshake & Event Loop
```rust
#[get("/ws/{user_id}")]
async fn ws_handler(
    req: HttpRequest,
    body: web::Payload,
    ws_manager: web::Data<WsManager>,
    path: web::Path<String>,
) -> Result<HttpResponse, actix_web::Error> {
    let user_id = path.into_inner();

    // Establish a bounded async channel to prevent backpressure (depth: 128)
    let (tx, mut rx) = tokio::sync::mpsc::channel::<String>(128);

    // Safely execute the HTTP -> WS Upgrade
    let (res, mut session, mut msg_stream) = match actix_ws::handle(&req, body) {
        Ok(parts) => parts,
        Err(e) => return Ok(http_bad(&format!("Handshake failed: {}", e))), 
    };

    // Register channel transmitter to the global state
    ws_manager.register(&user_id, tx);

    // Note: The async read/write event loop logic resides here
    Ok(res)
}

```
### 3.2 WsManager Capabilities
```rust
// Room Operations
ws_manager.join_room(&user_id, "trading_desk");
ws_manager.leave_room(&user_id, "trading_desk");

// Targeted Pub/Sub Propagation
ws_manager.msg_user(&user_id, "Order executed.");
ws_manager.msg_room("trading_desk", UserMsg::new("System", "trading_desk", "Market Open"));
ws_manager.broadcast("Server maintenance in 5 minutes.");

// Lifecycle Management
ws_manager.drop_user(&user_id);
ws_manager.drop_room("trading_desk");

```
## 4. Data Persistence & Caching Layers
### 4.1 Non-Blocking Database Engine (DbPool)
Direct execution wrappers mapped across pooled connections.
```rust
async fn process_user(pool: web::Data<DbPool>) -> Result<HttpResponse, Error> {
    // Parameterized DML Execution
    pool.execute(
        "INSERT INTO users (name, role) VALUES ($1, $2);", 
        &[&"Alice", &"admin"]
    ).await.map_err(ErrorInternalServerError)?;
    
    // Optional Row Fetching
    match pool.query_opt("SELECT id FROM users WHERE name = $1;", &[&"Alice"]).await {
        Ok(Some(row)) => Ok(send_json(row.get::<_, i32>("id"))),
        Ok(None) => Ok(http_not_found("User does not exist.")),
        Err(e) => Ok(http_server_error(&e.to_string())),
    }
}

```
### 4.2 Coordinated Redis Architecture
Managed via deadpool_redis to handle distributed caching and event streaming.
```rust
async fn process_redis_pipeline(redis: web::Data<RedisManager>) -> Result<(), Box<dyn std::error::Error>> {
    let mut conn = redis.get_connection().await?;

    // KV Cache Mutation
    let _: () = redis::AsyncCommands::set(&mut conn, "session_key", "active").await?;
    
    // Pub/Sub Broadcast
    let _: () = redis::AsyncCommands::publish(&mut conn, "global_events", "Data synced").await?;

    // Streaming Event Consumer
    let mut stream = redis.subscribe("global_events").await?;
    while let Some(msg) = futures_util::StreamExt::next(&mut stream).await {
        let payload: String = msg.get_payload()?;
        println!("Intercepted: {}", payload);
    }
    
    Ok(())
}

```
## 5. Security & Authentication Ecosystem
### 5.1 JSON Web Tokens (JwtService)
A highly configurable JWT engine supporting multiple algorithms (HS256, RS256, ES256, EdDSA), built-in blacklisting, and multiple token variants.
**Setup & Generation:**
```rust
// Initialize with Sharded Blacklist (Cleans up every 300s)
let blacklist = TokenBlacklist::new(300);
let jwt = JwtService::new_hs256("secret-key")
    .with_issuer("fr-rust-core")
    .with_blacklist(blacklist);

// Generate Standard Access Token (15m expiry)
let token = jwt.generate("user_99", TokenType::Access)?;

// Generate Custom Claims
let claims = Claims::new("user_99")
    .with_custom("clearance_level".to_string(), serde_json::json!(4));
let token = jwt.generate_with_claims(claims, TokenType::Access)?;

```
**Verification & Revocation:**
```rust
// Verify Token
match jwt.verify(&token) {
    Ok(claims) => println!("Valid JWT for User: {}", claims.sub),
    Err(JwtError::TokenExpired) => println!("Token lifetime exceeded."),
    Err(JwtError::TokenRevoked) => println!("Token exists on blacklist."),
    Err(_) => println!("Signature validation failed."),
}

// Revoke Token manually via JTI (JWT ID)
jwt.revoke_token(&token)?;

```
### 5.2 Cryptography Service
Asynchronous hashing and symmetric encryption to prevent thread-blocking during heavy cryptographic workloads.
```rust
async fn secure_password(crypto: web::Data<CryptoService>) -> HttpResponse {
    let raw_pass = "user_input_password";
    
    // Async Argon2/Bcrypt generation
    let hashed = crypto.hash_data(raw_pass).await.unwrap();
    
    // Async Verification
    let is_valid = crypto.verify_hash(raw_pass, &hashed.hash).await.unwrap_or(false);
    
    // Symmetric AES-256 Encryption for sensitive fields
    let encrypted = crypto.encrypt_text("SSN-123-456").unwrap();
    let decrypted = crypto.decrypt_text(&encrypted.encrypted_text).unwrap();
    
    http_ok("Crypto routines validated.")
}

```
### 5.3 OTP & Magic Links
```rust
// OTP Generation (Stored in Redis, valid for 300s)
let otp = otp_service.generate_otp("user_12", 6, 300).await; 
let is_valid = otp_service.verify_otp("user_12", &otp).await.unwrap_or(false);

// Link Verification (JWT-based magic links)
let magic_link_token = linkv_service.generate_token("user_12", 300).unwrap();
let is_link_valid = linkv_service.verify_token(&magic_link_token).unwrap_or(false);

```
## 6. External Integrations
### 6.1 Email Dispatch (EmailService)
A robust SMTP wrapper configured via the AppState.
```rust
async fn dispatch_alert(email_service: web::Data<EmailService>) -> HttpResponse {
    let data = EmailData {
        to: "admin@company.com".to_string(),
        subject: "CRITICAL: Database Latency".to_string(),
        body: "Connection pool saturation detected. Please scale workers.".to_string(),
    };
    
    match email_service.send_email(&data).await {
        Ok(_) => http_ok("Alert dispatched successfully."),
        Err(e) => http_server_error(&format!("SMTP Failure: {}", e)),
    }
}

```
## 7. Architectural Best Practices
To extract maximum performance from the fr-rust ecosystem, strictly adhere to the following paradigms:
 * **String Allocation:** Always prioritize http_bad_static("...") over http_bad(&format!("...")). String allocation on the critical hot path of error handling creates unnecessary heap pressure.
 * **Response Bodies:** Never use HttpResponse::Ok().body(data) for large files. Always utilize send_file_fast to leverage zero-copy mmap capabilities.
 * **Compression Headers:** Rely on Brotli (http_brotli) for JSON APIs where bandwidth is the bottleneck, but switch to LZ4 (http_lz4) for internal microservices where CPU decompression speed is critical.
 * **Status Codes:** Utilize the semantic helpers (http_no_content(), http_created()) rather than returning a 200 OK with a generic JSON status message. Accurate HTTP statuses allow edge proxies and load balancers to route correctly.

## Production Security Review Checklist
> [!IMPORTANT]
>  1. **No Runtime Panics:** Ensure .unwrap() and .expect() calls are entirely eliminated from request lifecycles to protect server runtime availability.
>  2. **Environment Isolation:** Do not hardcode secret string keys or connection strings. Always source them dynamically via env_var.
>  3. **Memory Limits:** Always configure the WsManager channels with reasonable bounds (e.g., 128) to prevent memory allocation exhaustion from slow client connections.
> 
 * **Encountered an implementation bug?** Please file an explicit Issue Tracker ticket or submit an isolated Pull Request fix.
 * **Inquiries or Corporate Support Requests:** Contact our engineering desk at: sayed.anower.17.2@gmail.com