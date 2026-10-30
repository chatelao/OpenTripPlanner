# Concept: Porting OpenTripPlanner `--load` Functionality to PHP

## 1. Top Goal

Portiere ausschliesslich die Java "--load"-Funktionalität auf PHP, um das extern vorcompilierte
"graph.obj" via HTTP/HTTPS bereitzustellen.

The primary objective is to replace or complement Java-based execution for loading pre-built
OpenTripPlanner (OTP) graph objects (`graph.obj`) with a PHP-based runtime solution. This enables
lightweight HTTP/HTTPS web servers running standard PHP environments (such as Apache, Nginx, or
PHP-FPM) to load pre-compiled graph data, answer transit/routing requests, or serve serialized graph
payloads efficiently without requiring a heavy persistent Java JVM process in resource-constrained
or PHP-native host environments.

---

## 2. Business & Use Cases

### 2.1 Business Goal & Value Proposition

- **Lower Resource Footprint & Hosting Costs**: Hosting full Java JVM instances for OTP can be
  memory-intensive. Translating the graph loading and serving functionality to PHP allows deployment
  on low-cost shared web hosting or serverless/FaaS environments that support PHP.
- **PHP Ecosystem Integration**: Organizations with existing PHP-based web infrastructures (e.g.,
  Symfony, Laravel, WordPress, custom enterprise portals) can natively load and serve graph features
  or proxy transit queries directly within their PHP codebase without calling inter-process Java
  bridges.
- **Decoupled Graph Building and Graph Serving**: Graph compilation remains an expensive offline
  task performed using standard OTP Java tools. The lightweight PHP runtime is strictly dedicated to
  loading the resulting binary artifact (`graph.obj`) and serving requests via standard HTTP/HTTPS
  interfaces.

### 2.2 Key Use Cases

#### UC-1: Direct Graph Binary HTTP/HTTPS Serving & Streaming

- **Actor**: External Client / Downstream Routing Nodes
- **Description**: A client sends an HTTP GET request to download or stream the precompiled
  `graph.obj` (or specific sub-sections/versioned graph assets).
- **Outcome**: PHP loads and streams the serialized binary file with appropriate cache headers (HTTP
  status 200/206 for range requests, ETag, Content-Length) without loading whole files into PHP
  process memory at once.

#### UC-2: Read-Only Graph Unserialization & Endpoint Querying

- **Actor**: Web Application / API Consumer
- **Description**: The PHP application initializes by parsing and deserializing the compiled
  `graph.obj` structure into memory (or using an efficient memory-mapped / cached representation) to
  expose routing or transit lookup endpoints via REST/JSON.
- **Outcome**: Incoming HTTP requests for transit routes or graph metadata are processed directly by
  PHP against the loaded graph model.

#### UC-3: Automated Precompiled Graph Refresh

- **Actor**: Automated CI/CD or Graph Build Pipeline
- **Description**: An external Java process periodically compiles updated GTFS/OSM data into a new
  `graph.obj` and pushes it to a designated storage location monitored by the PHP application.
- **Outcome**: The PHP application invalidates its local cache and loads the new `graph.obj`
  seamlessly for subsequent HTTP/HTTPS requests.

---

## 3. High-Level Architecture & Functional Components

### 3.1 Top-Level Architecture

The system consists of two major operational domains:

1. **Offline Java Graph Builder (Producer)**: Compiles GTFS timetable data and OpenStreetMap street
   networks into a standard `graph.obj` file using existing Java OTP tools.
2. **PHP Graph Loader & HTTP Server (Consumer / Server)**: A PHP runtime application structured with
   top-level functional components responsible for reading, deserializing, caching, and serving
   graph data over HTTP/HTTPS.

```
+------------------------------------+
|  Java OTP Graph Builder (Offline)  |
|  (Outputs serialized `graph.obj`)  |
+------------------------------------+
                  |
                  | HTTP/HTTPS / Storage Transfer
                  v
+-------------------------------------------------------------------+
|                     PHP Graph Server Application                  |
|                                                                   |
|  +-----------------------+      +------------------------------+  |
|  | Graph Storage & Binary|      |  Deserialization & Memory    |  |
|  | Reader Interface      |      |  Representation Layer        |  |
|  +-----------------------+      +------------------------------+  |
|              |                                 |                  |
|              v                                 v                  |
|  +-----------------------+      +------------------------------+  |
|  | Cache & Memory-Map    |      |  HTTP/HTTPS Routing API      |  |
|  | Management Component  |      |  Interface Controller        |  |
|  +-----------------------+      +------------------------------+  |
+-------------------------------------------------------------------+
```

