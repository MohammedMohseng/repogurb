
RepoGuard — Software Requirements Specification

Version: 1.0
Language: English
Implementation Language: Rust
Primary Interface: CLI
Secondary Interface: HTTP API
Repository Sources: Local filesystem, GitHub API, Git clone
Primary Purpose: Secret and sensitive-data detection in source repositories


---

1. Introduction

1.1 Purpose

RepoGuard is a fast security scanning tool designed to detect accidentally exposed secrets and sensitive information inside software repositories.

The system shall support:

1. Local repository scanning.


2. GitHub repository scanning through the GitHub REST API.


3. GitHub repository scanning through Git clone.


4. Command-line usage.


5. HTTP API usage for future web applications.


6. Deterministic security rules without requiring an AI/LLM.


7. Optional AI-assisted analysis in a future phase.



The primary use case is preventing developers from accidentally committing sensitive information such as:

API keys.

Database credentials.

Database connection strings.

JWT secrets.

Cloud credentials.

Private keys.

Passwords.

Tokens.

.env files.

Other high-entropy secrets.



---

2. Goals

RepoGuard shall be:

2.1 Fast

The scanner should efficiently process large repositories using Rust's performance characteristics, streaming, bounded concurrency, and parallel scanning where appropriate.

2.2 Secure

Repositories must be treated as untrusted input.

RepoGuard must never execute source code, scripts, package managers, or repository-provided commands.

2.3 Extensible

Detection rules must be modular so new secret types can be added without rewriting the scanner.

2.4 Source-independent

The scanning engine must not care whether files came from:

Local filesystem
GitHub API
Git clone

2.5 Reusable

The same scanning engine must power:

CLI
HTTP API
Future Web Application
CI/CD integrations


---

3. Non-Goals

Version 1.0 shall not attempt to:

Perform full vulnerability analysis.

Execute application code.

Perform penetration testing.

Exploit vulnerabilities.

Automatically rotate/revoke credentials.

Automatically modify repositories.

Automatically delete secrets.

Depend on an LLM for core detection.

Scan arbitrary internet URLs through the HTTP API.

Analyze every possible programming-language vulnerability.


RepoGuard is primarily a secret and sensitive-data scanner.


---

4. Target Architecture

The system shall follow this architecture:

┌─────────────────┐
                    │      CLI        │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Application     │
                    │ Services        │
                    └────────┬────────┘
                             │
                 ┌───────────▼───────────┐
                 │ Repository Source     │
                 │ Abstraction           │
                 └───────────┬───────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
          Local FS       GitHub API      Git Clone
              │              │              │
              └──────────────┼──────────────┘
                             ▼
                    ┌─────────────────┐
                    │ Scanner Core    │
                    └────────┬────────┘
                             ▼
                    ┌─────────────────┐
                    │ Detection       │
                    │ Engine          │
                    └────────┬────────┘
                             ▼
                    ┌─────────────────┐
                    │ Risk Engine     │
                    └────────┬────────┘
                             ▼
                    ┌─────────────────┐
                    │ Findings        │
                    └────────┬────────┘
                             │
                  ┌──────────┴──────────┐
                  ▼                     ▼
                CLI                 HTTP API


---

5. Technology Requirements

The implementation language MUST be Rust.

Recommended stack:

Component	Technology

Language	Rust
Async runtime	Tokio
CLI	Clap
HTTP server	Axum
HTTP client	Reqwest
Serialization	Serde
JSON	Serde JSON
Regex	Regex
Filesystem traversal	Walkdir / Ignore
Parallel processing	Rayon where appropriate
Temporary directories	Tempfile
Logging	Tracing
Errors	Thiserror + Anyhow
Testing	Rust built-in testing + integration tests


The agent may select alternatives only when there is a strong technical reason.


---

6. Project Structure

The implementation shall use a Cargo workspace.

Initial target structure:

repoguard/
│
├── Cargo.toml
│
├── crates/
│   ├── core/
│   ├── sources/
│   ├── cli/
│   └── api/
│
├── rules/
│
├── fixtures/
│
├── tests/
│
├── docs/
│
├── README.md
├── SECURITY.md
├── CONTRIBUTING.md
└── LICENSE

