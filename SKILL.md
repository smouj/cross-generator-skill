---
name: cross-generator
version: 1.2.0
description: AI-powered analysis generator that transforms raw data into structured insights, reports, and visualizations
author: SMOUJBOT
tags:
  - analysis
  - ai
  - automation
  - reporting
  - data-processing
maintainer: ops@openclaw.ai
last_updated: 2026-03-02
category: generators
dependencies:
  - openclaw-core >= 2.1.0
  - analysis-engine >= 1.4.0
  - report-renderer >= 2.0.0
  - python >= 3.9
  - pandas >= 1.5.0
  - matplotlib >= 3.6.0
  - numpy >= 1.24.0
required_env_vars:
  - CROSS_GEN_API_KEY (optional, for enhanced AI models)
  - ANALYSIS_OUTPUT_DIR (default: ./analysis-output)
  - CROSS_GEN_LOG_LEVEL (default: INFO)
compatible_platforms:
  - linux
  - darwin
  - windows
conflicts_with:
  - legacy-report-generator
  - manual-analyzer
---

# Cross Generator Skill

## Purpose

Cross Generator is an AI-powered analysis engine that automates the conversion of raw data into actionable insights, comprehensive reports, and visualizations. It eliminates manual data wrangling by intelligently detecting patterns, generating statistical summaries, and producing publication-ready outputs across multiple formats.

**Real Use Cases:**
- Convert system metrics logs (CPU, memory, network) into performance trend reports with anomaly detection
- Transform CSV/JSON datasets into executive summaries with key metrics and recommendations
- Generate security audit reports from firewall logs and access patterns
- Create dependency analysis reports from package.json/requirements.txt changes
- Automate weekly operational status reports from monitoring data with predictive insights

## Scope

Cross Generator operates within a confined workspace and produces analysis artifacts without modifying source data. Commands are executed through the OpenClaw CLI wrapper.

**Supported Input Formats:**
- CSV, TSV, JSON, YAML
- Log files (plain text, syslog, JSON logs)
- JSON Lines (NDJSON)
- SQLite databases (read-only)
- Directory structures with patterned files

**Output Formats:**
- Markdown reports with embedded charts
- HTML dashboards with interactive Plotly charts
- PDF (via LaTeX or wkhtmltopdf)
- JSON structured analysis results
- PNG/SVG visualization assets
- Excel/Workbook (.xlsx) with multiple sheets

**Commands:**

### `cross-gen analyze`
Main analysis command with AI-powered interpretation.

```
cross-gen analyze --input <path> --output <dir> [options]
```

**Flags:**
- `--input`, `-i`: Source file or directory (required)
- `--output`, `-o`: Output directory (default: current)
- `--type`, `-t`: Analysis type: `metrics`, `security`, `dependencies`, `custom`
- `--ai-model`, `-m`: AI model: `gpt-4`, `claude-3`, `local` (default: auto)
- `--format`, `-f`: Output format: `markdown`, `html`, `pdf`, `json` (default: markdown)
- `--template`, `-T`: Custom template file (optional)
- `--threshold`, `-th`: Anomaly detection threshold (0.01-0.99, default: 0.95)
- `--include-charts`, `-c`: Generate visualizations (default: true)
- `--summary-only`, `-s`: Skip detailed sections (faster)
- `--context`, `-ctx`: Additional context string for AI interpretation

### `cross-gen validate`
Pre-flight validation of input data.

```
cross-gen validate --input <path>
```

### `cross-gen list-reports`
List previously generated reports in output directory.

```
cross-gen list-reports --dir <path>
```

### `cross-gen compare`
Compare two analysis runs to detect changes.

```
cross-gen compare --baseline <file1> --current <file2> --output <dir>
```

## Work Process

1. **Initialization (1-2 seconds)**
   - Load configuration from `~/.openclaw/config/cross-gen.yaml` if exists
   - Verify dependencies: pandas, matplotlib, Jinja2 templates
   - Check API credentials if using remote AI models
   - Create timestamped output directory: `output/YYYY-MM-DD_HH-MM-SS/`

