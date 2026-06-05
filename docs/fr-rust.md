# fr-rust Documentation

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
use actix_web::{App, HttpServer, web::Data as AppData, web};

#[fr_rust::main]
async fn main() -> MainRlt {
    // 1. Load Environment Variables safely
    load_env();

    // 2. Configure DDoS Shield Middleware
    let ddos_shield = DdosShield::builder()
        .max_requests(5)          // Max requests per window
        .window_secs(1)           // Time window (1 second)
        .ban_duration_secs(20)    // Ban duration for violators
        .block_agent("malicious-bot")
        .allow_missing_ua(false)
        .build();

    // 3. Initialize Shared State & Services with explicit error fallback
    let jwt_secret = env_var("JWT_SECRET");
    let jwt = Jwt::new(jwt_secret);
    
    let email_config = EmailConfig {
        smtp_host: env_var("SMTP_HOST"),
        smtp_port: env_var("SMTP_PORT").parse().map_err(|_| actix_web::error::ErrorInternalServerError("Invalid SMTP_PORT"))?,
        smtp_user: env_var("SMTP_USER"),
        smtp_pass: env_var("SMTP_PASS"),
        from_name: env_var("FROM_NAME"),
        from_email: env_var("FROM_EMAIL"),
    };
    let email_service = EmailService::new(email_config)
        .map_err(|_| actix_web::error::ErrorInternalServerError("Failed to initialize Email Service"))?;
    
    let pool = DbPool::new(env_var("DATABASE_URL"));
    let redis = RedisManager::new(&env_var("REDIS_URL"))
        .map_err(|_| actix_web::error::ErrorInternalServerError("Failed to connect to Redis"))?;
    
    let key = env_var("AES_KEY");
    let key_bytes: &[u8; 32] = key.as_bytes().try_into()
        .map_err(|_| actix_web::error::ErrorInternalServerError("AES_KEY must be exactly 32 bytes"))?;
    let crypto_service = CryptoService::new(key_bytes)
        .map_err(|_| actix_web::error::ErrorInternalServerError("Failed to initialize Crypto Service"))?;
    
    let otp_service = OtpService::new(OtpConfig {
        secret: env_var("KEY"),
        crypto: crypto_service.clone(),
        redis: redis.clone(),
    });
    
    let linkv_service = LinkV::new(LinkVConfig {
        secret: env_var("KEY"),
        crypto: crypto_service.clone(),
        redis: redis.clone(),
        jwt: jwt.clone()
    });
    
    let ws = WsManager::new(WsConfig { server: 1, redis: redis.clone() });

    // 4. Start the HTTP Server
    let ip = env_var_or_default("IP", "0.0.0.0");
    let port = env_var_or_default("PORT", "8080");
    let address = format!("{}:{}", ip, port);
    println!("Starting production server at http://{}", address);
    
    HttpServer::new(move || {
        App::new()
            .app_data(AppData::new(email_service.clone()))
            .app_data(AppData::new(pool.clone()))
            .app_data(AppData::new(redis.clone()))
            .app_data(AppData::new(crypto_service.clone()))
            .app_data(AppData::new(otp_service.clone()))
            .app_data(AppData::new(linkv_service.clone()))
            .app_data(AppData::new(jwt.clone()))
            .app_data(AppData::new(ws.clone()))
            .configure(app_config)
            .wrap(ddos_shield.clone())
    })
    .bind(address)?
    .run()
    .await
}

pub fn app_config(cfg: &mut web::ServiceConfig) {
    cfg.service(index_file);
}

```
## 2. Framework Type Aliases
**fr-rust** provides semantic type wrappers over base actix-web engines to accelerate development flow:
| Type Alias | Native Equivalent | Context/Description |
|---|---|---|
| Rsp | HttpResponse | Default wrapper for outbound server responses. |
| Rqs | HttpRequest | Standard request metadata representation. |
| Rlt | Result<(), actix_web::Error> | Universal Fallible Internal Actix action. |
| MainRlt | std::io::Result<()> | Standard runtime entry return contract. |
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
| Response Macro/Helper | HTTP Status Code | Payload Type |
|---|---|---|
| http_ok(msg) | 200 OK | Plain text / String slice |
| http_bad(msg) | 400 Bad Request | Error Message String slice |
| send_str(msg) | 200 OK | Evaluated raw custom String string |
| send_json(data) | 200 OK | Structs/Vectors deriving serde::Serialize |
| http_ok_json(data) | 200 OK | Ad-hoc serde_json::json! structures |
| http_bad_json(data) | 400 Bad Request | JSON-formatted error responses |
#### Safe Route Implementation:
```rust
use serde::Serialize;
use std::collections::HashMap;

