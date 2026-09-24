# Forest Inventory Analyzer

[![CI](https://github.com/woodsy-will/forest-inventory-analyzer-rust/actions/workflows/ci.yml/badge.svg)](https://github.com/woodsy-will/forest-inventory-analyzer-rust/actions/workflows/ci.yml)

A cruise compiler written in Rust. It reads a plot tally from a variable-radius (prism) cruise, or a Survey123 or Field Maps Excel export (`Plot_form` sheets), and writes per-acre stand metrics, species composition, a text histogram of diameter classes, sampling error with confidence intervals, and a growth projection. Input and output are CSV, JSON or Excel. It runs from the command line; the web dashboard is optional. Volume equations are placeholders and the installers are unsigned. See [Methods and limitations](#methods-and-limitations).

## Features

- Cruise import: Survey123 and Field Maps Excel exports (`Plot_form` sheets) with BAF expansion for variable-radius plots
- Stand metrics: trees per acre, basal area, volume (cubic & board feet), quadratic mean diameter
- Species composition: breakdown by species with percentage of TPA and basal area
- Statistical analysis: confidence intervals, sampling error, standard error using Student's t-distribution
- Diameter distribution: text-based histogram of diameter classes
- Growth projections: exponential, logistic, and linear growth models with configurable mortality
- Multi-format I/O: read/write CSV, JSON, and Excel (.xlsx) files; export to GeoJSON
- Format conversion: convert CSV, JSON or Excel input to CSV, JSON, Excel or GeoJSON output
- Batch processing: analyze entire directories of inventory files with JSON report output
- Configuration file: optional `config.toml` for persistent settings (server, analysis, growth, database)
- Web UI: browser-based dashboard with file upload, interactive charts, data editing, and export
- SQLite persistence: server-side inventory storage with TTL-based eviction

## Installation

### Pre-built binaries

Download the latest release from [GitHub Releases](https://github.com/woodsy-will/forest-inventory-analyzer-rust/releases).

Windows (recommended). Two packages are published:
- MSI installer: run `forest-analyzer-0.2.0-x86_64-pc-windows-msvc.msi` (the version number changes with each release). It installs to `%LocalAppData%\ForestAnalyzer` with Start Menu and Desktop shortcuts and needs no admin rights.
- ZIP archive: extract it and double-click `start.bat` to launch the web dashboard.

macOS:
```bash
tar xzf forest-analyzer-*-apple-darwin.tar.gz
./forest-analyzer serve
```

Linux:
```bash
tar xzf forest-analyzer-*-x86_64-unknown-linux-gnu.tar.gz
./forest-analyzer serve
```

Supported platforms: Windows x64, Linux x64, macOS Intel (x86_64), macOS Apple Silicon (aarch64).

A SHA256 checksum (`.sha256` file) is published beside each artifact. Check the download against it.

### Build from source

Requires Rust 1.89 or later (`rust-version` in `Cargo.toml`). To install straight from GitHub:

```bash
cargo install --git https://github.com/woodsy-will/forest-inventory-analyzer-rust
```

Or clone and build:

```bash
# Clone the repository
git clone https://github.com/woodsy-will/forest-inventory-analyzer-rust.git
cd forest-inventory-analyzer-rust

# Build
cargo build --release

# The binary will be at target/release/forest-analyzer
```

## Usage

One subcommand per task. The flags shown are the ones you will change most often.

### Analyze inventory data

```bash
# Full analysis with default settings
forest-analyzer analyze --input data/samples/sample_inventory.csv

# Custom confidence level and diameter class width
forest-analyzer analyze --input inventory.csv --confidence 0.90 --diameter-class-width 4.0
```

### Growth projections

```bash
# Logistic growth model, 30-year projection
forest-analyzer growth --input inventory.csv --years 30 --model logistic --rate 0.03 --capacity 300

# Exponential growth with custom mortality
forest-analyzer growth --input inventory.csv --model exponential --rate 0.02 --mortality 0.01

# Linear growth
forest-analyzer growth --input inventory.csv --model linear --rate 2.0
```

### Convert between formats

```bash
# CSV to JSON
forest-analyzer convert --input inventory.csv --output inventory.json --pretty

# CSV to Excel
forest-analyzer convert --input inventory.csv --output inventory.xlsx

# Excel to CSV
forest-analyzer convert --input inventory.xlsx --output inventory.csv

# CSV to GeoJSON (plots with elevation/aspect/slope as features)
forest-analyzer convert --input inventory.csv --output inventory.geojson --pretty
```

### Batch analysis

```bash
# Analyze all inventory files in a directory, output JSON reports
forest-analyzer analyze-batch --input-dir ./inventories/ --output-dir ./reports/
```

### Quick summary

```bash
forest-analyzer summary --input inventory.csv
```

### Web UI

```bash
# Start the web server (default port 8080)
forest-analyzer serve

# Custom port
forest-analyzer serve --port 3000
```

Then open `http://localhost:8080` in a browser. From there you can:
- Upload CSV, JSON, and Excel files
- Edit data in the browser, with validation
- View stand metrics, statistics, and growth charts
- Export results as CSV, JSON, or GeoJSON

## Examples

Runnable examples are in the `examples/` directory:

```bash
# Basic analysis — load CSV, compute metrics, display tables and histogram
cargo run --example basic_analysis

# Growth projection — logistic and exponential models over 20 years
cargo run --example growth_projection

# Format conversion — CSV to JSON and Excel with round-trip verification
cargo run --example format_conversion
```

## CSV format

CSV input takes these columns:

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| plot_id | integer | Yes | Plot identifier |
| tree_id | integer | Yes | Tree identifier within plot |
| species_code | string | Yes | Species code (e.g., "DF") |
| species_name | string | Yes | Common name (e.g., "Douglas-fir") |
| dbh | float | Yes | Diameter at breast height (inches) |
| height | float | No | Total height (feet) |
| crown_ratio | float | No | Crown ratio (0.0 - 1.0) |
| status | string | Yes | Live, Dead, Cut, or Missing |
| expansion_factor | float | Yes | Trees represented per sample tree |
| age | integer | No | Age at breast height |
| defect | float | No | Defect percentage (0.0 - 1.0) |
| plot_size_acres | float | No | Plot size in acres (default: 0.2) |
| slope_percent | float | No | Slope percentage |
| aspect_degrees | float | No | Aspect in degrees |
| elevation_ft | float | No | Elevation in feet |

## Configuration

An optional `config.toml` sets persistent defaults. Every field is optional.

```toml
[server]
bind_address = "127.0.0.1"   # loopback by default; --bind overrides
port = 8080
max_upload_bytes = 52428800   # 50 MB

[analysis]
confidence_level = 0.95
diameter_class_width = 2.0

[growth]
default_model = "logistic"
annual_rate = 0.03
carrying_capacity = 300.0
mortality_rate = 0.005

[database]
path = "forest_analyzer.db"
```

Pass a custom config file with `--config path/to/config.toml` (defaults to `config.toml` in the current directory).

## Library Usage

The same functions are available as a Rust library. The example below loads a CSV, compiles stand metrics and prints the basal area confidence interval.

```rust
use forest_inventory_analyzer::{
    Analyzer, ForestInventory, Tree, Plot, Species, TreeStatus,
    GrowthModel, StandMetrics, SamplingStatistics,
    io::{CsvFormat, InventoryReader},
};

fn main() -> anyhow::Result<()> {
    // Load data
    let inventory = CsvFormat.read(std::path::Path::new("inventory.csv"))?;

    // Compute metrics via the Analyzer
    let analyzer = Analyzer::new(&inventory);
    let metrics = analyzer.stand_metrics();
    println!("TPA: {:.1}", metrics.total_tpa);
    println!("Basal Area: {:.1} sq ft/ac", metrics.total_basal_area);

    // Statistical analysis
    let stats = analyzer.sampling_statistics(0.95)?;
    println!("BA 95% CI: {:.1} - {:.1}",
        stats.basal_area.lower, stats.basal_area.upper);

    Ok(())
}
```

## Development

```bash
# Run tests
cargo test --all-features

# Run clippy
cargo clippy --all-features

# Format code
cargo fmt

# Build documentation
cargo doc --open
```

## Methods and limitations

Basal area, trees per acre, QMD and the sampling statistics use the standard forms. BA = 0.005454 x DBH^2 per tree. TPA = BAF / BA_tree for variable-radius plots. QMD = sqrt(BA / (0.005454 x TPA)). The confidence interval is a two-sided Student's t interval on plot means, df = n - 1.

Volume equations are generic placeholders, not published regional equations. The built-in cubic-foot form is V = 0.002454 x DBH^2 x H, a combined-variable approximation with total height. The board-foot form labelled "Scribner" is V = 0.01159 x DBH^2 x H - 4 x DBH. Neither is taken from a published species or regional table. For any real appraisal, replace them through `VolumeEquation` with the equations your region uses (for example the PNW-FIA tarif or regional Scribner equations) before relying on the volume columns.

Growth projections are illustrative curves (linear, exponential, logistic with a mortality term). They are not a calibrated growth-and-yield model such as FVS.

Installers are unsigned. Windows SmartScreen and macOS Gatekeeper will warn on first launch. Check the download against the `.sha256` file published beside each asset.

## License

MIT, see [LICENSE](LICENSE). Release notes are in [CHANGELOG.md](CHANGELOG.md); design notes in [docs/architecture.md](docs/architecture.md).