### 3.2 Top-Level Functional Components & Business Interfaces

#### 1. Binary Graph Loader Component (`IGraphBinaryReader`)

- **Responsibility**: Provides efficient file system or stream access to read the serialized
  `graph.obj` file format produced by Java serialization / Hessian serialization.
- **Business Interface**:
  - `openGraphStream(string $filePathOrUrl): StreamInterface`
  - `validateHeader(StreamInterface $stream): GraphHeaderMetadata`
  - `readBlock(StreamInterface $stream, int $offset, int $length): string`

#### 2. Java Serialization Parser Component (`IJavaObjectUnserializer`)

- **Responsibility**: Re-creates Java object graphs or maps serialized Java data structures (graphs,
  vertices, edges, transit indexes) into PHP object models or memory structures.
- **Business Interface**:
  - `unserializeGraph(StreamInterface $stream): GraphEntity`
  - `extractGraphMetadata(StreamInterface $stream): MetadataDTO`

#### 3. Graph Cache & Memory Management Component (`IGraphCacheManager`)

- **Responsibility**: Caches parsed graph entities in fast memory storage (e.g., OPcache, APCu,
  Redis, or memory-mapped files) to avoid parsing multi-megabyte/gigabyte `graph.obj` files on every
  stateless PHP HTTP request.
- **Business Interface**:
  - `storeCachedGraph(string $graphId, GraphEntity $graph): bool`
  - `getCachedGraph(string $graphId): ?GraphEntity`
  - `isCacheValid(string $graphId, string $fileHash): bool`

#### 4. HTTP/HTTPS Service Interface (`IGraphHttpServer`)

- **Responsibility**: Handles incoming HTTP/HTTPS REST/GraphQL requests, managing response headers,
  range requests, binary streaming, and query execution.
- **Business Interface**:
  - `handleGraphDownloadRequest(ServerRequestInterface $request): ResponseInterface`
  - `handleTransitQueryRequest(ServerRequestInterface $request): ResponseInterface`

---

## 4. Evaluation of Major Architectural Choices

For each key technical decision required for this concept, three distinct alternatives were
evaluated.

### 4.1 Choice 1: Strategy for Handling Java Serialization Format in PHP

- **Alternative A: Custom Pure-PHP Java Serialization Unserializer (Chosen)**
  - _Description_: Implement a native PHP parser tailored to read Java `ObjectOutputStream` /
    Hessian binary format and convert `graph.obj` structures into PHP objects or arrays.
  - _Pros_: Zero external C-extension dependencies; runs anywhere PHP runs; full control over object
    mapping.
  - _Cons_: Development effort to implement Java serialization protocol parser in PHP.
- **Alternative B: Shared-Memory / C Extension Wrapper (`php-java-bridge` / FFI to C library)**
  - _Description_: Use PHP FFI or C/C++ extensions to load a lightweight compiled C/Java shared
    library that reads the binary graph.
  - _Pros_: Fast binary parsing execution.
  - _Cons_: Requires server root access to install native extensions; reduces portability on shared
    hosting; bypasses pure PHP requirement.
- **Alternative C: Offline Export to Intermediate Neutral Binary Format (JSON / Protocol Buffers)**
  - _Description_: Modify Java graph builder to output Protocol Buffers or FlatBuffers alongside or
    instead of Java `graph.obj`, then read that with standard PHP Protobuf packages.
  - _Pros_: Easiest parsing in PHP; language-agnostic data structures.
  - _Cons_: Deviates from strict requirement of consuming original `graph.obj` produced by standard
    `--load` path.

_Selection_: **Alternative A** is chosen as the primary approach to maintain pure PHP runtime
compatibility while operating directly on `graph.obj`.

---