2. **Data Ingestion (depends on size)**
   - Detect input format by file extension or magic bytes
   - Stream large files (>100MB) to avoid memory pressure
   - Parse into standardized pandas DataFrame or JSON structure
   - Validate schema: required columns, data types, null counts
   - Log ingestion metrics: rows=1234, cols=12, memory=45MB

3. **Statistical Foundation (1-5 seconds)**
   - Compute descriptive statistics: mean, median, std, quartiles
   - Detect outliers using IQR or z-score (configurable)
   - Correlation matrix for numeric columns
   - Missing data analysis and imputation strategy
   - Trend detection: linear regression, moving averages

4. **AI Interpretation (3-15 seconds)**
   - Construct prompt with:
     - Data summary (shape, columns, sample rows)
     - Statistical highlights
     - Context from `--context` flag
     - Analysis type instructions
   - Call AI model (OpenAI, Anthropic, or local via Ollama)
   - Extract structured insights: key_findings, anomalies, recommendations
   - Cache results in `.cross-gen-cache/` to avoid re-running on identical data

5. **Report Generation (2-10 seconds)**
   - Load appropriate Jinja2 template based on format/type
   - Inject AI insights, statistics, and metadata
   - Render charts if enabled:
     - Time series: line plots with confidence bands
     - Distributions: histograms + KDE
     - Correlations: heatmap or scatter matrix
     - Categorical: bar charts with percentage labels
   - Save to output directory with timestamp prefix

6. **Validation & Packaging**
   - Verify generated files exist and are non-empty
   - Compute SHA256 checksums for reproducibility
   - Write manifest.json with file list and metadata
   - Optional: upload to S3 if `CROSS_GEN_S3_BUCKET` set

## Golden Rules

1. **Never modify source data** - Cross Generator operates strictly read-only on inputs. All outputs go to dedicated directories with timestamps.

2. **Cache aggressively** - The `.cross-gen-cache/` directory must be preserved between runs. If `INPUT_SHA256` matches cached data, skip AI call and reuse previous insights.

3. **Respect memory limits** - For files >500MB, force streaming mode. Fail gracefully with "memory limit exceeded" rather than OOM kills.

4. **Anonymize sensitive data** - If column names match patterns like `*password*`, `*token*`, `*secret*`, automatically redact values from logs and reports. Log redaction event at WARN level.

5. **Thresholds must be configurable** - Never hardcode anomaly detection threshold. Always read from `--threshold` or config file.

6. **Fail fast on schema mismatch** - If required columns missing, abort before AI call. Return exit code 2 with clear message: "Missing required column: 'timestamp'".

7. **Charts must be reproducible** - Set random seed `np.random.seed(42)` for all visualizations. Include data source and generation timestamp in chart footer.

8. **Exit codes are contracts**:
   - 0 = success
   - 1 = general error (with stderr message)
   - 2 = validation failure (missing deps, bad input)
   - 3 = AI service unavailable
   - 4 = quota exceeded

## Examples

**Example 1: Generate security audit from auth logs**

Input: `logs/auth.jsonl` (JSON Lines format)
Command:
```bash
cross-gen analyze \
  --input logs/auth.jsonl \
  --output reports/security-$(date +%Y%m%d) \
  --type security \
  --format html \
  --threshold 0.98 \
  --context "Weekly security review for PCI compliance"
```

Output structure:
```
reports/security-20260302/
├── index.html              # Main dashboard
├── charts/
│   ├── login_attempts_by_ip.png
│   ├── failed_auth_trends.svg
│   └── geographic_distribution.html  # Interactive Plotly
├── insights.json           # AI-generated findings
├── statistics.csv          # Raw numbers for audit
└── manifest.json           # Checksums and metadata
```

AI-generated insights excerpt (from `insights.json`):
```json
{
  "key_findings": [
    "Unusual spike in failed logins from IP 192.168.1.105: 1,247 attempts (99th percentile)",
    "Geographic spread expanded to 12 new countries vs last week",
    "Average session duration decreased by 23%, potential session hijacking"
  ],
  "anomalies_detected": 7,
  "recommendations": [
    "Block IP 192.168.1.105 at firewall immediately",
    "Enable MFA requirement for admin panel",
    "Review geolocation allowlist for non-business regions"
  ]
}
```