The exact internal module structure may evolve as long as architectural boundaries remain clear.


---

7. Core Domain Models

The system shall define structured models for:

Repository
RepositorySource
File
Finding
Rule
Severity
Confidence
Scan
ScanStatus
ScanOptions
ScanResult

Example conceptual Finding:

{
  "id": "finding-001",
  "rule_id": "postgres-url",
  "severity": "high",
  "confidence": 0.97,
  "file": "src/config.ts",
  "line": 14,
  "column": 20,
  "masked_match": "postgres://user:********@host/db",
  "message": "Potential database credential detected.",
  "remediation": "Move the credential to a secure environment variable."
}

Raw secret values must never be included in normal output.


---

8. Severity Levels

The system shall support:

CRITICAL
HIGH
MEDIUM
LOW
INFO

Example baseline:

Finding	Severity

Private key	CRITICAL
Production database credential	CRITICAL
Cloud secret	HIGH
API token	HIGH
Generic secret assignment	MEDIUM
Suspicious high-entropy value	MEDIUM
.env file	LOW/HIGH depending on contents
.env.example	INFO


The final severity must be determined by the rule and contextual confidence.


---

9. Repository Sources

9.1 Local Source

The CLI shall support:

repoguard scan .

and:

repoguard scan ./project

The local scanner shall recursively inspect files.


---

10. File Filtering

RepoGuard shall skip irrelevant files and directories by default.

Examples:

.git/
node_modules/
target/
dist/
build/
.next/
coverage/
vendor/

The scanner shall also:

Detect binary files.

Avoid scanning binary files by default.

Enforce configurable file-size limits.

Handle symbolic links safely.

Avoid path traversal.

Never execute files.


.env files must not be ignored.

Examples:

.env
.env.local
.env.production
.env.development

must be explicitly inspected.

.env.example should normally be allowed but may still be scanned for suspicious real credentials.


---

11. Detection Engine

The detection engine shall consist of independent detectors.

Initial detectors:

Environment File Detector
API Key Detector
Database URL Detector
JWT Detector
Cloud Credential Detector
GitHub Token Detector
Private Key Detector
Generic Secret Detector
Password Detector
High Entropy Detector

Each detector should produce standardized Findings.


---

12. Rule Engine

Rules shall be modular.

A rule conceptually contains:

Rule ID
Name
Description
Severity
Confidence
Patterns
File Constraints
Exclusions
Remediation

Example:

Rule:
    id: postgres-url
    name: PostgreSQL connection string
    severity: HIGH
    patterns:
        postgres://
        postgresql://

Rules must be easy to extend.


---

13. Regex Detection

Regex shall be used for deterministic patterns.

Examples include:

postgresql://
postgres://
mongodb://
mongodb+srv://
mysql://
redis://

and known token formats.

Regex patterns must be compiled once and reused.


---

14. Generic Secret Detection

The scanner shall detect suspicious assignments such as:

API_KEY="..."
SECRET_KEY="..."
JWT_SECRET="..."
PASSWORD="..."
TOKEN="..."
CLIENT_SECRET="..."
DATABASE_URL="..."

However, the scanner must attempt to reduce false positives.

For example:

API_KEY="your-api-key-here"

should not automatically be treated as a confirmed secret.


---

15. Entropy Detection

The scanner shall optionally calculate entropy for suspicious strings.

The entropy detector shall:

1. Ignore very short strings.


2. Ignore common words.


3. Consider character distribution.


4. Combine entropy with contextual information.


5. Produce a confidence score.



Entropy alone must not automatically classify a value as a secret.


---

16. False Positive Handling

Every Finding should contain a confidence score.

Example:

confidence: 0.98

means high confidence.

The system should support suppression mechanisms in future versions.

Potential future syntax:

# repoguard:ignore rule-id

This should not be implemented until the basic scanner is stable.


---

17. Secret Masking

This is a strict security requirement.

Given:

