+++
title = "Base64 vs Base64+Zstd: A Deep Dive into Solana's getAccountInfo Call Reponse Encoding"
date = 2025-10-09
[taxonomies]
tags =  ["solana", "rust"]
[extra]
+++

The following is a deep dive on the results of an experiment comparing the latencies of using `"encoding" = "base64` vs `"encoding" = "base64+zstd"` for Solana's [getAccountInfo](https://solana.com/docs/rpc/http/getaccountinfo) RPC call.

# Key Takeaways

- **For smaller accounts (<3KB)**: `base64`'s mean latency is faster than `base64+zstd`
- **For larger accounts (>5KB)**: `base64+zstd`'s mean latency is faster than `base64`
- A crossover point exists between ~3KB and ~5KB account sizes where `base64+zstd` begins to outperform `base64`
- Looking at the 95th percentile latency time, we can see that they yield comparable results as mean latencies, where `base64` outperforms with smaller account data, while `base64+zstd` outperforms with larger data accounts.

```
=== SUMMARY ANALYSIS (MEAN) - getAccountInfo===

Size (B)   Base64 (ms)  Zstd (ms)    Diff (ms)    P-value                        Winner
================================================================================
904        20.67        24.40        3.73         HIGHLY SIGNIFICANT (p < 0.01)  Base64
3232       25.43        30.95        5.52         HIGHLY SIGNIFICANT (p < 0.01)  Base64
5400       35.03        25.18        9.85         HIGHLY SIGNIFICANT (p < 0.01)  Zstd
7448       25.36        20.89        4.47         HIGHLY SIGNIFICANT (p < 0.01)  Zstd
10136      23.58        21.41        2.17         HIGHLY SIGNIFICANT (p < 0.01)  Zstd

```

```
=== SUMMARY ANALYSIS (P95) - getAccountInfo===

Size (B)   Base64 (ms)  Zstd (ms)    Diff (ms)    Winner
================================================================================
904        27.71        32.81        5.09         Base64
3232       34.34        36.81        2.47         Base64
5400       43.93        33.82        10.11        Zstd
7448       33.84        27.53        6.31         Zstd
10136      30.54        28.70        1.84         Zstd

```

# Methodology