**Example 2: Compare dependency changes**

Command:
```bash
# Generate baseline (last week)
cross-gen analyze -i package.json -o deps-baseline -t dependencies

# Generate current
cross-gen analyze -i package.json -o deps-current -t dependencies

# Compare
cross-gen compare --baseline deps-baseline/insights.json \
                 --current deps-current/insights.json \
                 --output deps-delta
```

Output `deps-delta/delta.md`:
```markdown
# Dependency Change Analysis

## Critical Updates
- **lodash** upgraded from 4.17.21 to 4.17.25 (CVE-2023-XXXX patched)
- **express** downgraded from 4.18.2 to 4.17.3 (introduced regression)

## New Dependencies Added
1. `winston` (3.8.2) - logging infrastructure
2. `helmet` (7.0.0) - security headers (PASS)

## Removed Dependencies
- `debug` (no longer needed after Winston integration)

## Risk Score: MEDIUM (6/10)
Recommended: Lock `express` at 4.18.2 via `npm install express@4.18.2`
```

**Example 3: Real-time metrics streaming**

Input: Directory of metrics files collected every 5 minutes
Command:
```bash
cross-gen analyze \
  --input metrics/20260302/ \
  --type metrics \
  --output reports/realtime-$(date +%H%M) \
  --format markdown \
  --include-charts \
  --summary-only \
  --context "Real-time Monolith P99 latency alert triage"
```

**Example 4: Custom analysis with manual override**

When default templates insufficient:
```bash
cross-gen analyze \
  --input data/campaign_performance.csv \
  --output reports/q1-campaign \
  --format pdf \
  --template templates/custom_marketing_report.html \
  --context "Q1 2026 product launch campaign; target ROAS 3.5"
```

Custom template variables available:
- `{{ ai_insights }}` - full JSON object
- `{{ stats_table }}` - HTML table
- `{{ chart_paths }}` - list of image paths
- `{{ metadata }}` - run timestamp, input checksum, config used

## Rollback Commands

Cross Generator generates immutable, timestamped output directories. Rollback means restoring a previous output directory to active status or removing a bad run.

**Remove a failed/bad analysis run:**
```bash
# Remove entire output directory safely (after verification)
rm -rf reports/security-20260302/
# Or move to quarantine
mv reports/security-20260302/ /tmp/quarantine/
```

**Restore previous report as current:**
```bash
# Symlink previous successful run as 'current'
ln -sfn reports/security-20260301 reports/current
# Update deployed symlink on web server
sudo ln -sfn /opt/openclaw/reports/security-20260301 /var/www/reports/latest
```

**Rollback AI cache (force recompute):**
```bash
# Clear cache for specific input file
rm .cross-gen-cache/8a3f2c1d.json

# Or clear entire cache (expensive - re-runs all analyses)
rm -rf .cross-gen-cache/
```

**Rollback configuration:**
```bash
# Restore default config
cp ~/.openclaw/config/cross-gen.default.yaml ~/.openclaw/config/cross-gen.yaml

# Or revert to version-controlled config
git checkout HEAD -- config/cross-gen.yaml
```

**Undo accidental data modification (defensive):**
Cross Generator never modifies source data. If source data corruption detected:
```bash
# Restore from backup (assumes snapshots exist)
rsync -av /backup/raw-data/20260301/ logs/
```

**Purge all generated reports (emergency cleanup):**
```bash
# List before deleting
find reports/ -maxdepth 1 -type d -name "20*" | sort
# Delete older than 30 days
find reports/ -maxdepth 1 -type d -name "20*" -mtime +30 -exec rm -rf {} \;
```

## Dependencies & Requirements

**Python packages (auto-installed via `cross-gen setup`):**
```
pandas>=1.5.0
numpy>=1.24.0
matplotlib>=3.6.0
seaborn>=0.12.0
plotly>=5.14.0
jinja2>=3.1.0
openpyxl>=3.0.0  # for Excel output
pyyaml>=6.0
requests>=2.28.0
```

