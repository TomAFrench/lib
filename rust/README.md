# lib/rust

Earthly's official collection of rust [UDCs](https://docs.earthly.dev/docs/guides/udc).

First, import the UDC up in your Earthfile:
```earthfile
VERSION --global-cache 0.7
IMPORT github.com/earthly/lib/rust:<version/commit> AS rust
```
> :warning: Due to [this issue](https://github.com/earthly/earthly/issues/3490), make sure to enable `--global-cache` in the calling Earthfile, as shown above.

## +INIT

This UDC stores the configuration required by the other UDCs in the build environment filesystem, and installs required dependencies.

It must be called once per build environment, to avoid passing repetitive arguments to the UDCs called after it, and to install required dependencies before the source files are copied from the build context.  

### Usage

Call once per build environment:
```earthfile
DO rust+INIT ...
```

### Arguments
#### `cache_id`
Overrides default ID of the global `$CARGO_HOME` cache. Its value is exported to the build environment under the entry: `$CARGO_HOME_CACHE_ID`.

#### `keep_fingerprints (false)`
Instructs the following `+CARGO` calls to don't remove the Cargo fingerprints of the source packages. Use only when source packages have been COPYed with `--keep-ts `option.
Cargo caches compilations of packages in `target` folder based on their last modification timestamps.
By default, this UDC removes the fingerprints of the packages found in the source code, to force their recompilation and work even when the Earthly `COPY` commands used overwrote the timestamps.

#### `sweep_days (4)`
`+CARGO` calls use cargo-sweep to clean build artifacts that haven't been accessed for this number of days.

## +CARGO

This UDC runs the cargo command `cargo $args` caching the contents of `$CARGO_HOME` and `target` for future builds of the same calling target. 

Notice that in order to run this UDC, [+INIT](#init) must be called first.

### Usage

After calling `+INIT`, use it to wrap cargo commands:

```earthfile
DO rust+CARGO ...
```
### Arguments

#### `args`
Cargo subcommand and its arguments. Required.

#### `output`
Regex to match the files within the target folder to be copied from the cache to the caller filesystem (image layers). 

Use this argument when you want to `SAVE ARTIFACT` from the target folder (mounted cache), always trying to minimize the total size of the copied fileset. 

For example `--output="release/[^\./]+"` would keep all the files in `/target/release` that don't have any extension.

### Thread safety
This UDC is thread safe. Parallel builds of targets calling this UDC should be free of race conditions.

## +RUN_WITH_CACHE

`+RUN_WITH_CACHE` runs the passed command with the CARGO caches mounted.

Notice that in order to run this UDC, [+INIT](#init) must be called first.

### Arguments
#### `command (required)` 
Command to run, can be any expression.

### Example
Show `$CARGO_HOME` cached-entries size:

```earthfile
DO rust-udc+RUN_WITH_CACHE --command "du \$CARGO_HOME"
```

## Complete example

Suppose the following project:
```
.
├── Cargo.lock
├── Cargo.toml
├── deny.toml
├── Earthfile
├── package1
│   ├── Cargo.toml
│   └── src
│       └── ...
└── package2
    ├── Cargo.toml
    └── src
        └── ...
```

The Earthfile would look like:

```earthfile
VERSION --global-cache 0.7

# Importing UDC definition from default branch (in a real case, specify version or commit to guarantee immutability)
IMPORT github.com/earthly/lib/rust AS rust

install:
  FROM rust:1.73.0-bookworm
  RUN apt-get update -qq
  RUN apt-get install --no-install-recommends -qq autoconf autotools-dev libtool-bin clang cmake bsdmainutils
  RUN cargo install --locked cargo-deny
  RUN rustup component add clippy
  RUN rustup component add rustfmt
  # Call +INIT before copying the source file to avoid installing depencies every time source code changes. 
  # This parametrization will be used in future calls to UDCs of the library
  DO rust+INIT --keep_fingerprints=true

source:
  FROM +install
  COPY --keep-ts Cargo.toml Cargo.lock ./
  COPY --keep-ts deny.toml ./
  COPY --keep-ts --dir package1 package2  ./

# build builds with the Cargo release profile
build:
  FROM +source
  DO rust+CARGO --args="build --release" --output="release/[^/\.]+"
  SAVE ARTIFACT ./target/release/ target AS LOCAL artifact/target

# test executes all unit and integration tests via Cargo
test:
  FROM +source
  DO rust+CARGO --args="test"

# fmt checks whether Rust code is formatted according to style guidelines
fmt:
  FROM +source
  DO rust+CARGO --args="fmt --check"

# lint runs cargo clippy on the source code
lint:
  FROM +source
  DO rust+CARGO --args="clippy --all-features --all-targets -- -D warnings"

# check-dependencies lints our dependencies via `cargo deny`
check-dependencies:
  FROM +source
  DO rust+CARGO --args="deny --all-features check --deny warnings bans license sources"
```