### 4.2 Choice 2: PHP Memory & Performance Management Model for Large `graph.obj`

- **Alternative A: Shared In-Memory State via APCu / Memory-Mapped Storage (FFI / Shmop) (Chosen)**
  - _Description_: Load parsed graph structures into shared memory (APCu, shared memory IPC, or
    memory-mapped files via `shmop`/`FFI`), making graph data accessible across stateless PHP worker
    requests.
  - _Pros_: Eliminates per-request parsing overhead; low latency HTTP response times.
  - _Cons_: Requires initial warm-up phase when updated graph is deployed.
- **Alternative B: Per-Request On-Demand Parsing with File Streaming**
  - _Description_: Read and parse `graph.obj` on every incoming HTTP request.
  - _Pros_: Simplicity; stateless architecture.
  - _Cons_: High RAM usage and CPU bottleneck on large graph files; unviable for large transit
    graphs under high request volume.
- **Alternative C: External Persistent Storage Daemon / Redis Layer**
  - _Description_: Convert `graph.obj` on load into key-value stores or spatial indexes in
    Redis/Memcached accessed by PHP.
  - _Pros_: Offloads memory management from PHP process manager.
  - _Cons_: Introduces additional infrastructure dependency (Redis daemon) contrary to standalone
    PHP deployment goals.

_Selection_: **Alternative A** is chosen for optimum performance in PHP web application
environments.

---

### 4.3 Choice 3: Execution Runtime & HTTP Delivery Engine

- **Alternative A: Modern PSR-7 / PSR-15 PHP Application Framework (Swoole / Workerman or Standard
  PHP-FPM) (Chosen)**
  - _Description_: Build the PHP application using standard PSR HTTP interfaces (PSR-7
    request/response, PSR-15 handlers) compatible with standard web servers (Nginx/Apache + PHP-FPM)
    or long-running PHP runtimes (Swoole/RoadRunner/Workerman).
  - _Pros_: High flexibility, standard industry architecture, easy testability, supports streaming
    responses for large binary downloads.
  - _Cons_: Slightly higher abstraction layer overhead.
- **Alternative B: Legacy Monolithic CGI Script (`index.php`)**
  - _Description_: Single procedural PHP script that outputs HTTP headers and streams file contents.
  - _Pros_: Simple single file implementation.
  - _Cons_: Poor maintainability, lack of testability, difficult to extend for complex transit query
    interfaces.
- **Alternative C: WebAssembly (Wasm) PHP Runtime**
  - _Description_: Run PHP compiled to WebAssembly on an edge compute provider (e.g. Cloudflare
    Workers / Fastly Wasm).
  - _Pros_: Global edge deployment.
  - _Cons_: Stringent WASM memory limits when processing large `graph.obj` files.

_Selection_: **Alternative A** is chosen for clean separation of concerns, testability, and standard
deployment options.

---

## 5. Summary of Discarded Alternatives

Below is a summary of the alternatives evaluated and discarded during the conceptual design phase:

| Choice Area            | Option                                             | Status    | Key Reason for Rejection                                                                        |
| :--------------------- | :------------------------------------------------- | :-------- | :---------------------------------------------------------------------------------------------- |
| **Java Serialization** | Alternative B (PHP C Extension / FFI)              | Discarded | Requires C compile toolchains and server root access; restricts deployment on shared PHP hosts. |
| **Java Serialization** | Alternative C (Format Conversion to Protobuf/JSON) | Discarded | Bypasses the core goal of direct compatibility with standard `graph.obj` binary files.          |
| **Memory Management**  | Alternative B (Per-Request Parsing)                | Discarded | Severe CPU/RAM performance penalty; non-viable for production web traffic.                      |
| **Memory Management**  | Alternative C (External Redis Layer)               | Discarded | Adds required infrastructure dependencies (Redis cluster) for simple web nodes.                 |
| **HTTP Engine**        | Alternative B (Legacy CGI Script)                  | Discarded | Hard to maintain, unmodular, and unsuitable for professional REST/GraphQL interfaces.           |
| **HTTP Engine**        | Alternative C (Edge WASM PHP)                      | Discarded | RAM limitations in WebAssembly runtimes prevent loading large serialized graph files.           |