**System dependencies:**
- `wkhtmltopdf` (optional, for PDF generation)
- `pdflatex` (optional, LaTeX PDF output)
- `ffmpeg` (optional, for animated charts)
- 512MB RAM minimum, 2GB recommended
- 500MB disk space for cache + output

**First-time setup:**
```bash
cross-gen setup
# Creates:
# - ~/.openclaw/config/cross-gen.yaml
# - ~/.openclaw/templates/cross-gen/
# - .cross-gen-cache/ directory
```

## Verification Steps

**Post-installation check:**
```bash
# 1. Verify CLI accessible
cross-gen --version
# Expected: cross-gen 1.2.0

# 2. Check dependencies
cross-gen doctor
# Output should include: [OK] pandas 1.5.3, [OK] matplotlib 3.6.2

# 3. Test with sample data
cross-gen analyze -i samples/sample_metrics.csv -o /tmp/test-output
# Verify: /tmp/test-output/index.html exists, non-zero size

# 4. Validate AI connectivity (if using remote)
cross-gen ping-ai
# Expected: "AI service reachable: gpt-4 (latency 2.3s)"
```

**On every run, Cross Generator writes to its log at `~/.openclaw/logs/cross-gen.log`. Check for:**
```
[INFO] Analysis started: input=..., output=...
[INFO] Ingested 15,423 rows, 8 columns in 2.1s
[INFO] AI interpretation completed: 3 key_findings, 2 anomalies
[INFO] Report generated: /path/to/output/index.html (1.2MB)
[INFO] SHA256 manifest: abc123... verified
```

**Verify output integrity:**
```bash
# Check manifest checksums
cd reports/latest/
sha256sum -c manifest.json.sha256  # should show "OK"

# Validate JSON structure
python -c "import json; json.load(open('insights.json'))"
# Should not raise exception

# Test HTML renders (quick syntax check)
grep -q "</html>" index.html && echo "HTML structure valid"
```

## Troubleshooting

**"MemoryError: Unable to allocate"**
- Cause: Input file too large for memory
- Fix: Use streaming mode (automatic for >100MB), or split input:
  ```bash
  split -l 100000 large.csv chunk_
  for f in chunk_*; do cross-gen analyze -i $f -o chunks/$(basename $f .csv); done
  ```

**"AI service unavailable: rate limit exceeded"**
- Cause: Too many requests in short period
- Fix: Increase `--context` caching (automatic), or switch to local model:
  ```bash
  cross-gen analyze -i data.csv -o out --ai-model local
  ```
  Requires Ollama running locally with `llama2` model.

**"Validation failed: Missing required column 'timestamp'"**
- Cause: Input schema mismatch
- Fix: Rename column or specify custom mapping:
  ```bash
  cross-gen analyze -i data.csv -o out --column-map '{"time":"timestamp"}'
  ```

**"PDF generation failed: wkhtmltopdf not found"**
- Cause: Missing system dependency
- Fix: Install wkhtmltopdf or use HTML output:
  ```bash
  sudo apt-get install wkhtmltopdf  # Ubuntu
  brew install wkhtmltopdf          # macOS
  ```
  Alternative: `cross-gen analyze --format html`

**Charts appear blank or missing**
- Cause: matplotlib backend issue in headless environment
- Fix: Set backend explicitly in config:
  ```yaml
  ~/.openclaw/config/cross-gen.yaml:
  matplotlib_backend: Agg
  ```

**"Permission denied: .cross-gen-cache/"**
- Cause: Cache directory owned by different user (sudo usage)
- Fix: Reset ownership:
  ```bash
  sudo chown -R $(whoami) ~/.openclaw/.cross-gen-cache/
  ```

**Slow performance on large datasets**
- Enable compression in config:
  ```yaml
  compression: true  # uses parquet intermediate format
  chunk_size: 50000  # process in chunks
  ```
- Or reduce chart resolution: `--chart-dpi 72`

**Silent failures (exit 0 but empty output)**
- Check log: `tail -50 ~/.openclaw/logs/cross-gen.log`
- Likely cause: input file empty or zero rows after filtering
- Add `--verbose` flag to see processing steps

---

**Last validated:** 2026-03-02 with OpenClaw 2.1.0 and Cross Generator 1.2.0
```