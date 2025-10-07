+++
title = "Solana Vanity Address Generator CLI"
date = 2025-10-06
[taxonomies]
tags =  ["project", "solana", "rust"]
[extra]
+++

# Solana Vanity Address Generator CLI

A high-performance, multi-threaded Solana vanity address generator CLI written in Rust. Generate custom Solana wallet addresses with specific prefixes or suffixes using flexible matching options.

A vanity public key is a Solana address that begins or ends with specific characters you choose. The more characters you want at the beginning of your vanity address, the longer it will take to generate one.

[Github](https://github.com/stevenlusonggao/solana_vanity_address)

## Features

- ⚡ Optimized Multi-threaded Performance - Built with [Rayon](https://docs.rs/rayon/latest/rayon/) for efficient parallel processing. Utilize multiple CPU cores for maximum performance.
- 🎯 Flexible Matching - Match patterns with lookalike characters (e.g., s matches S, 5).
- 🔤 Case Sensitivity - Choose between case-sensitive, case-insensitive. 
- 🔍 Multiple Match Types - Search for prefix, suffix, or either.
- ✅ Base58 Validation - Automatically validates patterns against Solana's [Base58](https://digitalbazaar.github.io/base58-spec/) character set.

## Installation

### Prerequisites:

- [Install Rust](https://rust-lang.org/tools/install/)
- Cargo (comes with Rust)

### Build from Source

```bash
# Clone the repository
git clone https://github.com/stevenlusonggao/solana_vanity_address
cd solana_vanity_address

# Build in release mode (optimized)
cargo build --release
```

## Usage

### Basic Usage

```bash
# Generate address with "Punk" prefix using 8 threads
cargo run --release -- -f "Punk" -t 8
```

### Command-Line Options

```
Options:
  -f, --find <FIND>
        Pattern to find 

  -t, --threads <THREADS>
        Number of threads to use. [default: 2]     

  -m, --match-type <MATCH_TYPE>
        Where the pattern search should take place. [default: prefix] [possible values: prefix, suffix, either]

  -s, --case-sensitivity 
        Enable case sensitivity. [default: false]

  -l, --flexible-chars
        Enable flexible char find. [default: true]        

  -h, --help                     
        Print help
```

### Examples

#### Generate a Prefix Vanity Address

```bash
# Find address starting with "Sol"
cargo run --release -- -f "Sol" -t 6
```

#### Generate a Suffix Vanity Address

```bash
# Find address ending with "Punk"
cargo run --release -- -f  "Punk" -m suffix -t 8
```

#### Match Either Prefix or Suffix

```bash
# Find "Cool" at start OR end
cargo run --release -- -f "Cool" -m either -t 8
```

#### Case-Sensitive Matching

```bash
# Only match exact case "DEGEN"
cargo run --release -- -f "DEGEN" --case-sensitivity true --flexible-chars false -t 8
```

#### Disable Flexible Character Matching

```bash
# Match only exact characters
 cargo run --release -- -f "Test" --flexible-chars false --case-sensitivity false -t 8

# Will match: Test, test, TEST
# Won't match: 7est (7 looks like T in flexible mode)
```

## Performance Tips

2. **Optimize Thread Count** - Use a higher number of threads to improve performance, though the program prevents making more threads than available CPU threads.
3. **Shorter Patterns** - Each additional character increases search time exponentially
4. **Use Flexible Mode** - Increases match probability
5. **Choose Prefix Over Suffix** - Slightly faster than suffix

## Base58 Character Set

Solana addresses use Base58 encoding, which excludes visually ambiguous characters:

**Valid characters:** `123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz`

**Excluded characters:** `0` (zero), `O` (capital o), `I` (capital i), `l` (lowercase L)

## Security Notes

⚠️ **Important Security Considerations:**

- **Never share your private key** - Keep it secure
- Consider using a hardware wallet for storing significant funds
- The private key is displayed only once - save it immediately

## Acknowledgments

- Built with [Solana SDK](https://docs.solana.com/developing/clients/rust-api)
- Powered by [Rayon](https://github.com/rayon-rs/rayon) for parallel processing
- CLI parsing with [Clap](https://github.com/clap-rs/clap)

## Support

For issues, questions, or suggestions, please open an issue on [Github](https://github.com/stevenlusonggao/solana_vanity_address). Contributions are welcome! Please feel free to submit a Pull Request.

