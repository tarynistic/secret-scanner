# secret-scanner

A command-line tool that scans code for hardcoded API keys, tokens, and credentials, and for a few common injection patterns, so they can be caught before they are committed or shipped. Built as a collaborative software engineering project.

Matched secrets are always **redacted** in the output (`AKIA****************`), so the report never becomes a second leak.

# Steps to Run
On macOS, `python3 -m secret_scanner` runs the project. The target rust file should exist just outside the secret_scanner directory as long as they're existing within the same folder 
### *Example*
`"Secret Scanner"/secret_scanner` and `"Secret Scanner"/vulnerable_sample.rs`.

> **Status: work in progress (v0.1.0).** The scanning pipeline (rules, entropy analysis, redaction, text/JSON reports) is written, but the project does not run straight from a fresh clone yet. See [Running it](#running-it) and [Known issues](#known-issues).

## What it detects

**Known secrets** (pattern rules, severity in brackets)

| Rule | Looks for | Severity |
|---|---|---|
| AWS Access Key | `AKIA` + 16 uppercase letters/digits | high |
| GitHub Token | `ghp_` + 36 alphanumerics | high |
| OpenAI API Key | `sk-` + 48 alphanumerics | high |
| JWT | `eyJ….eyJ….signature` | medium |

**Unknown secrets (entropy analysis).** Quoted or assigned strings of 20+ characters that look random (Shannon entropy above 4.5 for base64-style strings, 3.0 for hex). UUIDs, lockfile hashes (`sha512-…`), and placeholders like `YOUR_API_KEY_HERE` are ignored.

**Injection patterns**, checked only in the matching file types:

| Rule | File types | Severity |
|---|---|---|
| SQL injection | `.py` `.java` `.js` `.rs` | high |
| OS command injection | `.py` `.java` `.rs` | high |
| Code injection (`eval`/`exec`) | `.py` `.js` | high |
| Path traversal | `.py` `.java` `.js` `.rs` | medium |
| XSS | `.js` `.html` | medium |

Rules are plain data in `rules.py`. To support a new provider, add one `Rule(...)` entry to `RuleTable`; no scanner code changes.

## Requirements

- Python 3.10 or newer (the code uses `X | None` type syntax)
- `numpy` and `pandas` (currently imported by `fileWalkerV1.py`; see [Known issues](#known-issues))

```bash
pip install numpy pandas
```

## Running it

The modules use relative imports (`from .config import Config`), so they must be run as a **package**, not as loose scripts. Running `python cli.py` or `python ScannerV1.py` directly fails with `attempted relative import with no known parent package`.

Set it up once from the repo root:

```bash
git clone https://github.com/tarynistic/secret-scanner.git
cd secret-scanner

# put the modules in a package folder
mkdir secret_scan
cp *.py secret_scan/
touch secret_scan/__init__.py

# add an entry point
cat > secret_scan/__main__.py <<'EOF'
import sys
from .cli import Cli

sys.exit(Cli().run())
EOF
```

Then run it:

```bash
python -m secret_scan --help
python -m secret_scan .                          # scan the current directory
python -m secret_scan path/to/project            # scan a folder
python -m secret_scan path/to/file.py            # scan one file
python -m secret_scan . --format json            # machine-readable output
```

`vulnerable_sample.rs` in this repo contains fake keys and is a handy test target:

```bash
python -m secret_scan vulnerable_sample.rs
```

Once the [known issues](#known-issues) are fixed, you can also install it as a `secret-scan` command by adding a `pyproject.toml` with a console-script entry point.

## Options

| Option | Default | What it does |
|---|---|---|
| `PATH` | `.` | File or directory to scan |
| `--format {text,json}` | `text` | Output format |
| `--entropy-threshold FLOAT` | `4.5` | Entropy cutoff for base64-style strings; higher means fewer results |
| `--no-entropy` | off | Only use the known-pattern rules |
| `--no-injection` | off | Skip the injection rules |
| `--ignore GLOB` | none | Skip files matching a glob such as `"*.lock"`; repeatable |
| `--max-file-size BYTES` | `1000000` | Skip files larger than this (counted as skipped) |
| `--version` | | Print the version and exit |
| `--help` | | Show usage and exit |

Always skipped: `.git`, `target`, and `node_modules` directories, and binary files.

## Output

Text output (the default) groups findings by file, then prints any warnings and a summary:

```
src/config.py
  line 2: [HIGH] AWS Access Key  AKIA****************

Summary
  files scanned: 4
  files skipped: 1
  findings:      1
    AWS Access Key: 1
  elapsed:       0.01s
```

`--format json` returns the same data as `findings`, `warnings`, and `summary` objects, suitable for CI or other tooling.

### Exit codes

| Code | Meaning |
|---|---|
| `0` | Scan finished, nothing found |
| `1` | Scan finished, one or more findings |
| `2` | Error (path not found, invalid option, or output could not be written) |

This makes it usable as a CI gate: a non-zero exit fails the build.

## Silencing a line

Add `// secret-scan:ignore` to the end of a line to skip it entirely (secrets and injection rules).

```rust
let test_key = "AKIAIOSFODNN7EXAMPLE"; // secret-scan:ignore
```

Unreadable files produce a warning and the scan continues. Non-UTF-8 bytes are replaced rather than crashing the scan.

## Project layout

| File | Role |
|---|---|
| `cli.py` | Argument parsing, runs the scan, prints the report, returns the exit code |
| `config.py` | `Config` object holding every option |
| `ScannerV1.py` | `Scanner`: walks files, reads lines, runs rules, builds the report |
| `fileWalkerV1.py` | Finds files; skips excluded, binary, and oversized ones |
| `lineReader.py` | Reads one line at a time so big files never sit in memory |
| `rules.py` | Rule table (the detection patterns) |
| `rule_engine.py` | Compiles the rules and matches them against each line |
| `entropy.py` | Entropy-based detection and the false-positive allowlist |
| `redactor.py` | Masks secret values in output |
| `suppression.py` | `secret-scan:ignore` handling |
| `finding.py`, `report.py` | Data classes for findings and the scan report |
| `reporters.py` | Text and JSON output |
| `runOrder.md` | Step-by-step walkthrough of a run, numbered to match the `# STEP n` comments in the code |
| `vulnerable_sample.rs` | Sample file with fake secrets for testing |

New to the code? Read `runOrder.md` first; it traces one full run through every file.

> Renaming a file means updating every module that imports it.

## Known issues

These stop a fresh clone from producing a scan, and are the next things to fix:

1. **No package or entry point.** The code imports relatively but there is no `secret_scan/` folder or `__main__.py` (`runOrder.md` expects both). The setup above is a workaround.
2. **`fileWalkerV1.py` is unfinished.** `is_excluded()` and `is_binary()` still raise `NotImplementedError`, so any scan crashes. `too_large()` is defined twice (the second version, which handles broken symlinks, is the one in effect).
3. **`rules.py` / `rule_engine.py` mismatch.** `RuleEngine` calls `rule.applies_to(ext)`, but `InjectionRule.applies_to` takes `(line, file_extension)`, which raises a `TypeError` on the first line scanned.
4. **Stray code in `fileWalkerV1.py`.** The module prints `Script name:` and its arguments when imported, defines an unused `validFiles()` helper, and imports `numpy`/`pandas` without using them. Removing these also removes the numpy/pandas requirement.
5. **Tests are empty.** `test_filegrabber.py` and `test_main.py` contain only docstrings.
6. **Suppression is `//`-only.** `# secret-scan:ignore` is not yet recognized in Python files.

## Contributing

1. Branch from `main`.
2. Keep detection logic in `rules.py` where possible.
3. Add tests for any new rule or option.
4. Open a pull request.

# Steps to Run
On macOS, `python3 -m secret_scanner` runs the project. The target rust file should exist just outside the secret_scanner directory as long as they're existing within the same folder 
### *Example*
`"Secret Scanner"/secret_scanner` and `"Secret Scanner"/vulnerable_sample.rs`.