#[derive(Serialize)]
struct User {
    id: u64,
    name: String,
}

#[get("/test/responses/{type}")]
async fn test_responses(path: web::Path<String>) -> Rsp {
    match path.into_inner().as_str() {
        "ok" => http_ok("Operation completed successfully."),
        "bad" => http_bad("The request parameters were invalid."),
        "str" => send_str("Direct string text stream"),
        "json_struct" => send_json(User { id: 1, name: "Sayed".to_string() }),
        "json_vec" => send_json(vec![1, 2, 3]),
        "json_macro_bad" => http_bad_json(serde_json::json!({"success": false, "reason": "Unauthorized action"})),
        "json_map" => {
            let mut map = HashMap::new();
            map.insert("name", "Sayed");
            http_ok_json(map)
        },
        _ => http_bad("Unknown payload route variant requested.")
    }
}

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
```rust
// Token issuance with standard claims duration limits
let current_time = std::time::SystemTime::now()
    .duration_since(std::time::UNIX_EPOCH)?
    .as_secs() as usize;
    
let expiry_timestamp = current_time + 3600; // 1-hour validity window

// Generate Expiring JWT
let token = jwt.generate_exp_token("user_9821", expiry_timestamp)
    .map_err(|_| actix_web::error::ErrorInternalServerError("Token processing failure"))?;

// Validation verification check
if jwt.verify_token(&token) {
    let parsed_id = jwt.parse_token(&token)
        .map_err(|_| actix_web::error::ErrorUnauthorized("Corrupted token context"))?;
    println!("Authenticated context for: {}", parsed_id);
}

```
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
## 9. General Standalone Utilities
Quick-acting helper utilities for initialization phases or automated script contexts:
```rust
// Thread-blocking user IO capture for configuration terminals
let user_input = input("Enter administrative initialization code: ");

// Generate secure salt vectors or cryptographically random tokens
let secure_hex_key = generate_key(64); 
println!("Entropy block string key: {}", secure_hex_key);

```

## 10. Error Handling

Streamline error handling in your HTTP routes using the built-in `http_error!` macro. This intercepts failures and automatically formats the error response for the client.

```rust
use actix_web::{get, HttpResponse};
use fr_rust::prelude::*;

#[get("/hello")]
async fn do_something() -> Rsp {
    // Evaluates the future; if it fails, immediately returns the custom error message to the client
    let data = http_error!(
        my_async_function().await,
        "Failed to process request data"
    );
    // If Everything is fine:
    send_str("Hello, World!")
}
```

## Examples

Here are some projects demonstrating how to use this library:

* 👤 [Account Handling Routes](https://github.com/sayed-anower/fr-acc) – An advanced user account management demo featuring signup, login, and authentication flows.
* 💬 [Chat Server Handling](https://github.com/sayed-anower/chatapo) – A full-scale, production-grade chat server featuring multi-server support powered by Redis.

## Production Security Review Checklist
> [!IMPORTANT]
>  1. **No Runtime Panics:** Ensure .unwrap() and .expect() calls are entirely eliminated from request lifecycles to protect server runtime availability.
>  2. **Environment Isolation:** Do not hardcode secret string keys or connection strings. Always source them dynamically via env_var.
>  3. **Memory Limits:** Always configure the WsManager channels with reasonable bounds (e.g., 128) to prevent memory allocation exhaustion from slow client connections.
> 
 * **Encountered an implementation bug?** Please file an explicit Issue Tracker ticket or submit an isolated Pull Request fix.
 * **Inquiries or Corporate Support Requests:** Contact our engineering desk at: sayed.anower.17.2@gmail.com
