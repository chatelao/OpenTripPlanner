# Implementation Roadmap: PHP OpenTripPlanner Graph Loader & Server

## 1. Executive Summary & Business Objectives

### 1.1 Business Goal
**"Portiere ausschliesslich die Java '--load'-Funktionalität auf PHP, um das extern vorcompilierte 'graph.obj' via HTTP/HTTPS bereitzustellen."**

The primary objective of this roadmap is to establish a structured execution plan for porting OpenTripPlanner (OTP) Java `--load` functionality to PHP. By enabling standard PHP runtimes (PHP-FPM behind Nginx/Apache, Workerman, Swoole) to load, stream, cache, and query pre-compiled `graph.obj` binary files, this project achieves:

- **Significantly Reduced Hosting & Infrastructure Costs**: Eliminates the requirement for heavy persistent Java Virtual Machine (JVM) processes on graph-serving nodes.
- **Native PHP Ecosystem Integration**: Enables seamless integration with existing PHP web applications (Laravel, Symfony, custom enterprise portals) without inter-process communication overhead.
- **Decoupled Architecture**: Retains heavy offline graph compilation in Java OTP while offloading lightweight read-only graph loading and HTTP delivery to PHP.

---

## 2. Roadmap Overview & Phases

```
+-----------------------------------------------------------------------------------+
| Phase 1: Core Foundation & Binary Stream Reader (`IGraphBinaryReader`)            |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| Phase 2: Java Serialization Parser & Domain Models (`IJavaObjectUnserializer`)     |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| Phase 3: Graph Cache & Shared Memory Manager (`IGraphCacheManager`)               |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| Phase 4: HTTP/HTTPS Service Controller & Range Streaming (`IGraphHttpServer`)     |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| Phase 5: Integration, Security Hardening & Quality Assurance                      |
+-----------------------------------------------------------------------------------+
```

---

## 3. Phase Specifications & Deliverables

### Phase 1: Core Foundation & Binary Stream Reader

**Goal**: Establish project structure, configuration handling, and low-level binary streaming capability for serialized `graph.obj` files.

#### Key Deliverables
- **Project Scaffold**: Composer package structure adhering to PSR-4 autoloading standards.
- **Configuration Management**: Flexible configuration support via environment variables (`OTP_GRAPH_PATH`, `OTP_CACHE_DRIVER`, `OTP_ENABLE_DOWNLOAD`).
- **`IGraphBinaryReader` Implementation**:
  - `openGraphStream(string $filePathOrUrl): StreamInterface` supporting local files and remote HTTP URIs.
  - `validateHeader(StreamInterface $stream): GraphHeaderMetadata` verifying Java magic bytes (`0xACED 0x0005`).
  - `readBlock(StreamInterface $stream, int $offset, int $length): string` for chunked binary byte-range reading.

#### Success Criteria
- Unit tests verifying magic header validation for valid and invalid binary files.
- Memory-efficient stream reading (< 10 MB RAM peak usage when opening multi-hundred-megabyte streams).

---

### Phase 2: Java Serialization Parser & PHP Domain Models

**Goal**: Parse serialized Java object stream structures into lightweight, read-optimized PHP domain models.

#### Key Deliverables
- **PHP Domain Model Classes (`OTP\Domain\Model\*`)**:
  - `GraphEntity`: Core container for graph metadata, spatial envelopes, vertices, edges, and transit indexes.
  - `Vertex`, `Edge`, `TransitStop`: Lightweight read-only models representing transit nodes and connections.
- **`IJavaObjectUnserializer` Implementation**:
  - Binary parser handling Java serialization handles (`TC_CLASSDESC`, `TC_OBJECT`, `TC_STRING`, `TC_ARRAY`, `TC_REFERENCE`).
  - Class mapping mechanism resolving Java OTP classes (`org.opentripplanner.routing.graph.Graph`, `Vertex`, `Edge`) to corresponding PHP domain objects.
  - `extractGraphMetadata(StreamInterface $stream): MetadataDTO` for lightweight header inspection without parsing full graphs.

#### Success Criteria
- Successful deserialization of sample `graph.obj` artifacts built by Java OTP.
- Full unit test coverage for type descriptor resolution and object reconstruction.

---

### Phase 3: Graph Cache & Shared Memory Manager

**Goal**: Eliminate per-request parsing overhead by persisting unserialized graph representations in high-performance shared memory storage.

#### Key Deliverables
- **`IGraphCacheManager` Implementation**:
  - Cache drivers for **APCu**, **Shared Memory (`shmop`)**, and **OPcache memory-mapped file arrays**.
  - `storeCachedGraph(string $graphId, GraphEntity $graph): bool` and `getCachedGraph(string $graphId): ?GraphEntity`.
  - Automated cache validation via `isCacheValid(string $graphId, string $fileHash)` comparing file checksums/mtime on disk.
  - Hot-swapping and invalidation trigger (`invalidate(string $graphId)`).

#### Success Criteria
- Millisecond-level cache retrieval for pre-parsed graphs across concurrent stateless PHP requests.
- Automatic cache cache-invalidation and reload when updated `graph.obj` files are deployed.

---

### Phase 4: HTTP/HTTPS Service Controller & Range Streaming

**Goal**: Expose PSR-7 / PSR-15 compliant HTTP endpoints for binary file downloads with range support and read-only graph queries.

#### Key Deliverables
- **`IGraphHttpServer` Implementation**:
  - `handleGraphDownloadRequest`: HTTP 200/206 streaming binary file delivery. Full support for `Range`, `If-Range`, and `Accept-Ranges: bytes` headers using fixed buffer streaming (8 KB chunks).
  - `handleMetadataRequest`: JSON endpoint (`GET /graph/metadata`) exposing graph spatial bounds, build timestamps, and vertex/edge counts.
  - `handleTransitQueryRequest`: REST endpoints (`GET /graph/vertices`, `GET /graph/edges`) for read-only transit inspections.

#### Success Criteria
- Successful byte-exact `curl` downloads of multi-gigabyte `graph.obj` files via HTTP range requests.
- Low peak RAM consumption per download request (< 10 MB).

---

### Phase 5: Integration, Security Hardening & Quality Assurance

**Goal**: End-to-end system verification, security controls, performance benchmarking, and documentation.

#### Key Deliverables
- **Security Enhancements**:
  - Strict input sanitization and path traversal prevention for file path parameters.
  - Rate limiting and request size caps on stream requests.
- **Testing & Verification Suite**:
  - PHPUnit test suite covering unit, integration, and streaming scenarios.
  - End-to-end HTTP range verification against Java OTP-generated `graph.obj` files.
- **Documentation**:
  - API documentation, installation guide, and deployment strategies (PHP-FPM + Nginx, Swoole/Workerman).

#### Success Criteria
- 100% test pass rate across unit and integration tests.
- Byte-for-byte checksum match between streamed binary responses and original disk artifacts.

---

## 4. Alignment with Business Goals & Technical Requirements

| Business Goal / Technical Requirement | Implementation Strategy in Roadmap |
| :--- | :--- |
| **Replace Java `--load` with PHP Runtime** | Phase 1-4 deliver pure PHP graph loading, caching, and serving without JVM dependencies. |
| **Direct Binary `graph.obj` Streaming** | Phase 1 & 4 implement chunked streaming with HTTP 200/206 Range Request support. |
| **Low Resource & Memory Footprint** | Phase 3 utilizes APCu/shmop shared memory to prevent per-request parsing; Phase 4 streams via 8 KB buffers. |
| **Decoupled Workflow** | Offline Java builder outputs `graph.obj`; Phase 3 auto-detects and hot-swaps updated graph files. |
