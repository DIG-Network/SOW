**Technical Specification: Custom Rust-Based In-Memory Cache Server**  
*(Shared Across All DIG Node Submodules Running in Separate Processes)*

---

## **1. Introduction and Background**  
The DIG Node platform comprises multiple submodules running as separate processes, all requiring fast, centralized access to shared data. Traditional solutions like Redis are avoided because we want a **single-click installer** bundling a custom, lightweight in-memory cache. This specification details how to build a **Rust-based in-memory cache server** that can be run alongside the DIG Node services to provide quick, concurrent key-value storage with optional TTL, memory-based eviction, and a simple HTTP interface.

---

## **2. Objectives**  
1. **Centralized Shared Cache**  
   - Provide a shared in-memory store accessible to all DIG Node submodules, which operate as separate processes.  
   - Ensure minimal overhead in communication and maintenance.

2. **HTTP Endpoints**  
   - **GET /:key** to retrieve cached data.  
   - **POST /:key** to store data, accepting an optional TTL in the request body.

3. **TTL and Eviction**  
   - Support automatic expiration for items with a defined TTL.  
   - Implement an **oldest-first eviction** strategy when the cache approaches memory limits.

4. **One-Click Installer Compatibility**  
   - Build a **standalone Rust binary** that can be packaged within a single installer for easy deployment.  
   - Avoid external dependencies like Redis or separate services.

5. **Continuous Integration**  
   - Automate builds that produce a Docker image, pushing it to DockerHub for container-based deployments.

---

## **3. Scope of Work**  
### **3.1. In-Memory Data Store**  
- **Data Structure**:  
  - Use a **thread-safe** data store, e.g., a `HashMap` or similar structure wrapped in locks or using lock-free techniques.  
  - Each entry tracks:  
    - **Value**: The cached data (string or bytes).  
    - **Last Modified**: A timestamp indicating when the value was last updated.  
    - **TTL (optional)**: If present, the key will expire at the specified time.

- **Concurrency**:  
  - Must handle multiple simultaneous GET and SET requests from different DIG Node submodules.  
  - Employ efficient concurrency primitives (e.g., `parking_lot::RwLock` or `dashmap`) to allow high read throughput and safe writes.

### **3.2. HTTP Server Interface**  
- **GET /:key**  
  - Returns the associated value if found and not expired.  
  - Returns a 404 (or other appropriate HTTP status) if the key is missing or expired.

- **SET /:key**  
  - Accepts a JSON body with the shape:
    ```json
    {
      "value": "...",
      "ttl": 3600
    }
    ```
    - `ttl` is optional and specified in seconds.  
  - Overwrites or creates the key-value pair.  
  - Records the current timestamp as *Last Modified*.

### **3.3. TTL Handling**  
- **Expiration Checks**:  
  - A periodic task or on-demand logic removes expired entries from the cache.  
  - If a TTL is not specified, the value remains until eviction or manual deletion.

### **3.4. Memory Monitoring and Eviction**  
- **Memory Threshold**:  
  - Configurable upper limit (in MB/GB or system-based checks).  
- **Oldest-First Eviction**:  
  - When usage nears the threshold, remove the **oldest items** based on their last modified timestamps until memory returns below the threshold.

---

## **4. Technical Requirements**  
1. **Language**: Rust  
2. **HTTP Framework**: A popular Rust framework such as **Axum**, **Actix**, or **Warp**.  
3. **Concurrency and Synchronization**:  
   - Possible use of **parking_lot** locks or **DashMap** for high-performance concurrency.  
   - Must handle multiple client requests from separate processes with minimal contention.  
4. **Docker Integration**:  
   - Continuous Integration (CI) pipeline compiles the Rust application into a Docker image.  
   - On successful build, automatically push the image to DockerHub.  
5. **One-Click Install**:  
   - The final system includes a **single static binary** for the cache server that can be bundled into the DIG Node installer.  
   - Avoid reliance on external caching services or large dependencies.

---

## **5. Implementation Outline**  
1. **Data Structure**  
   - Create a struct, e.g. `CacheItem`, holding the `value`, optional `expires_at`, and `last_modified`.  
   - Store instances in a concurrent map keyed by strings.  

2. **API Endpoints**  
   1. **GET /:key**  
      - Look up the key in the map; check if expired; return `200` with the value or `404` if missing/expired.  
   2. **SET /:key**  
      - Parse JSON body for `value` and optional `ttl`.  
      - Calculate `expires_at` if `ttl` is present.  
      - Update `last_modified`.  

3. **TTL Enforcement**  
   - A background thread or tokio task periodically scans for expired keys.  
   - Could also lazily remove expired entries upon GET or SET requests.

4. **Memory Usage and Eviction**  
   - Use either OS metrics or a configurable memory limit.  
   - When the cache size is too large or memory usage is near the threshold, remove entries sorted by `last_modified` (oldest first).  

5. **Testing**  
   - Unit tests covering edge cases: missing keys, expired keys, eviction triggers, concurrent GET/SET calls.  
   - Integration tests under load to confirm stable performance and correct eviction behavior.

6. **CI/CD**  
   - Use GitHub Actions, GitLab CI, or similar to:  
     - Build the Rust binary.  
     - Run tests.  
     - Create/push Docker images to DockerHub if tests pass.  

---

## **6. Assumptions**  
1. **Separate Processes**: Each DIG Node submodule will interact via the cache server’s HTTP interface.  
2. **Deployment Environment**: Runs on Linux or a container-based environment where the Docker image can be deployed.  
3. **Memory Constraints**: Adequate RAM is available; the system will not exceed physical memory if configured properly.  
4. **Installer Integration**: The one-click installer team will bundle the final Rust binary as part of the DIG Node package.  

---

**End of Technical Specification**