DATABASE_URL=postgresql://admin:SuperSecret123@db.example.com/app

the output must never expose:

SuperSecret123

Instead:

DATABASE_URL=postgresql://admin:********@db.example.com/app

Logs must also avoid secret values.


---

18. GitHub API Source

RepoGuard shall support scanning public and authenticated GitHub repositories.

Conceptual command:

repoguard github owner/repo --source api

The GitHub source should:

1. Resolve repository metadata.


2. Resolve branch/ref.


3. Retrieve repository tree.


4. Filter files.


5. Retrieve relevant file contents.


6. Scan them.


7. Produce Findings.



The implementation should prefer Git tree/blob operations over making an individual directory traversal request for every directory.


---

19. GitHub Authentication

Authentication shall support an environment variable:

GITHUB_TOKEN

Future versions may support:

repoguard auth login

Tokens must:

Never be printed.

Never be written into scan results.

Never be stored in repository files.

Never appear in error messages.

Never be returned through the HTTP API.



---

20. GitHub Rate Limits

The GitHub client must detect:

401
403
429

and rate-limit information.

It shall provide useful errors such as:

GitHub API rate limit exceeded.
Try again later or configure GITHUB_TOKEN.

The client shall implement bounded retries only where appropriate.


---

21. Git Clone Source

RepoGuard shall support:

repoguard github owner/repo --source clone

The clone process should:

Validate repository
        ↓
Create temporary directory
        ↓
Shallow clone
        ↓
Scan filesystem
        ↓
Generate results
        ↓
Cleanup temporary directory

The default should use a shallow clone.


---

22. Untrusted Repository Security

A cloned repository must be treated as hostile input.

RepoGuard must never:

execute scripts
run npm install
run cargo build
run package managers
run Makefiles
execute shell commands from repository content

The scanner must also defend against:

Huge files.

Huge repositories.

Malicious filenames.

Symbolic links.

Path traversal.

Resource exhaustion.

Unexpected encodings.

Binary files.



---

23. CLI Requirements

The initial CLI shall provide:

repoguard scan <PATH>

repoguard github <OWNER/REPO>

repoguard github <OWNER/REPO> --source api

repoguard github <OWNER/REPO> --source clone

repoguard scan <PATH> --format json

repoguard scan <PATH> --fail-on high


---

24. CLI Output

Human-readable output:

RepoGuard

Scanning repository...
Files scanned: 1,284

Findings:

CRITICAL  src/config.ts:14
          PostgreSQL credential detected
          postgres://admin:********@db.example.com/app

HIGH      src/api.ts:8
          API key detected
          sk_live_****************

LOW       .env.production
          Environment file detected

────────────────────────────────
3 findings
1 critical
1 high
1 low


---

25. JSON Output

The CLI shall support:

repoguard scan . --format json

The output must be machine-readable and stable enough for CI/CD integrations.


---

26. Exit Codes

The CLI shall use:

0 = scan successful and no findings above threshold
1 = findings detected above threshold
2 = scanner/configuration/runtime error

Example:

repoguard scan . --fail-on high

If a HIGH finding exists:

exit code = 1


---

27. HTTP API

The API shall expose the same scanning engine.

Initial endpoints:

GET /health

POST /v1/scans

GET /v1/scans/{scan_id}

GET /v1/scans/{scan_id}/findings


---

28. Scan API

Example:

POST /v1/scans
Content-Type: application/json

{
  "provider": "github",
  "owner": "owner",
  "repository": "repo",
  "ref": "main",
  "source": "api"
}

Response:

{
  "scan_id": "scan_123",
  "status": "queued"
}


---

29. Scan Lifecycle

A scan shall have states:

QUEUED
RUNNING
COMPLETED
FAILED
CANCELLED

Conceptual lifecycle:

Request
  ↓
Validation
  ↓
Queue
  ↓
Running
  ↓
Source acquisition
  ↓
Scanning
  ↓
Risk analysis
  ↓
Completed


---

30. API Security

The API must:

Validate all input.

Never expose raw secrets.

Apply request limits.

Restrict repository providers.

