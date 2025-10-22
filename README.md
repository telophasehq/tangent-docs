# Tangent

Discord 

## What is Tangent?

Tangent is an log data pipeline focused on providing the best developer experience. We allow users to transform logs in any language that can compile to WebAssembly. We've greatly simplified the toolchain for shipping WASM programs, and focus on making WASM performant so our users can focus on the transformation business logic.

## How is Tangent different from other log pipelines?

Tangent is the fastest way to ship logs with **heavy transformations** to a **data warehouse**.

1. **WebAssembly-based transformations** – Most log pipelines require a DSL for transformations. That quickly becomes cumbersome when you want to transform to a schema like [OCSF](https://ocsf.io/). Transformations like OCSF are much easier with the expressiveness, testability, and shareability of a full language. Finally, we want writing transformations to be as LLM-friendly as possible, and using an known programming language like Python or Go is the best way to accomplish that.
2. **First-class support for data warehouses/lakes** – We expect many of our users to land logs in a data warehouse. We will have native support Iceberg, Delta Lake, et al. as sinks.

## Quickstart

Prerequisites:

- Rust toolchain (1.72+ recommended)
- For Go components: TinyGo and Go toolchain
- For Python components (optional): componentize-py toolchain

Build binaries:

```bash
cargo build --release
# Binaries: target/release/tangent (CLI), target/release/tangentd (runtime)
```

Scaffold a new processor project:

```bash
target/release/tangent scaffold --name my-processor --lang go   # or py
```

Compile a processor to WASM (component):

```bash
target/release/tangent compile-wasm \
  --config ./tangent.yaml \
  --wit ./assets/wit \
  --out ./compiled
```

Run the runtime with a config:

```bash
# Using the CLI wrapper
target/release/tangent run --config ./tangent.yaml

# Or run the runtime binary directly
target/release/tangentd --config ./tangent.yaml
```

Benchmark a config:

```bash
target/release/tangent bench --config ./crates/bench/tangent.yaml --seconds 30
```

## Configuration (tangent.yaml)

Top-level fields:

- `entry_point` (string): Source path to your processor entrypoint (e.g., `wrapper.go`, `main.py`, `.` for module root)
- `module_type` (string): `go` or `py`
- `batch_size` (number): Per-batch size in KB (default: 256)
- `batch_age` (number): Max batch age in milliseconds (default: 5)
- `workers` (number): Runtime worker threads (default: number of CPUs)
- `sources` (map): Named source blocks
- `sinks` (map): Named sink blocks

Example:

```yaml
module_type: go
entry_point: .
batch_size: 1024      # KB
batch_age: 10         # ms
workers: 4

sources:
  socket_main:
    type: socket
    socket_path: "/tmp/sidecar.sock"

sinks:
  s3_bucket:
    type: s3
    bucket_name: my-bucket
    region: us-east-1
    max_file_age_seconds: 15
    # Common sink options (see below):
    encoding: ndjson
    compression:
      type: zstd
      level: 3
    object_max_bytes: 134217728
    in_flight_limit: 8
    default: true
```

### Sources

All sources use a tagged `type` field.

- `type: socket`
  - `socket_path` (string, default: `/tmp/sidecar.sock`)
- `type: file`
  - `path` (string)
  - `decoding` (object, optional)
    - `format` (auto|ndjson|json|json-array|text|msgpack; default `auto`)
    - `compression` (auto|none|gzip|zstd; default `auto`)
- `type: sqs`
  - `queue_url` (string)
  - `wait_time_seconds` (number, default: 20)
  - `visibility_timeout` (number, default: 60)
  - `decoding` (same shape as file source)
- `type: msk` (Kafka / Amazon MSK)
  - `bootstrap_servers` (string)
  - `topic` (string)
  - `group_id` (string, default: `tangent-node`)
  - `security_protocol` (string, default: `PLAINTEXT`)
  - `auth` (object)
    - `mode: scram`
    - `sasl_mechanism` (string, default: `SCRAM-SHA-512`)
    - `username` (string)
    - `password` (string; stored securely at runtime)
  - `decoding` (same shape as file source)

### Sinks

All sinks support common options via `sinks.<name>` block:

- `encoding` (ndjson|json|avro|parquet; default: ndjson)
- `compression` (none|gziplevel|zstdlevel; default: zstd level 3)
- `object_max_bytes` (number; default: 134,217,728)
- `in_flight_limit` (number; default: 16)
- `default` (bool; if true, used when a processor output does not specify a sink)

Sink kinds:

- `type: s3`
  - `bucket_name` (string)
  - `region` (string, optional)
  - `wal_path` (string, default: `/tmp/wal`)
  - `max_file_age_seconds` (number, default: 60)
- `type: file`
  - `path` (string)
- `type: blackhole`
  - no additional fields

## Writing a Processor

Processors implement the `processor` world defined in `assets/wit/processor.wit`:

- Input: `process-logs(logs: list<u8>)`
- Output: `result<list<output>, string>` where each `output` contains `data` and a `sink` variant (s3, file, blackhole, default)

See example processors:

- `examples/basic-go` – minimal Go/TinyGo example
- `examples/ocsf-go` – OCSF mapping in Go
- `examples/zeek` – Zeek log mapping in Go with tests
- `examples/vector-sink` – Python example

Build the Zeek example:

```bash
cd examples/zeek
make build
target/release/tangent compile-wasm --config ./tangent.yaml --wit ../../assets/wit --out ./compiled
```

Run with the Zeek config:

```bash
target/release/tangent run --config ./examples/zeek/tangent.yaml
```

## Deployment

See `DEPLOYMENT.md` for an ECS deployment guide and best practices.

## Q/A

### Why WebAssembly?

- LLM benchmarks – Easier testing – Shareable

### Isn't WebAssembly slow?

WebAssembly

## License

Apache-2.0. See `LICENSE`.