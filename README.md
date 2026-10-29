# ferrous-query-engine

> **Apache DataFusion Vectorized Analytical Query Engine & Catalog Federation for FerrousCX**  
> Pure-Rust analytical query execution engine powered by [Apache DataFusion](https://datafusion.apache.org/) and `delta-lake-rs`, providing sub-3ms in-memory vectorized SQL scans, extensible `TableProvider` federation, and Arrow Flight SQL query serving.

[![Target: Apache DataFusion](https://img.shields.io/badge/engine-Apache%20DataFusion%2040+-blue.svg)](https://datafusion.apache.org/)
[![Table Format: Delta Lake](https://img.shields.io/badge/storage-Delta%20Lake%20(delta--rs)-orange.svg)](https://delta.io/)
[![Protocol: Arrow Flight SQL](https://img.shields.io/badge/protocol-Arrow%20Flight%20SQL-yellow.svg)](https://arrow.apache.org/)
[![DevSecOps: Tier 1 Rust](https://img.shields.io/badge/DevSecOps-Tier%201%20Rust%20CI-brightgreen.svg)](#devsecops--ci-standards)

---

## 🎯 Architectural Role

`ferrous-query-engine` serves as the dedicated analytical query tier of the FerrousCX platform:

1. **Sub-3ms Vectorized SQL Scans:** Compiles and executes complex analytical SQL queries against 100k+ record Parquet datasets using Apache DataFusion's SIMD-vectorized execution plans.
2. **Delta Lake `TableProvider` Catalog:** Registers Delta Lake storage partitions (`orders_raw_events`, `orders_current_state`, `llm_benchmarks`) from `ferrous-lakehouse` via `delta-lake-rs`, reading ACID versioned transaction logs (`_delta_log/`) with zero IPC overhead.
3. **Extensible Query Federation:** Connects Delta Lake cold storage, in-memory tables, and external operational datastores under a unified SQL catalog interface.
4. **Standalone Analytical Serving:** Exposes high-performance Arrow Flight SQL (port 50051) and HTTP query endpoints (port 8085) for heavy data exploration without degrading the operational Axum API gateway (`ferrous-api`).
5. **Decoupled from Stream Ingestion:** Push-based stream processing and 2PC micro-batch commits are handled upstream in `ferrous-streaming` via Arroyo, preserving clean separation of concerns between write-heavy streaming and read-heavy OLAP analytics.

```text
┌────────────────────────────────────────────────────────┐
│               ANALYTICAL QUERY CONSUMERS               │
│   (Svelte Studio Screen 6, In-Process API, CLI Tools)  │
└───────────────────────────┬────────────────────────────┘
                            │ Arrow Flight SQL (50051) / HTTP (8085)
                            ▼
┌────────────────────────────────────────────────────────┐
│                ferrous-query-engine                    │
│                                                        │
│  ┌───────────────────────┐  ┌───────────────────────┐  │
│  │ SessionContext Engine │  │ Vectorized SIMD Plans │  │
│  └───────────┬───────────┘  └───────────┬───────────┘  │
│              │                          │              │
│              ▼                          ▼              │
│  ┌──────────────────────────────────────────────────┐  │
│  │     Delta Lake TableProvider Catalog             │  │
│  └───────────────────────┬──────────────────────────┘  │
└──────────────────────────┼─────────────────────────────┘
                           │ Zero-Copy In-Memory Vectorized Scans (<3ms)
                           ▼
┌────────────────────────────────────────────────────────┐
│        ferrous-lakehouse (Delta Lake Storage Tier)     │
│  • orders_raw_events   • orders_current_state          │
│  • llm_benchmarks      • olist_orders / reviews        │
└────────────────────────────────────────────────────────┘
```

---

## 🔌 Port Mappings & Network Topology

| Service | Port | Protocol | Purpose |
| :--- | :---: | :---: | :--- |
| **Arrow Flight SQL** | `50051` | gRPC | High-throughput columnar record batch streaming |
| **HTTP Query API** | `8085` | HTTP / JSON | RESTful ad-hoc analytical queries & healthchecks |

---

## 🔒 Security & Service Authentication

- Governed under Kanidm OAuth2 M2M service accounts.
- Standalone execution runs under a non-root unprivileged container user (`UID 10001`).
- Mounts Delta Lake storage tier (`/data/delta`) in strict read-only mode (`ro`).

---

## 🛠️ Developer Quickstart

```bash
# Build standalone binary
cargo build --release

# Run DataFusion standalone query server
cargo run --bin query_server -- --delta-path /data/delta --port 8085

# Execute interactive SQL query
cargo run --bin query_cli -- --sql "SELECT count(*) FROM orders_current_state WHERE status = 'delivered'"
```

---

## 🛡️ DevSecOps & CI Standards

Adheres strictly to **Tier 1 Pure-Rust Standards**:
- Pre-commit: `cargo fmt --check`, `cargo clippy -- -D warnings`, `gitleaks`.
- Security Gates: `cargo audit`, `cargo deny` (zero HIGH/CRITICAL CVEs).
- Image Hardening: Multi-stage Dockerfile (`cargo-chef` builder + `distroless` runtime, non-root UID 10001, `hadolint` clean).
