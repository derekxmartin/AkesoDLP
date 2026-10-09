<p align="center">
  <img src="docs/assets/logo.png" alt="AkesoDLP" width="200">
</p>

# AkesoDLP

An educational, experimental Data Loss Prevention platform for studying content inspection, policy evaluation, Windows endpoint monitoring, and network enforcement.

AkesoDLP brings together a Windows minifilter and C++ agent, a Python management and detection backend, HTTP/SMTP inspection components, and a React management console. The repository contains substantial implementations of these components, with important integration work still open.

**Use only in authorized, isolated test environments. This is a learning and portfolio project, not production security software.**

> Implementation status below is based on static review of commit [`4776971`](https://github.com/derekxmartin/AkesoDLP/commit/4776971f726b24462e7114ec1017b106d43ad036), reviewed October 9, 2026. It is not a build, test, deployment, or security certification. Commands are documented development entry points; no commands or tests were run for this review.

## Why this project exists

DLP combines several difficult problems: identifying sensitive content, expressing policies without excessive false positives, inspecting different file formats, and deciding where an action can still be prevented. AkesoDLP makes those mechanisms available to read and experiment with, from a kernel write callback to an analyst-facing incident screen.

Useful areas to explore include:

- Regex and keyword matching with secondary validation, such as Luhn and IBAN checks
- Message decomposition into envelope, subject, body, and attachment components
- Compound policies, exceptions, match thresholds, and severity tiers
- Synchronous file-write verdicts versus asynchronous monitoring
- Local persistence, agent/server communication, and management workflows

## Architecture and components

| Component | Stack | Implementation scope |
| --- | --- | --- |
| [Windows minifilter](agent/driver/) | C, Windows Driver Kit | File-operation callbacks and a Filter Manager communication port; can complete an intercepted write with `STATUS_ACCESS_DENIED` after a block verdict. |
| [Endpoint agent](agent/) | C++17, CMake, Windows APIs | Driver communication, content inspection, native policy evaluation, clipboard and browser monitoring, discovery scans, response helpers, SQLite persistence, and gRPC client code. |
| [Management server](server/) | Python 3.12, FastAPI, SQLAlchemy, PostgreSQL | REST API, JWT/TOTP authentication, role permissions, policy and incident administration, agent/discovery RPCs, reporting, and export services. Redis supports policy-update messaging. |
| [Server detection library](server/detection/) | Python, google-re2, pyahocorasick, python-magic | Pluggable analyzers, document and archive inspection, fingerprint matching, and a separate compound-policy evaluator. |
| [Network inspection](network/) | Python, mitmproxy, aiosmtpd | HTTP and SMTP parsing, normalization, detection, and prevention logic for traffic explicitly routed through these components. |
| [Management console](console/) | React 19, TypeScript, Vite 8, Tailwind CSS 4, Recharts | Dashboard, incident review, policy editing, agents, discovery, fingerprint management, reports, risk views, and settings pages. |

The principal endpoint path in code is:

```text
Windows file-write callback
  -> filter communication port
  -> agent detection pipeline
  -> content classification and policy evaluation
  -> allow/block verdict returned to the driver
  -> incident and response work
```

The console uses the server's REST API. Agent communication uses the shared [protobuf definitions](proto/akesodlp.proto). The Python detection engine is a library used by individual entry points; the current REST, gRPC, and network paths do not all configure or invoke the same analyzers and policy engine.

## Implemented capabilities

### Content detection and policy evaluation

The [server detection library](server/detection/) includes:

- Compiled google-re2 regex analyzers
- Aho-Corasick keyword matching with case, whole-word, and proximity options
- Pattern-based data identifiers with secondary checksum or format validation
- File-type analysis and document fingerprinting
- Component-aware policy evaluation: AND within a rule, OR across rules, sender/recipient constraints, whole-message or matched-component exceptions, and severity thresholds
- PDF, DOCX, XLSX, PPTX, email, HTML, and text extraction through [FileInspector](server/detection/file_inspector.py)
- A separate [ArchiveInspector](server/detection/archive_inspector.py) with recursive extraction and depth, byte, file-count, and compression-ratio limits

The [native agent detection code](agent/src/detection/) includes optional Hyperscan regex matching, an Aho-Corasick keyword matcher, validators, magic-byte classification, bounded extraction, and a C++ policy evaluator. Capabilities depend on the libraries available at build time and on how the pipeline is configured. Native extraction is narrower than the Python implementation: PDF/OLE and unsupported archive paths use printable-string fallbacks rather than a full document parser.

The [seed script](server/scripts/seed.py) defines ten identifier examples and six suspended policy templates: PCI-DSS, HIPAA, GDPR, SOX, source-code leakage, and confidential documents. These are starting points for experiments, not evidence of regulatory compliance or comprehensive coverage of those data categories.

### Management and analysis

The server and console provide code for policy and response-rule management, incident triage, agent inventory, discovery jobs, dictionaries, identifiers, fingerprints, reporting, and user-risk views. The repository also includes notification, SIEM/syslog export, dead-letter, metrics, and database-maintenance modules.

Optional AkesoSIEM integration is implemented in the [SIEM emitter](server/services/siem_emitter.py). A configured destination and a working event-producing path are still required; the presence of an emitter does not establish cross-product correlation in a deployed environment.

### Endpoint and network channels

| Channel | Current mechanism | Enforcement boundary |
| --- | --- | --- |
| File writes | [Minifilter pre-write callback](agent/driver/akeso_dlp_filter.c) and user-mode verdict | Can deny an intercepted write. The preview is limited to the first 4 KiB of that write buffer, with skip and fail-open paths. All volume types are currently marked for monitoring, rather than only removable/network volumes. |
| Clipboard | [Clipboard format listener](agent/src/clipboard_monitor.cpp) | Observes Unicode text after a clipboard update and can subsequently clear it. This is not a pre-set clipboard hook; Windows service-session restrictions also matter. |
| Browser activity | [ETW browser file-access monitoring](agent/src/browser_upload_monitor.cpp) | Uses browser file opens/reads as an upload heuristic. It does not hook HTTP send calls or cancel uploads. A logged “block” action in this path is not proof that a transfer was stopped. |
| HTTP uploads | [mitmproxy entry point](network/mitmproxy_entry.py) and prevention logic | Contains an HTTP 403 response path for routed requests. The supplied entry point has a constructor mismatch that must be fixed before treating it as runnable. |
| SMTP email | [SMTP entry point](network/smtp_entry.py) and prevention logic | Contains reject and redirect paths for routed mail. The supplied entry point has the same constructor issue; its modify verdict does not apply message modifications before forwarding. |
| Data at rest | [DiscoverScanner](agent/src/discover_scanner.cpp) and discovery APIs | Includes incremental scanning, throttling, and scan/result plumbing. Detection depends on active pipeline policies and complete result reporting. |

HTTP/SMTP attachment handling currently decodes bytes to text in its parsing paths; it does not automatically use the rich PDF/Office extraction available in `FileInspector`.

## Current integration status

These are source-level findings at the reviewed revision. They distinguish implemented building blocks from complete operational workflows.

### Endpoint lifecycle, policies, and incident delivery

- The `--console` branch in [main.cpp](agent/src/main.cpp) constructs and registers the agent components. The default Windows service path calls `RunAsService()` without constructing the service instance used by `ServiceMain`; the SCM path needs lifecycle wiring.
- Server policy retrieval and streaming endpoints are implemented. The agent does not connect the received-policy callback or hydrate cached policies into the detection pipeline. The explicit policy-loading path in `main.cpp` is currently `--test-policy`.
- The heartbeat update branch calls `PullPolicies` while holding a mutex that `PullPolicies` also acquires. This is a static deadlock risk on that branch.
- [PolicyCache](agent/src/policy_cache.cpp) and [IncidentQueue](agent/src/incident_queue.cpp) use SQLite. The incident queue has persistence, deduplication, and bounded eviction; no operational queue-draining incident uploader is wired into the reviewed agent startup path.

Consequently, a connected agent, a cache file, or an “Enforcing” state is insufficient evidence that server-authored policies are active or incidents are reaching the server. The pipeline allows writes when its policy set is empty.

### Detection entry points and two-tier detection

- [`POST /api/detect` and `/api/detect/file`](server/api/detection.py) configure **eight fixed identifiers**: credit card, SSN, phone, email, IBAN, IPv4, date of birth, and ABA routing number. The factory's comment says ten, but its actual list contains eight. These routes return matches rather than evaluating the configured policy collection.
- The file endpoint invokes `FileInspector`; the separate archive inspector is not automatically included in that route.
- The agent's two-tier request sends its available write preview. The current [gRPC detection handler](server/grpc_server.py) configures a smaller identifier set and returns an empty policy-results list. It does not complete full-document, configured-policy, or fingerprint evaluation for the agent.

### Network startup and enforcement

The [HTTP](network/mitmproxy_entry.py) and [SMTP](network/smtp_entry.py) startup factories instantiate `DataIdentifierConfig` and `DataIdentifierAnalyzer` without the arguments required by their [definitions](server/detection/analyzers/data_identifier_analyzer.py). This is a direct source-level startup incompatibility, not a runtime failure reproduced for this README. Prevention classes and tests exist, but they should not be presented as a working Compose deployment until the entry points and delivery paths are repaired and exercised.

### Availability and failure behavior

The minifilter and pipeline contain intentional allow-on-error or bypass paths, including missing agent communication, empty policies/content, and selected system or paging operations. This favors availability and means inspection is not exhaustive. The driver has a full-scan verdict placeholder, but no completed full-file rescan path. Do not interpret a successful test write, a healthy HTTP endpoint, or one blocked sample as a guarantee of leak prevention.

## Development setup

### Prerequisites

- Python 3.12 and the packages in [requirements.txt](requirements.txt); native `libmagic` support is needed for file-type detection. [server/Dockerfile](server/Dockerfile) records the Linux system packages used by the container.
- Node.js 22 for the console, matching [console/Dockerfile](console/Dockerfile), and npm
- PostgreSQL 16 and Redis 7, matching the development Compose file
- Docker with Compose v2 if using the supplied infrastructure containers
- For the Windows agent: Visual Studio 2022 C++ tools, vcpkg, and CMake. The preset file uses schema version 6; use CMake 3.25 or newer for those presets. The Ninja presets also require Ninja. Driver builds additionally require the WDK.

### Local server and console

Use a disposable development environment. Review [.env.example](.env.example), [server/config.py](server/config.py), and [docker-compose.yml](docker-compose.yml) before starting services.

From the repository root, prepare a Python virtual environment, activate it using your platform's normal command, and install dependencies:

```bash
python -m venv .venv
# Activate .venv before continuing.
python -m pip install -r requirements.txt
```

Create a local `.env` from `.env.example` and replace its placeholders. Set the database URL for your development PostgreSQL instance and the Redis URL for your Redis instance. Set `DLP_DEBUG=true` only for isolated local development, and keep `DLP_SIEM_ENABLED=false` unless you intentionally configure an authorized test receiver.

If using the provided infrastructure, this starts only PostgreSQL and Redis:

```bash
docker compose up -d postgres redis
```

Those Compose services publish ports on host interfaces. Restrict access to your test environment. The `.env` database settings must match the chosen database; copying the example without editing it is insufficient.

The server's documented application entry point is:

```bash
python -m uvicorn server.main:app --host 127.0.0.1 --port 8000 --reload
```

Startup creates missing ORM tables and attempts to start gRPC. It does **not** run the seed script. For a fresh disposable database, after tables have been created, the explicit seed entry point is:

```bash
python -m server.scripts.seed
```

Review the script first: it creates a known demonstration admin account, roles, identifiers, and suspended templates. It is not an idempotent migration; do not repeatedly run it against an existing database. Replace demonstration credentials before using any non-disposable environment.

In a separate terminal, start the console locally:

```bash
cd console
npm install
npm run dev
```

The configured local URLs are `http://localhost:3000` for the console, `http://localhost:8000/docs` for the API schema, and `http://localhost:8000/api/health` for basic API liveness. The health route returns a static response and does not verify gRPC, policy loading, incident delivery, or enforcement.

### Compose caveats

The full Compose file also defines the server, console, MailHog, HTTP proxy, and SMTP relay. It is not a verified one-command deployment:

- Review and remove the bundled SIEM credential-like value, and disable the default SIEM destination unless intentionally using it.
- The console's [Vite proxy](console/vite.config.ts) targets `localhost:8000`. That works for a local console and local API, but inside the console container it points back to that container; containerized API routing needs adjustment.
- The network startup incompatibilities described above apply to the proxy and relay services.
- `make install` prints admin login information but does not invoke the seed script. Neither its success message nor container liveness establishes a working demo.

The file named [docker-compose.prod.yml](docker-compose.prod.yml) and [deployment script](scripts/deploy.sh) are deployment scaffolding. They do not establish production readiness; the current container definitions still run development servers.

### Windows agent build

From an x64 Visual Studio developer environment with vcpkg available through `VCPKG_ROOT`:

```powershell
cd agent
cmake --preset debug
cmake --build --preset debug
```

The `debug` preset enables tests and disables the driver. Other presets, including `release-driver`, are defined in [CMakePresets.json](agent/CMakePresets.json). Inspect CMake's dependency summary: optional discovery and feature macros mean a configured build is not proof that every detection or communication feature is enabled. GoogleTest must be found for native tests to be registered.

Use [agent/config/config.yaml.example](agent/config/config.yaml.example) as the configuration reference. The existing [development guide](docs/DEVELOPMENT.md) and [manual endpoint test plan](tests/agent/test_e2e.md) contain lab procedures. Driver installation, signing, service lifecycle, and policy loading require separate validation. Use a disposable Windows VM and synthetic files; this README does not recommend loading the driver on a daily-use machine.

## Tests and verification

The repository includes Python detection/API/network tests, live-service integration tests, native GoogleTest targets, Playwright console tests, benchmark scripts, and a manual Windows endpoint plan. Their presence does not imply that they pass at the reviewed revision.

The current [parity suite](tests/integration/parity_test.py) exercises the Python REST API; it does not perform C++/Python differential validation. The Actions history reviewed contained dependency-maintenance jobs, so it provides no application-CI success claim for this snapshot.

Source-grounded entry points include:

```bash
# Focused Python detection suite
python -m pytest tests/detection/ -v

# Python API suite
python -m pytest tests/api/ -v

# Live-service integration suite, after configuring a working test server
python -m pytest tests/integration/ -v

# Server lint and formatting checks
python -m ruff check server/
python -m ruff format --check server/
```

[Integration fixtures](tests/integration/conftest.py) use `DLP_TEST_URL`, `DLP_TEST_USER`, and `DLP_TEST_PASS`. These tests authenticate and exercise an actual server; use a disposable test database.

For the console, `npm run build` and `npm run lint` are defined in [package.json](console/package.json). Playwright scripts and a [configuration](console/playwright.config.ts) exist, but `@playwright/test` is not declared in that manifest. Resolve the test dependency and browser installation first; the browser tests also assume a seeded backend. Do not treat `npm install` alone as complete E2E setup.

For native tests, after a successful test-enabled build:

```powershell
ctest --test-dir agent/build/debug --output-on-failure
```

Run that command from the repository root. At this revision, `make test` points pytest at `server/`, while the Python suites are under `tests/`; use explicit suite paths. No passing test count or coverage claim is made here.

## Security and deployment boundaries

- Keep testing authorized, isolated, and limited to synthetic sensitive data. Inspection results, incident snippets, recovery files, quarantine files, and exports can themselves contain sensitive information.
- Replace sample credentials and secrets, restrict listening ports, review CORS and authentication configuration, and define retention and access controls before any broader deployment. If a checked-in credential has ever been valid, revoke or rotate it at its provider.
- gRPC supports a certificate-based mTLS branch, but ordinary FastAPI startup currently passes only the port and selects the insecure branch. Merely setting TLS fields in configuration does not wire that startup path to mTLS. The [agent client](agent/src/grpc_client.cpp) also falls back to insecure credentials if its CA certificate cannot be read.
- HTTPS inspection requires deliberate proxy routing and certificate trust in the test environment. The proxy does not automatically observe all endpoint traffic. Do not install a test interception CA on unrelated or production systems.
- Fail-open behavior, partial content previews, fallback extraction, asynchronous clipboard handling, and heuristic browser monitoring leave coverage gaps. Treat false positives, false negatives, and uninspected content as expected evaluation concerns.
- Quarantine, recovery, and response code can move or modify files. Use backups and disposable fixtures when evaluating them.

## Repository layout

```text
AkesoDLP/
├── agent/                  C++ agent, WDK driver, native tests, installer sources
├── console/                React application and browser tests
├── server/
│   ├── api/                FastAPI route handlers
│   ├── detection/          Analyzers, extraction, and policy evaluation
│   ├── models/             SQLAlchemy models
│   ├── schemas/            API schemas
│   ├── services/           Management, reporting, and export logic
│   ├── proto/              Generated Python protobuf/gRPC code
│   ├── scripts/            Seed and administrative utilities
│   └── tasks/              Maintenance tasks
├── network/                HTTP and SMTP inspection components
├── proto/                  Shared protocol definition
├── migrations/             Alembic migrations
├── tests/                  Python suites, integration tests, benchmarks, fixtures
├── docs/                   Architecture, API, development, and demo documents
├── scripts/                Deployment, certificate, and VM utilities
├── tools/                  Scenario generation
├── Makefile                Development command shortcuts
├── docker-compose.yml      Development service definitions
└── docker-compose.prod.yml Deployment scaffolding
```

## Next integration milestones

1. Complete the Windows service lifecycle and policy flow: initial retrieval, cache hydration, live updates, and heartbeat locking.
2. Connect durable incident delivery with retry, acknowledgement, and failure tests; verify discovery result completeness.
3. Repair network entry points, apply SMTP modifications, and validate routed HTTP/SMTP traffic end to end.
4. Unify configured-policy evaluation across REST, gRPC, and network paths; complete document/fingerprint escalation where intended.
5. Wire TLS into normal startup, make console/container routing consistent, and validate a repeatable fresh-environment setup.
6. Reconcile test targets and dependencies, then record reproducible build, test, and adversarial validation results before making stronger enforcement claims.

[REQUIREMENTS.md](REQUIREMENTS.md) preserves the project's broader design and task history. Its phase-completion labels and older documentation describe implementation goals and milestones; they should not substitute for verifying the operational paths above.

## License

MIT. See [LICENSE](LICENSE). Copyright © 2026 Derek Martin.

