+++
title = "Jito gRPC Client"
date = 2025-10-03
[taxonomies]
tags =  ["project", "solana", "jito", "rust"]
[extra]
+++

# Intro

As a frequent user of [Jito](https://docs.jito.wtf/), anywhere I look online, there were only resources showing how to connect to Jito's block engine endpoints via `JSON-RPC`, eventhough `gRPC` connections are also supported. Below is my implementation of a `Rust` client for connecting to Jito's blockengine nodes via `gRPC`. 

Currently, this library only supports non-auth connections, though an auth key connection can be implemented in the future if there's enough interest.

[Crates.io](https://crates.io/crates/jito-grpc-client) | [Github](https://github.com/stevenlusonggao/jito_grpc_client)

# Features

- **Bundle Transactions**: Send jito bundles via gRPC, no auth key needed
- **Dynamic Region Selection**: Option to automatically connect to the fastest available region based on latency measurements
- **Retry Logic**: Automatic retry with configurable jitter

---

# Getting Started

Basic usage example:

```rust
#[tokio::main]
async fn main() -> JitoClientResult<()> {
    // Connect to fastest region automatically
    let mut client = JitoClient::new_dynamic_region(None).await?;
    
    let transactions: Vec<VersionedTransaction> = vec![
        // Your transactions
    ];
    
    // Send bundle
    let uuid = client.send(&transactions).await?;
    println!("Bundle submitted with UUID: {}", uuid);
    
    Ok(())
}
```

---

# Client Creation

## `JitoClient::new_dynamic_region(timeout: Option<u64>) -> JitoClientResult<Self>`

Creates a new client instance that automatically connects to the fastest available region. This method measures latency to all available regions and selects the one with the lowest response time for optimal performance.

**Parameters:**

- `timeout` - Connection and request timeout in seconds. Defaults to 2 seconds if not specified.

**Example:**

```rust
// Use default 2-second timeout
let client = JitoClient::new_dynamic_region(None).await?;

// Use custom 5-second timeout
let client = JitoClient::new_dynamic_region(Some(5)).await?;
```

## `JitoClient::new(endpoint: &'static str, timeout: Option<u64>) -> JitoClientResult<Self>`

Creates a new client instance connected to a specific endpoint.

**Parameters:**

- `endpoint` - The gRPC endpoint URL
- `timeout` - Connection and request timeout in seconds. Defaults to 2 seconds if None.

**Example:**

```rust
// Connect to specific endpoint
let client = JitoClient::new("https://ny.mainnet.block-engine.jito.wtf:443", None).await?;

// Connect with custom timeout
let client = JitoClient::new("http://ny.testnet.block-engine.jito.wtf:443", Some(10)).await?;
```

---

# Transaction Submission

## `send(transactions: &[VersionedTransaction]) -> JitoClientResult<String>`

Sends a bundle of transactions to a Node via gRPC.

**Parameters:**

- `transactions` - A vec of [VersionedTransaction](https://docs.rs/solana-transaction/latest/solana_transaction/versioned/struct.VersionedTransaction.html) to send.

**Returns:**

- Returns a String containing the unique bundle ID.

**Example:**

```rust
use solana_transaction::versioned::VersionedTransaction;

let transactions = vec![/* your VersionedTransaction */];

match client.send(&transactions).await {
    Ok(uuid) => println!("Bundle submitted with UUID: {}", uuid),
    Err(e) => eprintln!("Failed to send bundle: {}", e),
}
```

## `send_with_retry(transactions: &[VersionedTransaction], retry_logic: RetryLogic) -> JitoClientResult<String>` 

Sends a bundle of transactions with automatic retry logic by using random jitter between attempts.

**Parameters:**

- `transactions` - A vec of [VersionedTransaction](https://docs.rs/solana-transaction/latest/solana_transaction/versioned/struct.VersionedTransaction.html) to send.
- `retry_logic` - Configuration for retry behavior including max attempts and wait times, see below for info on the retry config.

**Example:**

```rust
use solana_transaction::versioned::VersionedTransaction;

// 3 retries with default 5-25ms jitter
let retry_config = RetryLogic::new(3);
let transactions = vec![/* your VersionedTransaction */];

match client.send_with_retry(&transactions, retry_config).await {
    Ok(uuid) => println!("Bundle submitted with UUID: {}", uuid),
    Err(e) => eprintln!("Bundle failed: {}", e),
}
```

---

# Retry Configuration

## `RetryLogic::new(max_retries: u8) -> Self`

Creates retry configuration with default wait bounds (5-25ms).

**Parameters:**

- `max_retries` - Maximum number of retry attempts

**Example:**

```rust
// With default jitter (5-25ms)
let retry_config = RetryLogic::new(5);
```

## `RetryLogic::new_with_wait_bounds(max_retries: u8, min_wait: u64, max_wait: u64) -> JitoClientResult<Self>`

Creates retry configuration with custom wait time bounds.

**Parameters:**

- `max_retries` - Maximum number of retry attempts
- `min_wait` - Minimum wait time in milliseconds
- `max_wait` - Maximum wait time in milliseconds

**Example:**

```rust
// 5 retries with custom jitter (10-100ms)
let retry_config = RetryLogic::new_with_wait_bounds(5, 10, 100)?;
```

---

# Utility Methods

## `JitoClient::get_endpoint() -> &'static str`

Returns the endpoint URL that this client is currently connected to.

**Example:**

```rust
let client = JitoClient::new_dynamic_region(None).await?;
println!("Connected to: {}", client.get_endpoint());
```

## `JitoClient::all_regions() -> &'static [NodeRegion]`

Returns all available node regions that can be used for connections.

**Example:**

```rust
// List all available regions
for region in JitoClient::all_regions() {
    println!("Region: {} - Endpoint: {}", region, region.endpoint());
}
```

---

# Error Handling

The client uses `JitoClientResult<T>` which wraps `JitoClientError`. Common error types include:

- Network/gRPC connection errors
- Max retries reached
- gRPC server response errors
