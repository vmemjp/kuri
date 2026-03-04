# Agentic Browdie 🧁

> A high-performance browser automation & web crawling toolkit for AI agents — written in **Zig**.
>
> Inspired by [Pinchtab](https://github.com/pinchtab/pinchtab) (browser control via HTTP API) and [Pathik](https://github.com/justrach/pathik) (high-performance web crawler), rebuilt from scratch in Zig for maximum performance, minimal memory footprint, and zero-dependency deployment.

---

## Why Zig?

Both Pinchtab (Go) and Pathik (Go core + Python/JS bindings) are excellent tools. Zig lets us push further:

| Dimension | Go (Pinchtab / Pathik) | Zig (Agentic Browdie) |
|---|---|---|
| **Memory** | GC pauses, ~50-100 MB baseline | No GC, arena allocators, ~5-15 MB baseline |
| **Binary size** | ~15-30 MB | ~2-5 MB (static, no libc) |
| **Concurrency** | Goroutines (M:N scheduler) | `io_uring` / kqueue + async frames, zero-alloc task pool |
| **Startup** | ~50-100ms | ~1-5ms |
| **C interop** | CGo overhead | Native C ABI, zero-cost FFI |
| **Cross-compile** | `GOOS/GOARCH` | Single binary, any target from any host |

---

## Architecture Overview

Agentic Browdie combines the two reference projects into a unified system:

```
┌─────────────────────────────────────────────────────────────────┐
│                        HTTP API Layer                           │
│  (Zig std.http.Server — async, zero-alloc request parsing)      │
├───────────────┬─────────────────────────┬───────────────────────┤
│  Browser      │   Crawler Engine        │   Storage / Streaming │
│  Bridge       │   (Pathik-style)        │                       │
│  (Pinchtab)   │                         │                       │
├───────────────┼─────────────────────────┼───────────────────────┤
│ CDP Client    │ HTTP/TLS Fetcher        │ Kafka Producer        │
│ Tab Registry  │ Readability Extractor   │ R2/S3 Uploader        │
│ A11y Snapshot │ HTML→Markdown Converter │ Local File Writer     │
│ Stealth       │ Parallel URL Pipeline   │ Compression (zstd,lz4)│
│ Ref Cache     │ Rate Limiter            │                       │
└───────────────┴─────────────────────────┴───────────────────────┘
                            │
                    ┌───────┴───────┐
                    │  Zig Runtime  │
                    │  io_uring /   │
                    │  kqueue async │
                    │  Arena allocs │
                    └───────────────┘
```

---

## Features (Mapped from Pinchtab + Pathik)

### 🌐 Browser Control (from Pinchtab)

Pinchtab provides an HTTP API for AI agents to control Chrome via the Chrome DevTools Protocol. We replicate this:

- **Chrome DevTools Protocol (CDP) client** — pure Zig WebSocket client speaking CDP JSON-RPC
- **Bridge & Tab Registry** — thread-safe tab lifecycle management with `std.Thread.RwLock`
- **Accessibility Tree Snapshots** — structured page representation (far more token-efficient than screenshots for LLMs)
- **Token Optimization** — `filter=interactive` (75% reduction), `depth=N` (40%), `format=text` (40-60%), `diff=true` (delta-only)
- **Stealth Mode** — JS injection to bypass bot detection (`navigator.webdriver`, user-agent spoofing)
- **Session Persistence** — save/restore tab state across restarts
- **Element Actions** — click, type, fill, scroll, hover via cached ref IDs

### 🕷️ Web Crawler (from Pathik)

Pathik is a Go crawler with Python/JS bindings that converts pages to HTML + Markdown. We replicate this:

- **Headless browser fetching** — via CDP (replaces go-rod)
- **Readability extraction** — clean article content from noisy HTML
- **HTML → Markdown conversion** — structured output for LLM consumption
- **Parallel URL pipeline** — bounded concurrency with async task pool
- **Rate limiting** — token bucket rate limiter to avoid DOS
- **URL & IP validation** — block private IPs, enforce HTTPS
- **Kafka streaming** — stream crawled content with compression (gzip, zstd, lz4, snappy)
- **R2/S3 upload** — cloud storage integration
- **Proxy rotation** — WebSocket proxy support with random selection

### 🆕 Zig-Native Additions

- **io_uring / kqueue async I/O** — true async without green threads
- **Arena allocators** — per-request memory arenas, freed in bulk (zero fragmentation)
- **SIMD HTML parsing** — vectorized tag scanning for 2-4x throughput
- **Memory-mapped file I/O** — zero-copy file writes for large pages
- **Static binary** — single ~3MB binary, no runtime dependencies

---

## Project Structure

```
agentic-browdie/
├── build.zig                  # Build system
├── build.zig.zon              # Package manifest
├── src/
│   ├── main.zig               # Entry point, CLI parsing, HTTP server startup
│   │
│   ├── server/
│   │   ├── router.zig         # HTTP route registration & dispatch
│   │   ├── middleware.zig     # Auth, CORS, logging middleware
│   │   └── response.zig      # JSON response helpers
│   │
│   ├── bridge/
│   │   ├── bridge.zig         # Central Bridge struct (tab registry + snapshot cache)
│   │   ├── tab.zig            # TabEntry, TabContext resolution
│   │   └── config.zig         # Environment config, timeouts, defaults
│   │
│   ├── cdp/
│   │   ├── client.zig         # WebSocket CDP client (connect, send, recv)
│   │   ├── protocol.zig       # CDP domain types (Page, DOM, Accessibility, Runtime)
│   │   ├── actions.zig        # High-level actions (click, type, navigate, evaluate)
│   │   └── stealth.zig        # Stealth JS injection
│   │
│   ├── snapshot/
│   │   ├── a11y.zig           # Accessibility tree node types
│   │   ├── builder.zig        # buildSnapshot — filter, depth, flatten
│   │   ├── diff.zig           # Delta diffing between snapshots
│   │   ├── formatter.zig      # JSON / text / compact output formats
│   │   └── ref_cache.zig      # ref→backendNodeId mapping cache
│   │
│   ├── crawler/
│   │   ├── fetcher.zig        # Page fetching with retries, rate limiting
│   │   ├── extractor.zig      # Readability content extraction
│   │   ├── markdown.zig       # HTML → Markdown converter
│   │   ├── pipeline.zig       # Parallel URL processing pipeline
│   │   └── validator.zig      # URL validation, private IP blocking
│   │
│   ├── storage/
│   │   ├── local.zig          # Local file writer
│   │   ├── kafka.zig          # Kafka producer with compression
│   │   └── r2.zig             # Cloudflare R2 / S3 uploader
│   │
│   └── util/
│       ├── allocator.zig      # Arena allocator helpers
│       ├── json.zig           # Fast JSON serialization (std.json)
│       └── pool.zig           # Async task pool (bounded concurrency)
│
├── js/
│   ├── stealth.js             # Stealth injection script (embedded at comptime)
│   └── readability.js         # Readability extraction script (embedded at comptime)
│
└── test/
    ├── cdp_test.zig
    ├── snapshot_test.zig
    ├── crawler_test.zig
    └── integration_test.zig
```

---

## HTTP API (Pinchtab-Compatible)

All endpoints return JSON. Authentication via `PINCHTAB_SECRET` header.

### Health & Status

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | Chrome connection status + open tab count |
| `GET` | `/tabs` | List all open tabs (id, url, title) |

### Page Inspection

| Method | Path | Params | Description |
|--------|------|--------|-------------|
| `GET` | `/snapshot` | `tabId`, `filter=interactive`, `depth=N`, `format=text\|json`, `diff=true` | Accessibility tree snapshot |
| `GET` | `/screenshot` | `tabId`, `quality=80`, `encoding=base64\|raw` | JPEG screenshot |
| `GET` | `/text` | `tabId`, `readability=true` | Extracted page text |

### Browser Control

| Method | Path | Body | Description |
|--------|------|------|-------------|
| `POST` | `/navigate` | `{"url": "...", "tabId": "...", "newTab": true}` | Navigate or open new tab |
| `POST` | `/action` | `{"kind": "click\|type\|fill\|scroll", "ref": "e0", ...}` | Interact with elements |
| `POST` | `/evaluate` | `{"expression": "...", "tabId": "..."}` | Execute JavaScript |
| `POST` | `/tab` | `{"action": "new\|close", "tabId": "..."}` | Tab management |

### Crawler Endpoints (Pathik-style)

| Method | Path | Body | Description |
|--------|------|------|-------------|
| `POST` | `/crawl` | `{"urls": [...], "parallel": true, "outputDir": "..."}` | Crawl URLs → HTML + Markdown |
| `POST` | `/crawl/r2` | `{"urls": [...], "uuid": "..."}` | Crawl + upload to R2 |
| `POST` | `/crawl/kafka` | `{"urls": [...], "topic": "...", "compression": "zstd"}` | Crawl + stream to Kafka |

---

## Key Implementation Details

### 1. CDP Client (Pure Zig WebSocket)

Pinchtab uses Go's `chromedp` library. We implement a raw CDP client:

```zig
const CdpClient = struct {
    ws: std.http.WebSocket,
    next_id: std.atomic.Value(u32),
    pending: std.AutoHashMap(u32, *ResponsePromise),
    allocator: std.mem.Allocator,

    pub fn send(self: *@This(), method: []const u8, params: anytype) !CdpResponse {
        const id = self.next_id.fetchAdd(1, .monotonic);
        const msg = try std.json.stringifyAlloc(self.allocator, .{
            .id = id,
            .method = method,
            .params = params,
        });
        defer self.allocator.free(msg);
        try self.ws.writeText(msg);
        return self.waitForResponse(id);
    }

    pub fn getAccessibilityTree(self: *@This()) ![]A11yNode {
        const resp = try self.send("Accessibility.getFullAXTree", .{});
        return parseA11yNodes(resp.result.nodes, self.allocator);
    }
};
```

### 2. Arena-Per-Request Memory Model

Every HTTP request gets its own arena allocator — all memory freed in one operation at request end:

```zig
fn handleRequest(raw_req: std.http.Server.Request) void {
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit(); // One bulk free — zero fragmentation

    const allocator = arena.allocator();
    // All parsing, snapshot building, JSON serialization uses this allocator
    // No individual frees needed, no leaks possible
}
```

### 3. Snapshot Builder with Token Optimization

Port of Pinchtab's `buildSnapshot()` with filter/depth/diff:

```zig
pub fn buildSnapshot(
    nodes: []const RawA11yNode,
    opts: SnapshotOpts,
    allocator: std.mem.Allocator,
) ![]A11yNode {
    var result = std.ArrayList(A11yNode).init(allocator);

    for (nodes) |node| {
        if (opts.max_depth) |max| if (node.depth > max) continue;
        if (opts.filter_interactive and !isInteractive(node.role)) continue;

        try result.append(.{
            .ref = try std.fmt.allocPrint(allocator, "e{d}", .{result.items.len}),
            .role = node.role,
            .name = node.name,
            .value = node.value,
            .backend_node_id = node.backend_dom_node_id,
        });
    }

    return result.toOwnedSlice();
}
```

### 4. Parallel Crawler Pipeline

Port of Pathik's concurrent URL processing with bounded concurrency:

```zig
pub fn crawlUrls(urls: []const []const u8, max_concurrent: usize, allocator: std.mem.Allocator) ![]CrawlResult {
    var pool = try AsyncPool.init(allocator, max_concurrent);
    defer pool.deinit();

    var results = try allocator.alloc(CrawlResult, urls.len);

    for (urls, 0..) |url, i| {
        try pool.spawn(crawlOne, .{ url, &results[i] });
    }

    pool.waitAll();
    return results;
}

fn crawlOne(url: []const u8, result: *CrawlResult) void {
    const html = fetchPage(url) catch |err| {
        result.* = .{ .err = err };
        return;
    };
    const content = extractReadability(html) catch |err| {
        result.* = .{ .err = err };
        return;
    };
    const markdown = htmlToMarkdown(content) catch |err| {
        result.* = .{ .err = err };
        return;
    };
    result.* = .{ .html = html, .markdown = markdown };
}
```

---

## Optimization Strategies

### vs. Pinchtab (Go)

| Area | Pinchtab (Go) | Agentic Browdie (Zig) | Expected Gain |
|------|---------------|----------------------|---------------|
| **JSON parsing** | `encoding/json` (reflection) | `std.json` (comptime-generated) | 3-5x faster |
| **Snapshot cache** | `map[string]*refCache` (GC-traced) | `AutoHashMap` + arena | No GC pauses |
| **WebSocket framing** | `gobwas/ws` | Direct `std.http.WebSocket` | Less allocation |
| **Concurrent tabs** | `sync.RWMutex` + goroutines | `std.Thread.RwLock` + thread pool | Lower overhead |
| **Snapshot diffing** | Allocated diff nodes | In-place diff on arena | Zero extra allocs |
| **Binary embed** | `//go:embed` | `@embedFile` (comptime) | Same (both zero-cost) |

### vs. Pathik (Go + Rod)

| Area | Pathik (Go) | Agentic Browdie (Zig) | Expected Gain |
|------|-------------|----------------------|---------------|
| **Browser automation** | go-rod (high-level) | Direct CDP over WebSocket | Less overhead |
| **HTML parsing** | goquery + readability | SIMD-assisted tag scanner | 2-4x throughput |
| **Markdown conversion** | html-to-markdown lib | Comptime-optimized walker | Less allocation |
| **Rate limiting** | `golang.org/x/time/rate` | Token bucket (lock-free atomic) | Zero contention |
| **File I/O** | `ioutil.WriteFile` | Memory-mapped writes | Zero-copy for large files |
| **Kafka producer** | segmentio/kafka-go | Raw Kafka protocol (Zig) | Fewer allocations |
| **Memory per crawl** | ~50MB (browser instance) | ~5MB (shared browser, arena) | 10x reduction |

### General Zig Optimizations

1. **Comptime JSON schema generation** — serialize/deserialize without runtime reflection
2. **Arena allocators** — per-request arenas eliminate fragmentation and GC
3. **SIMD string operations** — `@Vector` for fast HTML tag scanning and whitespace stripping
4. **`@prefetch`** — prefetch next accessibility nodes during tree walk
5. **Packed structs** — minimize cache line usage for `A11yNode` and `TabEntry`
6. **io_uring batching** — batch multiple CDP commands into single syscall submission
7. **Zero-copy response** — write JSON directly to socket buffer, no intermediate string

---

## Quick Start

### Prerequisites

- Zig ≥ 0.15.1
- Chrome / Chromium (for browser control features)

### Build & Run

```bash
# Build optimized release binary
zig build -Doptimize=.ReleaseFast

# Run the server (browser control + crawler API)
./zig-out/bin/agentic-browdie --port 9222

# Or connect to existing Chrome
CDP_URL=ws://localhost:9222 ./zig-out/bin/agentic-browdie --port 8080
```

### Usage Examples

```bash
# Health check
curl http://localhost:8080/health

# Take an accessibility snapshot (token-optimized for LLMs)
curl "http://localhost:8080/snapshot?filter=interactive&format=text"

# Navigate to a page
curl -X POST http://localhost:8080/navigate \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com"}'

# Click an element by ref
curl -X POST http://localhost:8080/action \
  -H "Content-Type: application/json" \
  -d '{"kind": "click", "ref": "e3"}'

# Crawl URLs to HTML + Markdown
curl -X POST http://localhost:8080/crawl \
  -H "Content-Type: application/json" \
  -d '{"urls": ["https://example.com"], "parallel": true}'

# Extract readable text (cheapest for LLMs — ~800 tokens vs ~10k for snapshots)
curl "http://localhost:8080/text?readability=true"
```

---

## Token Cost Comparison (from Pinchtab benchmarks)

For a 50-page monitoring task:

| Method | Tokens | Cost ($) | Latency |
|--------|--------|----------|---------|
| `/text` (readability) | ~40,000 | $0.20 | 50-200ms |
| `/snapshot?filter=interactive` | ~180,000 | $0.90 | 100-300ms |
| `/snapshot` (full) | ~525,000 | $2.63 | 150-400ms |
| `/screenshot` (vision) | ~100,000 | $1.00 | 300-800ms |

**Recommendation:** Use `/text` for read-heavy tasks (13x cheaper than screenshots). Use `/snapshot?filter=interactive` when you need to interact with elements.

---

## Roadmap

- [x] Research & architecture design (this document)
- [ ] CDP WebSocket client (`src/cdp/client.zig`)
- [ ] Bridge & tab registry (`src/bridge/`)
- [ ] Accessibility tree snapshot pipeline (`src/snapshot/`)
- [ ] HTTP server with Pinchtab-compatible API (`src/server/`)
- [ ] Stealth mode injection (`src/cdp/stealth.zig`)
- [ ] Crawler fetcher + readability extractor (`src/crawler/`)
- [ ] HTML → Markdown converter (`src/crawler/markdown.zig`)
- [ ] Parallel crawl pipeline with async pool (`src/crawler/pipeline.zig`)
- [ ] Kafka streaming with compression (`src/storage/kafka.zig`)
- [ ] R2/S3 upload integration (`src/storage/r2.zig`)
- [ ] Session persistence (`src/bridge/`)
- [ ] SIMD HTML parser optimization
- [ ] Python & JS FFI bindings (like Pathik)
- [ ] Benchmarks vs. Pinchtab & Pathik

---

## References

- **[Pinchtab](https://github.com/pinchtab/pinchtab)** — Go, browser control for AI agents via HTTP + CDP. Flat package, ~1,100 LOC. Key insight: accessibility trees are 13x cheaper than screenshots for LLMs.
- **[Pathik](https://github.com/justrach/pathik)** — Go core crawler with Python/JS bindings. Uses go-rod for browser automation, go-readability for content extraction, html-to-markdown for conversion. 10x less memory than Playwright.
- **[Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/)** — The protocol both tools use to control Chrome.

---

## License

Apache-2.0 — same as Pathik.