Avoid arbitrary outbound URLs.

Prevent SSRF.

Limit scan size.

Limit concurrent scans.

Never execute repository code.


The first API version should support GitHub repositories explicitly rather than accepting arbitrary Git URLs.


---

31. Database

The CLI does not require a database.

The first HTTP API implementation may operate without persistent storage if scans remain in memory.

A future production web application may use:

PostgreSQL

for:

Users
Projects
Repositories
Scans
Findings
Rules
Scan history

Database integration is not part of the initial scanner milestone.


---

32. Performance Requirements

The scanner shall:

Avoid loading the entire repository into memory.

Stream large files where practical.

Enforce maximum file size.

Compile regexes once.

Avoid unnecessary allocations.

Use bounded concurrency.

Parallelize independent file scanning when beneficial.

Avoid scanning ignored directories.

Avoid binary files by default.


Performance optimization must happen after correctness is established.


---

33. Testing Requirements

The project must include:

Unit tests

For:

Rules
Regex detectors
Entropy detector
Severity
Confidence
Masking
File filtering

Integration tests

For:

Local repository scanning
CLI
GitHub client
Clone source
HTTP API

Security tests

For:

Path traversal
Malicious filenames
Large files
Binary files
Symlinks
Secret leakage
API abuse
SSRF protection


---

34. Test Fixtures

Create synthetic repositories:

fixtures/
├── clean/
├── api-key/
├── database-url/
├── jwt/
├── private-key/
├── env/
├── false-positive/
├── binary/
├── large-file/
├── malicious-path/
└── multiple-secrets/

All test credentials must be fake.


---

35. Git History

Git history scanning is a future feature.

Future command:

repoguard history .

It should eventually detect secrets that were deleted from the working tree but remain in Git history.

This is explicitly excluded from the first implementation phases.


---

36. CI/CD

Future usage:

- name: RepoGuard
  run: repoguard scan . --fail-on high

The tool should return a non-zero exit code when security findings exceed the configured threshold.

SARIF output should be considered in a later phase.


---

37. AI Integration

AI must be optional.

The deterministic scanner remains the source of truth.

AI may later be used for:

Finding explanation
False-positive classification
Risk prioritization
Remediation suggestions
Code-context analysis

The AI must never receive raw secrets unless explicitly designed and secured for that purpose.

Default AI input should use masked values.


---

38. Implementation Phases

The coding agent MUST implement the project in the following order.

Phase 0 — Architecture

Do not implement scanner functionality.

Create:

Cargo workspace
Architecture documentation
ADR documents
Module boundaries
Initial domain models

Then run:

cargo check
cargo test
cargo fmt --check

Stop and report.


---

Phase 1 — Core Models

Implement:

Finding
Rule
Severity
Confidence
Scan
ScanResult
RepositorySource

Add serialization.

Add unit tests.

Do not implement GitHub or CLI yet.


---

Phase 2 — Local Scanner

Implement:

Directory traversal
File filtering
Binary detection
File-size limits
Content reading

Command:

repoguard scan .

At this point the scanner can locate files but does not need every detection rule.


---

Phase 3 — Detection Engine

Implement:

.env detection
Database URLs
API keys
JWT
Private keys
Generic secrets
Entropy

Add fixtures and tests.

Implement secret masking.


---

Phase 4 — Risk Engine

Implement:

Severity
Confidence
Deduplication
Threshold filtering

Test false positives.


---

Phase 5 — CLI

Implement:

scan
github
--format
--fail-on
--verbose

Implement stable exit codes.


---

Phase 6 — GitHub API

Implement:

GitHub authentication
Repository metadata
Ref resolution
Tree retrieval
Blob retrieval
Rate-limit handling
Retries
Errors

Add mocked tests.


---

Phase 7 — Git Clone

Implement:

Repository validation
Temporary directory
Shallow clone
Ref handling
Cleanup
Error handling

Add security tests.


---

Phase 8 — HTTP API

Implement:

Axum
/health
POST /v1/scans
GET /v1/scans/{id}
/findings

Use the existing scanner core.

Do not duplicate scanner logic.


