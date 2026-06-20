# fr-search Contentation
**fr-search** is a high-performance, ultra-optimized search engine and document tagging system built in Rust. By leveraging cutting-edge underlying libraries and Meta's FastText model, it delivers lightning-fast search suggestions and highly accurate automated document tagging with minimal resource overhead.
## 1. Quick Start & Prerequisites
### Ecosystem Dependencies
To use **fr-search** effectively, you must pair it with its core storage and runtime foundations. Ensure your Cargo.toml includes the following adjacent dependencies:
```toml
[dependencies]
fr-search = "0.1"
tokio = { version = "1", features = ["full"] }
sonyflake = "0.5"
serde = { version = "1.0", features = ["derive"] }

```
### Subsystem Initialization
Initialize structural state, instantiate runtime connections to machine learning artifacts, and safely build zero-allocation prefix indexes.
```rust
use fr_search::prelude::*;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 1. Compile static/dynamic autocomplete layers
    let keywords = vec!["apple", "banana", "apple", "applied", "application", "cherry"];
    let fst = Fst::new(keywords)?;

    // 2. Hydrate ML components and full-text index configs
    let tagger = Tagger::new("lid.176.ftz")?;
    let searcher = Fts::new(FtsConfig {
        index_path: "./search_index".to_string(),
    })?;

    Ok(())
}

```
## 2. Instant Search Suggestions (FST Subsystem)
The Fst module compiles dictionaries into an ordered, highly compressed directed acyclic graph. This layout guarantees predictable performance characteristics for typeahead mechanics and autocomplete widgets.
### Autocomplete & Prefix-Matching Implementation
```rust
fn run_autocomplete() -> Result<(), Box<dyn std::error::Error>> {
    let keywords = vec!["apple", "banana", "apple", "applied", "application", "cherry"];
    
    // Completely static error handling & creation
    let fst = Fst::new(keywords)?; 

    // Search for matching prefixes
    let results = fst.search("app");
    println!("Results for 'app': {:?}", results);
    // Output: ["apple", "application", "applied"]

    // Handling lookup misses safely
    let empty_results = fst.search("mango");
    println!("Results for 'mango': {:?}", empty_results);
    // Output: []

    Ok(())
}

```
## 3. Intelligent Engine & ML Content Auto-Tagging
The Full-Text Search (Fts) and Tagger modules operate in tandem to create a unified data enrichment and retrieval pipeline. The classification layer abstracts Meta’s .ftz models to assign categories before indexing documents into the inverted search index.
### Advanced Ingestion Pipeline with Mutators
```rust

use serde::{Serialize, Deserialize};

// A helper struct to bundle the internal data before serializing it into Content.content
#[derive(Serialize, Deserialize)]
struct DocumentPayload {
    pub title: String,
    pub body: String,
    pub tags: Vec<String>,
    pub timestamp: i64,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Separate construction of classification and indexing infrastructure
    let tagger = Tagger::new("lid.176.ftz")?; 
    let searcher = Fts::new(FtsConfig {
        index_path: "./search_index".to_string(),
    })?;

    // Inbound payload processing
    let body_text = "Understanding memory layout in Rust is crucial.";
    let mut document_tags = vec!["programming".to_string(), "rust".to_string()];

    // Programmatically extract additional classifications
    let mut tags = tagger.generate_tags(body_text.to_string());
    document_tags.append(&mut tags);

    // [!] At this point, document_tags can be mirrored safely to an RDBMS

    // Helper closure to package our data into the new Content format
    let create_content_payload = |id: &str, tags_list: Vec<String>| -> Result<Content, serde_json::Error> {
        let payload = DocumentPayload {
            title: "Rust Memory Management".to_string(),
            body: body_text.to_string(),
            tags: tags_list,
            timestamp: 1717800000,
        };
        
        Ok(Content {
            id: id.to_string(),
            content: serde_json::to_string(&payload)?, // Converts the struct to a JSON string
        })
    };

    // Core Content Ingestion
    let content_to_add = create_content_payload("1", document_tags.clone())?;
    searcher.add_doc(content_to_add).await?;

    // Overwriting / Updating an existing document
    let content_to_edit = create_content_payload("1", document_tags)?;
    searcher.edit_doc("1".to_string(), content_to_edit).await?;

    // Content Scanning and Query Matching
    let limit = 10;
    let offset = 0;
    let results = searcher.search("keyword", limit, offset)?;

    // Evicting a document from the collection
    searcher.del_doc("1".to_string()).await?;

    Ok(())
}

```
## 4. Distributed Unique Sequence Identifiers
To decouple system synchronization bottlenecks, **fr-search** pairs natively with distributed ID mechanics like Sonyflake. This prevents primary key collisions when executing broad, decoupled node architectures scaled across multiple database partitions and physical web servers.
### 64-Bit ID Allocation
```rust
use sonyflake::Sonyflake;

pub fn generate_cluster_id() -> u64 {
    // Instantiation context (maintain within global or application states)
    let sf = Sonyflake::new().expect("Failed to initialize Snowflake state");

    // Secure, chronological, 64-bit unique sequence emission
    let next_id = sf.next_id().expect("System clock skew or drift detected");

    println!("Generated ID: {}", next_id);
    *next_id
}

```
## 5. RDBMS Synchronization & Data Patterns
While **fr-search** manages volatile index schemas, analytical tasks or critical consistency flows require transactional engines (such as PostgreSQL, MySQL, or SQLite).
### Relational Schema Design
When persisting the generated tags alongside distributed identifiers, map keys cleanly onto BIGINT attributes and structured collections:
```sql
CREATE TABLE articles (
    id BIGINT PRIMARY KEY,          -- Allocated securely via Sonyflake
    title VARCHAR(255) NOT NULL,
    body TEXT NOT NULL,
    tags TEXT[] NOT NULL,           -- Populated using the Tagger output
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

```
### Hydrating Records From Search Node Arrays
Once index searches resolve matching structural numeric identifiers, complete relational mappings should be pulled down atomically using precise array lookups:
```sql
-- Query data matching distributed identifiers harvested from fr-search nodes
SELECT 
    id, 
    title, 
    body, 
    tags 
FROM 
    articles
WHERE 
    id IN (12, 85, 91, 103);

```
## 6. Architectural Trade-offs & Limitations
 * **Opinionated Defaults:** To ensure maximum developer velocity and zero configurations, low-level engine primitives (like tokenizers, stemmers, and BM25 parameter layouts) are preconfigured internally. Power users requiring highly specific, fine-grained control over underlying inverted index files may encounter design boundaries.
 * **Model Files Dependency:** The auto-tagging system demands direct access to Meta's .ftz model file at startup. Missing or malformed asset paths will yield a static failure initialization error.