- **RPC Method Tested**: [getAccountInfo](https://solana.com/docs/rpc/http/getaccountinfo) (single account call)
- **RPC Provider**: [Helius](https://www.helius.dev/pricing) paid developer plan
- **Sample Size**: 1,000 RPC requests per account size, for each encoding type
- **For each request, I measured end-to-end latency**: the time from sending the RPC request to receiving and decoding and decompressing the complete response
- I collected the mean, media, stdev, min value, max value, and P95/P95 values from each account size

## Test Accounts

| Account Size | Address |
|--------------|---------|
| 904 bytes | `Po57HwPXoG22SajaQCBn7Zg8hPp15pUTMcTwcr4GhDu` |
| 3,232 bytes | `2uLdBfEax2Gzk3KRGTiWoYQPmXWgSG6g1UDax1T9J6MM` |
| 5,400 bytes | `4yRzzxU6brZsY9CTZ2Jp9QWhnRftYP6RyZ8FGugVEXN8` |
| 7,448 bytes | `CXh2s3cJVwJxZBa7q2ur3paKsWrrznZfApmvjQCPQJpc` |
| 10,136 bytes | `Hij4zmmrdmQ49bfgy63C8VF12EYysjF6WGFeZJpfipCk` |


## Example Result Output

```
============================================================
Testing GetAccountInfo with 3232 byte account, address: 2uLdBfEax2Gzk3KRGTiWoYQPmXWgSG6g1UDax1T9J6MM
============================================================

[base64] .................................................. Completed
[base64+zstd]   .................................................. Completed

GetAccountInfo call, base64 - 3232 bytes:
  Mean:      25.43 ms
  Median:    24.00 ms
  Std Dev:   5.79 ms
  Min:       19.43 ms
  Max:       80.75 ms
  P95:       34.34 ms
  P99:       50.08 ms
  Successful calls: 1000

GetAccountInfo call, base64+zstd - 3232 bytes:
  Mean:      30.95 ms
  Median:    29.94 ms
  Std Dev:   5.02 ms
  Min:       25.10 ms
  Max:       98.74 ms
  P95:       36.81 ms
  P99:       46.61 ms
  Successful calls: 1000
```

# Understanding `getAccountInfo` Encoding Types

## What is `base64` Encoding?

When you request account data via `getAccountInfo` from a Solana RPC node, the raw binary account data needs to be transmitted over HTTP/JSON. Since JSON cannot directly represent binary data, Solana uses [base64](https://www.freecodecamp.org/news/what-is-base64-encoding/) encoding to convert binary data into ASCII text.

The process of encoding data using base64 increases the data size by ~33%, but it is a necessary tradeoff for reliable data transmission over JSON-RPC.

## What is `base64+zstd`

[Zstandard (Zstd)](https://facebook.github.io/zstd/) is a modern compression algorithm developed by Facebook that offers fast compression and decompression speeds and good compression ratios.

In the context of Solana, zstd compresses the account data before base64 encoding, reducing the amount of data transmitted over the network. The general logic steps for handling `base64+zstd` encoding in a Solana RPC call includes:

**Server-side (RPC node):**
1. Fetch raw account data from storage
2. Compress the binary data using `zstd`
3. Encode the compressed data with `base64`
4. Send JSON response with compressed+encoded data

**Client-side:**
1. Receive JSON response
2. Use `base64` to decode to get compressed binary
3. Decompress using `zstd` to get original account data

Using `base64+zstd`, you save network bandwidth latency due to a smaller data size, but at the cost of latency from additional CPU cycles for compression/decompression. 

**The question is: at what data size does the benefit of compression outweight the additional CPU overhead it costs?**

# Interpretation of Results

After conducting the experiment described in the Methodology section above, the following takeaways could be retrieved from the data:

- **For smaller accounts (<3KB)**: `base64`'s mean latency is faster than `base64+zstd`
- **For larger accounts (>5KB)**: `base64+zstd`'s mean latency is faster than `base64`

These results are in line with my expectations as they adhered to the tradeoff princible between bandwidth savings and additional costs from decompression. 

Every RPC call has three latency components:

1. **Network Latency**: latency dependent on the quality of the client and server's respective networks
2. **Data Transfer**: latency from bandwidth
3. **CPU Processing**: latency from CPU cycles needed to compress and decompress the data

At smaller data sizes, you're spending extra CPU resources for decompression but not saving enough latency from a smaller data size transfer. 

At larger data sizes, the bandwidth savings exceed the overhead from the additional decompression. A crossover point exists between ~3KB and ~5KB account sizes where `base64+zstd` begins to outperform `base64`.

**Small Data Accounts:**

```
base64 costs:     Network + Small Data transfer + CPU for decoding
base64+zstd costs:       Network + Tiny Data transfer + CPU for decoding and decompressing
```

**Larger Data Accounts:**

```
base64 costs:     Network + Larger Data transfer + CPU for decoding
base64+zstd costs:       Network + Medium Data transfer + CPU for decoding and decompressing
```

<!--
# Testing for Significance: Welch's T-Test

To determine if observed performance differences is a real signal or just random noise, I used [**Welch's t-test**](https://sites.nicholas.duke.edu/statsreview/means/welch/), a statistical method for comparing two groups with potentially different variances.

The t-statistic formula I used:

```
t = |mean₁ - mean₂| / √((s₁²/n₁) + (s₂²/n₂))
```

Where:
- `mean₁, mean₂`: Average latencies for `base64` and `base64+zstd`
- `s₁, s₂`: Standard deviations
- `n₁, n₂`: Sample sizes

## Interpreting the T-Statistic

For an n=1000 sample size, the t-distribution likely approximates to a normal distribution. The critical values are The t-statistic tells us how many standard errors apart the two means are:

- **t > 2.58**: Highly significant (p < 0.01) - Less than 1% chance this is random
- **t > 1.96**: Significant (p < 0.05) - Less than 5% chance this is random
- **t < 1.96**: Not significant - Could be random variation

All our tests showed **t > 2.58**, meaning we can be 99% confident the performance differences are real.
-->