---

Phase 9 — Security Hardening

Perform a dedicated security review.

Test:

SSRF
Path traversal
Symlinks
Resource exhaustion
Huge repositories
Huge files
Malicious filenames
Secret leakage
Token leakage
Concurrency abuse


---

Phase 10 — Performance

Benchmark:

1K files
10K files
50K files
Large files
Many findings
Large repository

Optimize only where measurements identify bottlenecks.


---

Phase 11 — Documentation

Create:

README.md
CLI documentation
API documentation
Rule documentation
Security model
Configuration documentation
Contribution guide


---

39. Definition of Done

The project is considered complete only when:

✓ Rust implementation
✓ CLI works
✓ Local scanning works
✓ GitHub API scanning works
✓ Git clone scanning works
✓ Detection engine works
✓ Secrets are masked
✓ JSON output works
✓ Exit codes work
✓ HTTP API works
✓ Security tests pass
✓ Integration tests pass
✓ Documentation exists
✓ cargo fmt passes
✓ cargo clippy passes
✓ cargo test passes
✓ cargo build --release succeeds


---

40. Mandatory AI Agent Rules

The coding agent must follow these rules throughout implementation:

1. Use Rust exclusively for the backend/scanner implementation.

2. Do not implement all phases at once.

3. Before starting a phase, inspect the current repository.

4. Do not overwrite existing working code unnecessarily.

5. Do not add dependencies without justification.

6. Write tests for every security-sensitive component.

7. Never use real credentials in tests.

8. Never print discovered secrets.

9. Never execute repository code.

10. Treat GitHub repositories as untrusted input.

11. Keep the scanner core independent from CLI and HTTP concerns.

12. Do not duplicate business logic between CLI and API.

13. Run formatting, compilation and tests after every phase.

14. Fix failures before moving to the next phase.

15. Do not claim success without evidence from executed tests.

16. Document architectural decisions when changing the architecture.

17. Prefer simple implementations before optimization.

18. Optimize only after benchmarking.

19. Stop after each phase and provide a concise implementation report.

20. Never silently skip a failed requirement.


---

41. Agent Execution Protocol

الجزء ده أنا شايفه مهم جدًا عشان الـ agent ما ينطلق ويعمل ليك Frankenstein project 😂.

خليه يتصرف كده:

START
  │
  ▼
Read SRS
  │
  ▼
Inspect repository
  │
  ▼
Determine current phase
  │
  ▼
Create implementation plan
  │
  ▼
Implement ONLY current phase
  │
  ▼
Run tests
  │
  ├── FAIL → Fix
  │            │
  │            └── Run tests again
  │
  ▼
Run:
cargo fmt
cargo check
cargo test
cargo clippy
  │
  ▼
Security review
  │
  ▼
Generate phase report
  │
  ▼
STOP

والـ report يكون:

Phase: 3 — Detection Engine

Implemented:
- API key detector
- Database URL detector
- JWT detector
- Secret masking

Tests:
- 42 passed
- 0 failed

Security:
- Raw secret values are masked
- Synthetic credentials only

Files changed:
- ...
- ...

Known limitations:
- ...

Status:
READY FOR REVIEW

ولا ينتقل للـ Phase 4 إلا بعد موافقتك.


---

42. Final Product

في النهاية تكون قادر تعمل:

# Local
repoguard scan .

# GitHub through API
repoguard github MohammedMohseng/project --source api

# GitHub through clone
repoguard github MohammedMohseng/project --source clone

# CI
repoguard scan . --fail-on high

# Machine readable
repoguard scan . --format json

والـ Web App لاحقًا يتعامل مع نفس الـ engine:

RepoGuard
                      │
             ┌────────┴────────┐
             │                 │
            CLI               API
             │                 │
             └────────┬────────┘
                      │
                Scanner Core
                      │
       ┌──────────────┼──────────────┐
       │              │              │
     Local        GitHub API       Clone
       │              │              │
       └──────────────┼──────────────┘
                      │
              Detection Engine
                      │
                 Risk Engine
                      │
                  Findings

