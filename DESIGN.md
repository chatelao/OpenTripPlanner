# System Design Document: PHP OpenTripPlanner Graph Loader & Server

## 1. Executive Summary & Core Objective

### 1.1 Goal Statement

The objective of this project is to port the OpenTripPlanner (OTP) Java `--load` functionality to
PHP ("Portiere ausschliesslich die Java '--load'-Funktionalität auf PHP, um das extern
vorcompilierte 'graph.obj' via HTTP/HTTPS bereitzustellen").

This allows standard PHP web environments (such as PHP-FPM behind Nginx/Apache or long-running PHP
runtimes like Workerman/Swoole) to consume externally pre-compiled `graph.obj` binary files, serving
the graph asset over HTTP/HTTPS, handling read-only transit/graph queries, and managing local
caching without requiring a heavy persistent Java Virtual Machine (JVM) process.

### 1.2 In-Scope vs Out-of-Scope

#### In-Scope

- Reading and streaming serialized Java binary graph objects (`graph.obj`) over HTTP/HTTPS with
  range request support (HTTP 200/206).
- Native PHP parsing/deserialization of Java binary object streams into structured PHP graph
  representations.
- In-memory sharing and caching strategies (APCu, shared memory `shmop`, OPcache) for
  multi-megabyte/gigabyte graph objects across stateless PHP requests.
- HTTP API controller layer exposing graph metadata, graph download endpoints, and read-only query
  endpoints adhering to PSR-7 / PSR-15 standards.
- Automated cache validation and hot-swapping when updated `graph.obj` files are provided by
  external build pipelines.

#### Out-of-Scope

- Offline graph compilation/building (GTFS and OpenStreetMap ingestion), which remains strictly
  within the Java OTP build pipeline.
- Real-time GTFS-RT updates requiring graph mutation, unless limited to read-only overlay snapshots
  in memory.
- Writing back modified graph structures into binary `graph.obj` files.

---

## 2. High-Level System Architecture

```
+-------------------------------------------------------------------+
|               Java OTP Offline Builder (Producer)                  |
|  - Compiles GTFS + OSM into `graph.obj`                           |
|  - Publishes `graph.obj` to Shared Storage / S3 / HTTP Storage     |
+-------------------------------------------------------------------+
                                  |
                   HTTP/HTTPS / Shared Disk Sync
                                  v
+-------------------------------------------------------------------+
|                 PHP Graph Server Application (Consumer)           |
|                                                                   |
|  +-------------------------------------------------------------+  |
|  |                 HTTP / HTTPS Entrypoint (PSR-15)            |  |
|  |            (Nginx / Apache + PHP-FPM or Swoole/Workerman)   |  |
|  +-------------------------------------------------------------+  |
|                                 |                                 |
|  +------------------------------v------------------------------+  |
|  |                IGraphHttpServer Controller                  |  |
|  |  - GET /graph/download   - GET /graph/metadata              |  |
|  |  - GET /graph/vertices   - GET /graph/edges                 |  |
|  +------------------------------+------------------------------+  |
|                                 |                                 |
|        +------------------------+------------------------+        |
|        |                                                 |        |
|        v                                                 v        |
|  +----------------------------+             +------------------+  |
|  | IGraphCacheManager         |             | IGraphBinaryReader| |
|  | - APCu / Shared Memory     |             | - Stream Handler |  |
|  | - Cache Validation & Swap  |             | - Header Check   |  |
|  +--------------+-------------+             +--------+---------+  |
|                 |                                    |            |
|                 +------------------+-----------------+            |
|                                    |                              |
|                                    v                              |
|  +-------------------------------------------------------------+  |
|  |              IJavaObjectUnserializer                        |  |
|  |  - Java Serialization Parser                                |  |
|  |  - Construct PHP Graph Domain Objects                       |  |
|  +-------------------------------------------------------------+  |
+-------------------------------------------------------------------+
```

---

## 3. Detailed Component Architecture & Specifications

### 3.1 Binary Graph Reader Component (`IGraphBinaryReader`)

The `IGraphBinaryReader` provides abstraction over reading `graph.obj` binary streams from local
file systems, memory-mapped files, or remote HTTP/S URIs.

#### Interface Specification

```php
namespace OTP\Graph\Reader;

use Psr\Http\Message\StreamInterface;

interface IGraphBinaryReader
{
    /**
     * Opens a binary stream for reading the given graph object file or URI.
     */
    public function openGraphStream(string $filePathOrUrl): StreamInterface;

    /**
     * Validates the Java serialization magic headers (0xACED 0x0005) and class metadata.
     */
    public function validateHeader(StreamInterface $stream): GraphHeaderMetadata;

    /**
     * Reads a specific byte range from the serialized binary graph file.
     */
    public function readBlock(StreamInterface $stream, int $offset, int $length): string;
}
```

### 3.2 Java Serialization Parser Component (`IJavaObjectUnserializer`)

The `IJavaObjectUnserializer` parses Java `ObjectOutputStream` binary serialization formats into PHP
object graphs.

#### Key Mechanics

- **Magic Header Validation**: Expects stream bytes `0xACED` (STREAM_MAGIC) and `0x0005`
  (STREAM_VERSION).
- **Type Handles & Deserialization**: Resolves Java type descriptors (`TC_CLASSDESC`, `TC_OBJECT`,
  `TC_STRING`, `TC_ARRAY`, `TC_REFERENCE`) and maps standard OTP classes (e.g.,
  `org.opentripplanner.routing.graph.Graph`, `Vertex`, `Edge`, `TransitStop`) into corresponding PHP
  class representations in namespace `OTP\Domain\Model\*`.

#### Interface Specification

```php
namespace OTP\Graph\Deserializer;

use Psr\Http\Message\StreamInterface;
use OTP\Domain\Model\GraphEntity;
use OTP\Domain\DTO\MetadataDTO;

interface IJavaObjectUnserializer
{
    /**
     * Unserializes the full binary graph stream into a PHP GraphEntity.
     */
    public function unserializeGraph(StreamInterface $stream): GraphEntity;

    /**
     * Extracts lightweight header metadata without parsing the entire object graph.
     */
    public function extractGraphMetadata(StreamInterface $stream): MetadataDTO;
}
```

### 3.3 Graph Cache & Memory Management Component (`IGraphCacheManager`)

Because parsing multi-megabyte or gigabyte `graph.obj` streams on every stateless PHP HTTP request
is prohibitively slow, the `IGraphCacheManager` abstracts fast memory storage (APCu, `shmop` shared
memory, or memory-mapped OPcache file caches).

#### Interface Specification

```php
namespace OTP\Graph\Cache;

use OTP\Domain\Model\GraphEntity;

interface IGraphCacheManager
{
    /**
     * Stores the parsed graph representation in shared memory.
     */
    public function storeCachedGraph(string $graphId, GraphEntity $graph): bool;

    /**
     * Retrieves the parsed graph from shared memory if available.
     */
    public function getCachedGraph(string $graphId): ?GraphEntity;

    /**
     * Verifies if the cached version matches the file checksum/mtime on disk.
     */
    public function isCacheValid(string $graphId, string $fileHash): bool;

    /**
     * Purges invalid or outdated graph structures from memory.
     */
    public function invalidate(string $graphId): bool;
}
```

### 3.4 HTTP/HTTPS Service Controller Component (`IGraphHttpServer`)

The `IGraphHttpServer` component implements standard PSR-7 request handling and PSR-15 middleware
interfaces to expose binary file streaming and read-only graph endpoints.

An OpenAPI 3.1 specification document defining the web interface and JSON domain schemas is
available in [`openapi.yaml`](openapi.yaml).

#### Interface Specification

```php
namespace OTP\Graph\Http;

use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\ResponseInterface;

interface IGraphHttpServer
{
    /**
     * Handles requests to download or stream the raw `graph.obj` file.
     * Supports HTTP Range Requests (Status 206) for chunked binary downloading.
     */
    public function handleGraphDownloadRequest(ServerRequestInterface $request): ResponseInterface;

    /**
     * Handles graph inspection/metadata REST queries.
     */
    public function handleMetadataRequest(ServerRequestInterface $request): ResponseInterface;

    /**
     * Handles transit lookup and read-only queries against the loaded graph model.
     */
    public function handleTransitQueryRequest(ServerRequestInterface $request): ResponseInterface;
}
```

---

## 4. PHP Domain Data Model Representation

The PHP application translates serialized Java structures into lightweight, read-optimized PHP
domain models:

```
+-----------------------------------------------------------+
|                       GraphEntity                         |
| - string $graphId                                         |
| - string $buildTimestamp                                  |
| - Envelope $boundingEnvelope                              |
| - array<string, Vertex> $vertices                         |
| - array<string, Edge> $edges                              |
| - TransitIndex $transitIndex                              |
+-----------------------------------------------------------+
                               |
            +------------------+------------------+
            |                                     |
            v                                     v
+-----------------------+             +-----------------------+
|        Vertex         |             |         Edge          |
| - string $id          |             | - string $id          |
| - float $lat          |             | - Vertex $from        |
| - float $lon          |             | - Vertex $to          |
| - string $label       |             | - float $distance     |
| - array $outgoingEdges|             | - string $name        |
+-----------------------+             +-----------------------+
```

---

## 5. Sequence Flows

### 5.1 Binary Streaming Flow (`GET /graph/download`)

```
Client               HttpServerController           BinaryReader            Storage System
  |                           |                          |                        |
  |--- GET /graph/download -->|                          |                        |
  |    (Optional Range Header)|                          |                        |
  |                           |--- openGraphStream() --->|                        |
  |                           |                          |--- Read file handle -->|
  |                           |                          |<-- Return stream ------|
  |                           |<-- Return StreamInterface|                        |
  |                           |                          |                        |
  |                           |--- Validate Range Headers                        |
  |                           |--- Prepare PSR-7 Response (200 / 206)             |
|<-- 200/206 Binary Stream ---|                                                  |
```

### 5.2 Read-Only Query Flow with Cache Lookup (`GET /graph/metadata` or Query)

```
Client               HttpServerController        CacheManager        ObjectUnserializer       BinaryReader
  |                           |                       |                      |                     |
  |--- GET /graph/query ----->|                       |                      |                     |
  |                           |-- getCachedGraph() -->|                      |                     |
  |                           |                       |                      |                     |
  |                           |   [Cache Hit]         |                      |                     |
  |                           |<-- Return GraphEntity-|                      |                     |
  |                           |                       |                      |                     |
  |                           |   [Cache Miss]        |                      |                     |
  |                           |<-- Return NULL -------|                      |                     |
  |                           |                       |                      |                     |
  |                           |----------------------- openGraphStream() ------------------------->|
  |                           |<---------------------- Return Stream ------------------------------|
  |                           |                      |                      |                     |
  |                           |---------------------- unserializeGraph() -->|                     |
  |                           |<--------------------- Return GraphEntity ---|                     |
  |                           |                      |                      |                     |
  |                           |-- storeCachedGraph()->|                      |                     |
  |                           |                      |                      |                     |
  |--- Return JSON Response ->|                      |                      |                     |
```

---

## 6. Performance, Memory Management & Streaming Strategy

### 6.1 Avoiding Per-Request Deserialization Overhead

Large transit graphs can produce `graph.obj` files from 50MB up to several gigabytes. Reading and
deserializing a binary file of this size on every HTTP request in a stateless PHP worker is
unfeasible.

To guarantee high throughput and millisecond response times:

1. **Shared Memory Caching**: The unserialized PHP graph representation is persisted in APCu /
   Shared Memory (`shmop`) or serialized into OPcache-optimized PHP array files during warm-up.
2. **Chunked Memory-Mapped Streaming**: For binary HTTP downloads, responses are streamed directly
   from disk or cloud storage chunks using `Psr\Http\Message\StreamInterface` with a fixed buffer
   size (e.g. 8 KB buffers), maintaining a minimal peak RAM memory footprint per request (< 10 MB
   RAM).

### 6.2 HTTP Range Request Support

To support resumed binary graph downloads and multi-threaded stream consumers, `IGraphHttpServer`
supports `Range` and `If-Range` headers:

- `HTTP/1.1 206 Partial Content`
- `Accept-Ranges: bytes`
- `Content-Range: bytes <start>-<end>/<total>`

---

## 7. Configuration & Security Considerations

### 7.1 Configuration Model (`config.php` / Environment Variables)

```php
return [
    'graph' => [
        'file_path' => env('OTP_GRAPH_PATH', '/var/otp/graphs/graph.obj'),
        'graph_id'  => env('OTP_GRAPH_ID', 'default'),
        'auto_reload' => env('OTP_GRAPH_AUTO_RELOAD', true),
    ],
    'cache' => [
        'driver' => env('OTP_CACHE_DRIVER', 'apcu'), // 'apcu', 'shmop', 'file'
        'ttl'    => env('OTP_CACHE_TTL', 86400),
    ],
    'http' => [
        'enable_download' => env('OTP_ENABLE_DOWNLOAD', true),
        'max_chunk_bytes' => 8192,
        'cors_allowed_origins' => ['*'],
    ]
];
```

### 7.2 Security Controls

- **Path Traversal Prevention**: File paths supplied via HTTP parameters or headers are sanitized
  against directory traversal attacks (`..`).
- **Input Validation**: Graph ID parameters and range header values are strictly parsed and
  validated against regular expressions before file system access.
- **Resource Limits**: Strict time and memory limits applied to stream operations to prevent
  Denial-of-Service (DoS) vectors on binary stream operations.

---

## 8. Quality Assurance & Verification Strategy

1. **Unit Testing (PHPUnit)**:
   - Binary deserializer tests with mock Java `ObjectOutputStream` binary payloads.
   - Reader tests verifying correct byte buffer offsets and range response construction.
   - Cache manager tests validating store, fetch, invalidation, and checksum verification.

2. **Integration & Performance Verification**:
   - Verify compatibility with sample `graph.obj` files generated by the Java OTP build pipeline.
   - Run end-to-end HTTP range requests using `curl` to verify checksum equality between local
     `graph.obj` and HTTP streamed responses.
